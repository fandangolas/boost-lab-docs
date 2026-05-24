# Design Critique: BoostLab

**Audience lean:** tuner / enthusiast
**Stage:** late refinement (polished dark theme, working presets, live Monte Carlo viz)
**Scope:** the single-page web UI at `static/index.html`, in context with the Rust simulation in `src/physics.rs` and `src/simulation.rs`.

---

## Overall Impression

It looks like a real engineer's tool — restrained dark theme, monospace everywhere, gold-on-blue dyno palette that nods to traditional dyno-sheet conventions. A tuner opening this will trust it on sight. The biggest opportunity is that the chart is half a dyno: it shows power but not torque, and the Monte Carlo varies the two parameters tuners *can't* change (displacement, compression) rather than the ones they tune at the dyno (AFR, ignition, boost target, fuel quality). Fixing those two gaps would move this from "cool toy" to "I'd actually open this before a dyno session."

---

## Usability

| Finding | Severity | Recommendation |
|---|---|---|
| **Torque curve is collected but never drawn.** `torqBucket` is populated and used for the peak-torque annotation, but the chart only renders HP. Every real dyno graph overlays HP + torque. | 🔴 Critical | Add a second Y-axis (Nm/lb-ft) with the mean torque line in blue, plus its own p10–p90 band. |
| **Monte Carlo varies the wrong things.** `simulation.rs::sample_variation` jitters only displacement (±2%) and CR (±2%) — manufacturing tolerance, not tuning variance. | 🔴 Critical | Vary the inputs a tuner actually moves: AFR/λ (0.78–0.88), ignition timing margin, boost target tolerance, fuel quality (91/93/100 oct), IAT. The p10–p90 band then means "expected outcome range across a dyno session" — decision-useful. |
| **No unit toggle.** Boost is bar, torque is Nm, no lb-ft / PSI / kW options. US and UK tuners live in PSI and lb-ft. | 🟡 Moderate | Single metric/imperial toggle in the header. Keep SI internally, convert on display. |
| **No run comparison / overlay.** "What if I raise CR to 12 and switch to E85" requires running, memorizing numbers, then running again. | 🟡 Moderate | Even one ghost slot — last run plotted as a faded line under the current one — covers 80% of the value. |
| **No export.** Tuners share results in forums and Discords. There's no way to save the chart. | 🟡 Moderate | "Save dyno sheet" button → 1200×800 PNG with engine spec header baked in. |
| **Presets need two clicks.** Click preset → set form → press Run. Exploring 6 presets is 12 clicks. | 🟡 Moderate | Click preset = instant Monte Carlo run. |
| **Empty chart at load.** First-time impression is a blank canvas with a form. | 🟡 Moderate | Auto-run the default config (2.0L NA 4-cyl) once on load — the empty state is the worst version of the product. |
| **Sim count (10–2000) has no guidance.** Tuners don't know what 300 buys them vs. 1000. | 🟢 Minor | Either three pills (Fast / Balanced / Precise) or a one-liner: "300 = quick read, 1000+ = tighter confidence bands". |
| **No cancel during a run.** Run button just disables. Backend already supports `CancellationToken`. | 🟢 Minor | Toggle the button to "■ Cancel" during simulation — the plumbing is already there. |
| **No tooltips on technical fields.** Even tuners benefit from "CR 11+ assumes 98 RON; raise boost target with caution under 9:1." | 🟢 Minor | Tooltip on Compression Ratio and the Aspiration options (mention the assumed AFR=0.88, fuel=98 RON). |
| **No live preview as inputs change.** Bumping CR from 10.5 to 11.0 should give immediate visual feedback. | 🟢 Minor | Render a deterministic single-curve preview on input change; the full Monte Carlo only runs on Run. |
| **Boost field uses `display:none` toggle without ARIA management.** | 🟢 Minor | `aria-hidden` + `inert` when not applicable, so screen-reader and keyboard tab order match the visible UI. |

---

## Visual Hierarchy

**What draws the eye first** — the gold "Peak Power" stat in the top stats row. For a tuner that's the correct hero metric, so the eye lands well.

**Reading flow** — header → stats row → status/progress bar → chart → (sidebar last). The chart is the actual product but it's the *last* thing the eye reaches because the stats row sits above it. Once a run finishes, the stats are also redundant with the on-chart annotations (`⚡ Peak Power XXX HP @ YYYY RPM`). Two places saying the same thing dilutes both.

**Emphasis issues**

- `Simulations: 0` is given the same visual weight (gold-less but same font size) as Peak Power and Peak Torque. It's a *process* metric — should be smaller, or live in the status row.
- The 2px progress bar is invisible-thin and stays at 100% after a run, wasting the row. Either hide it post-run or merge it into the status text ("Done — 300 simulations · 1.4s").
- Annotation density is approaching its limit. With 200+ faint curves, p10–p90 band, mean line, peak-power vline, peak-torque vline, spool box, full-boost vline, "p10–p90 spread" label, and "mean" label — all on one chart — it reads fine for a tuner but a newcomer will be lost. Consider a "Detail / Clean" toggle that strips annotations on demand.

**Suggested move:** push the four primary stats into a small card overlaid in the top-right of the chart area (translucent over `--bg`). Chart becomes the hero, stats stay glanceable, and the row above can collapse to status + progress only.

---

## Consistency

| Element | Issue | Recommendation |
|---|---|---|
| **"Engine Spec" section** also contains "Simulations" — a runner knob, not a physical spec. | Mental model leak. | Split into two sections: `Engine` and `Run`. |
| **Field labels:** "Displacement (L)" spelled out, "Comp. Ratio" abbreviated. | Inconsistent abbreviation rule. | Pick one. "Disp. (L)" + "CR" reads tuner-native; or spell both out. |
| **Preset cards vs. input fields** share `--surface` + `--border`. Hard to scan at a glance. | Two component types look identical. | Slightly warmer surface for presets, or a faint left accent stripe, or a tiny maker silhouette. |
| **Emoji icons** (`⚡ 🔩 🌀 ⚙ ▶`) mix functional and decorative. | Toy-ish in a tuner tool. | Swap for thin SVG glyphs (`feather-icons` zap / bolt / fan / play) — same shape language, more "instrument cluster" tone. |
| **Boost field** appears only when aspiration != NA, but the *target* boost is a single number with no curve/onset RPM. | Asymmetric with how the physics actually models spool (Watson & Janota correlation in `calculate_boost_pressure`). | Expose at least "Target boost" + "Spool start RPM" — the model already varies onset by aspiration type; let the user override. |

---

## Accessibility

Spot-checked against the actual token values in `:root`.

| Token | Value | Used for | Contrast vs `--bg #09090d` | WCAG AA |
|---|---|---|---|---|
| `--text` | `#d4d4e8` | body text | 13.61 : 1 | ✅ AAA |
| `--gold` | `#f5c842` | peak HP, mean line | 12.52 : 1 | ✅ AAA |
| `--accent` | `#4f8ef7` | focus, run button bg | 6.19 : 1 (text on bg) | ✅ AA normal |
| `--muted` | `#4a4a6a` | **labels, section headers, axis ticks, status text, placeholder text** | **2.35 : 1** | ❌ **Fails AA normal (4.5:1) and even AA Large (3:1)** |
| Run-btn white-on-blue | `#fff on #4f8ef7` | primary CTA | 3.21 : 1 | ⚠️ Passes AA Large only (text is bold 0.8rem, borderline) |

- **`--muted` is the biggest issue** — it carries every label and every axis tick, and at 2.35:1 it actually fails even the lenient AA-Large threshold (3:1). Bumping to `#8a8aaa` lands at ~5.96:1 (clears AA normal) and barely shifts the aesthetic.
- **Focus state** relies solely on a `border-color` swap on inputs. On dark backgrounds, keyboard-only users (rare for tuners but real) can miss it. Add `outline: 2px solid var(--accent); outline-offset: 1px` on `:focus-visible`.
- **Touch targets** — Run button is fine. Preset buttons are ~28px tall (0.4rem padding × 0.68rem font). Below the 44×44 mobile minimum. Tuners are often on tablets at the dyno or in the garage with greasy fingers — bump to ≥40px.
- **Canvas has no text alternative** — screen readers get nothing. Add `aria-label="Dyno chart showing power curve from N runs"` and a hidden `<table>` (or a downloadable CSV) with the mean curve data points.
- **No responsive breakpoint.** `grid-template-columns: 280px 1fr` plus `overflow: hidden` on body means a phone viewport collapses to "sidebar fills the screen, chart unreachable." Tuners *do* check things from a phone. At ≤768px, stack vertically: form on top (collapsed), chart below.
- **Hidden boost field** — when toggled out via `display:none`, the input is removed from the tab order, which is correct. But if you later animate it, prefer `inert` so the SR state matches the visual state.

---

## What Works Well

- **Aesthetic is on-brand for the audience.** Restrained dark theme + monospace reads "real instrument", not "consumer app." Tuners distrust glossy.
- **Monte Carlo concept is genuinely novel** for a hobby dyno tool. Showing uncertainty bands is honest — tuners respect that more than a single confident line.
- **The presets are well-chosen** and the specs are accurate (K20A at 11.5:1 / 8600, 2JZ-GTE at 8.5:1 / 7000 twin-turbo) — a tuner notices these details instantly and decides whether to trust the rest.
- **Spool zone + "Full Boost" vline** are exactly the kind of domain detail that separates an enthusiast project from a generic "calculator."
- **SSE streaming with the rolling progress bar + counter** makes the simulation feel alive. It mimics watching a dyno screen during a pull — emotionally correct.
- **Gold mean line over faded blue runs** is the right idiom — racing telemetry and Bayesian-inference visualizations both use this and it reads well.
- **Sensible defaults** (2.0L, 4 cyl, 7000 RPM, 10.0 CR) — the most common "first run" config, won't intimidate.
- **The header is appropriately invisible** — it gets out of the chart's way.

---

## Priority Recommendations

1. **Plot the torque curve.** The data is already collected (`torqBucket`). Add a second Y-axis with mean torque in blue and its own p10–p90 band. This single change moves the most needles: tuner trust, hierarchy of the chart, redundancy of the stats row, and accuracy as a "dyno" rather than a "power graph."
2. **Vary the parameters tuners actually tune.** Replace or extend `sample_variation()` to jitter AFR/λ, ignition timing margin, boost target tolerance, IAT, and fuel quality. The p10–p90 band then represents real session-to-session variance, which is what tuners want to see before they commit dyno time.
3. **Fix `--muted` contrast.** Bump `#4a4a6a` → `#8a8aaa` (or similar). Affects nearly every label in the UI and clears WCAG AA. Doesn't break the quiet aesthetic.
4. **Add a unit toggle and a comparison overlay.** Metric/imperial in the header; "compare to previous run" as a ghost line. These are the two single-feature requests tuners will send in a Discord channel within 24 hours of trying the tool.
5. **Make presets one-click runs and add a PNG export.** Presets should auto-Run. After a run, a "Save dyno sheet" button produces a 1200×800 PNG with the engine spec baked into a header — that's the artifact that gets shared and pulls more users in.
6. **Reorganize the section labels and demote the sim counter.** Split `Engine` (physical specs) from `Run` (sim count, future: cancel, presets, compare). Move `Simulations: 0` into the status row at half size; it's not a result.
7. **Add a "Clean / Detail" annotation toggle.** Current annotation density (spool box + full-boost line + peak-power vline + peak-torque vline + spread label + mean label + 200 faint curves) is at the readable limit. A one-tap toggle keeps tuners happy ("show me everything") and unlocks the tool for the "curious newcomer" sub-audience if you ever want it.

---

## Out-of-Scope but Worth Noting

- The physics module bakes in lambda = 0.88, fuel = 98 RON, mech efficiency = 0.88. These are reasonable defaults but they're invisible — surface them as locked fields with tooltips so a tuner can see what the model assumed and trust (or distrust) the result.
- There's no way for the model to flag "this config is unrealistic" (CR 14:1 with 2.5 bar boost on pump gas would knock itself apart). A small ⚠️ on the peak-power stat with a tooltip ("Detonation risk — boost target exceeds fuel headroom") would be genuinely educational and is in scope of what your physics already knows.
