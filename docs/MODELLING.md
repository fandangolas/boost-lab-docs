# Modelling Decisions — BoostLab

This document tracks decisions made about the physics and statistical model. Architecture and system design decisions live in [ARCHITECTURE.md](ARCHITECTURE.md).

The model is intentionally versioned here. Each section describes what was chosen, why, and what the known limitations are — so future improvements have a clear baseline to build from.

---

## Model v1

### Why Monte Carlo — and how it is applied here

**Decision:** Engine output is estimated by running N independent simulations, each with a different sample of uncertain input parameters, rather than solving a single deterministic equation.

**Why:** A single-point calculation produces a single-point answer. That answer is only as trustworthy as the precision of its inputs — and in engine tuning, the inputs are never perfectly known. Lambda drifts across pulls, ambient temperature changes between morning and afternoon, fuel quality varies by station, timing effectiveness depends on knock margins that shift with heat soak. A deterministic model silently absorbs all of that uncertainty into its output and presents it as a fact.

Monte Carlo makes the uncertainty explicit and honest. By sampling each uncertain input from a realistic distribution and running the full physics model for each sample, the method produces a *distribution of outcomes* rather than a point estimate. The width of that distribution is not an error — it is the answer. A tight p10–p90 band means the engine is insensitive to session conditions; a wide band means the opposite, and a tuner should investigate why.

Alternatives considered:
- *Analytical uncertainty propagation* — linearises the model around the nominal point and propagates variances through partial derivatives. Fast, but only valid when the model is approximately linear and inputs are small perturbations. The BMEP formula is nonlinear in CR and lambda, so this would understate the true spread.
- *Latin Hypercube Sampling* — a more statistically efficient sampling strategy that ensures better coverage of the input space with fewer samples. Worth considering for a future version where each simulation is more expensive.
- *Single deterministic run* — what every other dyno calculator does. Tells you nothing about confidence.

**How it works in practice:**

```
for i in 0..N:
    spec_i = sample_variation(base_spec)   // draw from input distributions
    curve_i = simulate_engine(spec_i)      // full physics sweep, 1000 RPM → redline
    accumulate(curve_i)                    // bucket HP and torque by RPM

stats = bucket_to_curves(buckets)          // mean, p10, p90 per RPM point
```

Each simulation is fully independent — no shared state between runs, no warm-starting from the previous result. This is intentional: it keeps the samples unbiased and makes the loop trivially parallelisable in a future version.

**Sample count trade-offs:**
| N | Band stability | Typical runtime |
|---|---|---|
| 50–100 | Noisy — p10/p90 jump between runs | < 0.3 s |
| 300 | Good enough for visual inspection | ~0.8 s |
| 1000 | Stable bands, tight percentile estimates | ~2.5 s |
| 2000 | Diminishing returns beyond this | ~5 s |

The default of 300 is chosen as the crossover point where the bands are stable enough to be meaningful but the result appears before the user loses interest.

---

### Physics: Heywood BMEP correlation

**Decision:** Engine torque is derived from the Heywood brake mean effective pressure (BMEP) formula.

**Why:** The correlation is physically grounded, generalisable to arbitrary displacement/cylinder configurations without per-engine lookup data, and was calibrated against real dyno data in the previous version (±3% accuracy). Alternatives considered:
- *Lookup tables* — accurate for specific engines, useless for novel configurations.
- *Empirical polynomial fits* — overfit to the training engines and degrade badly outside the calibrated range.
- *Full 1D gas dynamics (e.g. GT-Power style)* — correct, but orders of magnitude more complex to implement and too slow for Monte Carlo use.

**Formula:**
```
bmep = VE × p_intake × CR^0.30 × k_lambda × k_fuel × 7.5
```

**Baked-in assumptions (v1):**
| Constant | Value | What it represents |
|---|---|---|
| `CR^0.30` | — | Heywood exponent for compression ratio effect |
| `k_fuel` | 1.0 | 98 RON pump fuel assumed |
| `mech_eff` | 0.88 | Mechanical friction losses |
| `p_intake` | 1.01325 bar | Standard atmospheric pressure (NA); boosted via spool curve |

These are invisible to the user in v1. A future version should surface them as locked fields with tooltips so the user understands what the model assumed.

---

### Volumetric efficiency curve

**Decision:** VE is modelled as a smoothstep curve — rising from idle to peak torque RPM, then falling off to redline.

**Why:** Real engines follow this shape due to intake/exhaust tuning. A flat VE would produce a torque curve that rises monotonically with RPM, which is physically wrong for a naturally aspirated engine.

**Values:**
- `76%` at idle (1000 RPM)
- `100%` at peak torque RPM
- `90%` at redline

**Peak torque RPM** is derived from bore/stroke ratio:
```
peak_torque_rpm = redline × (0.75 − (S/B − 0.95) × 0.20).clamp(0.60, 0.76)
```
Undersquare engines (long stroke, S/B > 1) peak earlier; oversquare (short stroke) peak later.

**Known limitation:** The VE curve shape is fixed. Real engines have highly individual VE curves shaped by cam timing, port geometry, and intake runner length. A future version could allow the user to specify the VE peak RPM and width directly.

---

### Forced induction spool modelling

**Decision:** Boost pressure builds via an S-curve (turbos) or linear ramp (supercharger) rather than a step function at a fixed RPM.

**Why:** A step function produces a discontinuous torque kink that looks wrong and misleads the user about the driveability of the engine. The S-curve approximates the real behaviour of a turbocharger as it comes onto boost.

**Onset points (v1):**
| Type | Spool start | Full boost |
|---|---|---|
| Single Turbo | 25% of redline | 60% of redline |
| Twin Turbo | 20% of redline | 50% of redline |
| Supercharged | 0% (linear from idle) | 40% of redline |

**Boost multiplier:** `(1 + boost_bar) × 0.82` — the 0.82 factor accounts for intercooler efficiency and charge temperature losses.

**Known limitation:** Spool onset RPM is fixed per aspiration type and not user-configurable. Real turbo setups vary enormously based on turbine sizing. A future version should expose spool start RPM and rate as explicit inputs.

---

### Monte Carlo: what to vary

**Decision:** Each simulation samples AFR/λ, intake air temperature, ignition timing margin, and boost target tolerance from probability distributions. Displacement and compression ratio are held fixed.

**Why:** The original v0 model varied displacement and CR (±2%) — manufacturing tolerances that are invisible to a tuner and constant between dyno pulls. The p10–p90 band was therefore modelling production variance, not session variance. Replacing those with tuner-relevant parameters makes the confidence band answer the question a tuner actually has: *"what range of power should I expect across pulls on the day?"*

**Distributions (v1):**
| Parameter | Distribution | σ | Clamp |
|---|---|---|---|
| Lambda (λ) | Normal(μ = spec) | 0.015 | [0.76, 0.92] |
| IAT (°C) | Normal(μ = spec) | 5.0 | [10, 55] |
| Timing margin | Normal(μ = 0) | 0.02 | [−0.05, +0.05] |
| Boost scatter (FI) | Normal(μ = 0 bar) | 0.04 | [−0.10, +0.10] |

**Known limitation:** Lambda, IAT, and timing margin are varied independently. In reality they are correlated — a hot ambient day raises IAT, which forces a richer lambda target, which reduces power more than either factor alone. A future version could model these as a joint distribution.

---

### Monte Carlo: statistical representation

**Decision:** Results are presented as a p10–p90 confidence band around a mean curve, not as ±1σ or min/max.

**Why:**
- *Min/max* is dominated by outlier samples and overstates the true spread.
- *±1σ* assumes a symmetric normal distribution, which the output curves are not (they are slightly right-skewed due to the lambda and IAT interactions).
- *p10–p90* is a non-parametric 80% interval that is robust to skew and familiar to engineers from reliability and performance testing contexts.

**IAT air-density correction:**
```
bmep_corrected = bmep × (298 / (IAT_celsius + 273))
```
Standard day is defined as 25 °C (298 K). Hotter air is less dense, reducing volumetric efficiency and therefore power.

---

## Known model limitations (v1 → future)

| Limitation | Impact | Possible fix |
|---|---|---|
| Fixed fuel type (98 RON) | Underestimates knock risk on 91/95 RON; overestimates power on E85 | Add fuel quality selector |
| Fixed lambda (0.88 default) | May be too rich for some NA setups, too lean for some boosted | Already user-varied by Monte Carlo; could expose as an explicit spec field |
| Fixed mech efficiency (0.88) | Doesn't account for high-drag builds (dry sump, all-wheel-drive losses) | Expose as a locked field with tooltip |
| No detonation / knock model | CR 14:1 on 91 RON with 2.5 bar boost is silently accepted | Add ⚠️ warning when boost + CR exceeds fuel headroom |
| VE curve shape is fixed | Real engines have unique VE peaks shaped by cam and port geometry | Let the user specify VE peak RPM and bandwidth |
| Spool onset RPM is fixed per type | A small single turbo on a big engine spools much later than the default | Expose spool RPM as an input |
| Independent parameter sampling | λ, IAT, timing are correlated in reality | Model as a joint distribution |
