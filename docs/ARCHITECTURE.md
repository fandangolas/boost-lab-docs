# Architecture Decisions — BoostLab

Physics and statistical modelling decisions are tracked separately in [MODELLING.md](MODELLING.md).

---

## 1. SSE over WebSockets

**Decision:** The simulation streams events to the browser via Server-Sent Events (`/events`), not WebSockets.

**Why:** Simulation data flows in one direction only — server to client. SSE is a better fit than WebSockets for unidirectional push: it runs over plain HTTP/1.1, the browser's `EventSource` API reconnects automatically, and the implementation on both sides is significantly simpler. WebSockets would have added handshake complexity and a full-duplex abstraction that the use case doesn't need.

**How:** `tokio::sync::broadcast` channel fanouts events from the simulation task to all connected SSE handlers. Each `GET /events` connection subscribes to the same channel independently, so multiple browser tabs work without any extra plumbing.

---

## 2. Cancellation via `CancellationToken`

**Decision:** Each simulation run is controlled by a `tokio_util::sync::CancellationToken` stored behind an `Arc<Mutex<…>>`.

**Why:** A user pressing Run mid-simulation should immediately stop the previous run. The pattern — lock, cancel the old token, create and store a new one, drop the lock, spawn the task with the new token — ensures the old task self-terminates at the next yield point without any `abort()` or unsafe teardown.

**How:** The simulation loop checks `cancel.is_cancelled()` every iteration and returns early if set. `tokio::task::yield_now().await` every 5 iterations gives the runtime a chance to deliver the cancellation signal.

---

## 3. Frontend: vanilla JS + Chart.js, no framework

**Decision:** The frontend is a single HTML file with no build step, no npm, no bundler.

**Why:** The UI has one screen and one job. Introducing a framework (React, Vue, etc.) or a build pipeline would add complexity with no functional return. Chart.js handles all rendering; the application logic (SSE handling, bucketing, percentile computation) is straightforward enough to write in plain JS without abstractions.

**Performance decisions inside the frontend:**
- `animation: false` on Chart.js — redraws during streaming would lag behind the SSE feed
- `chart.update('none')` — skips the animation phase on each batch flush
- 40 ms flush timer — batches incoming SSE events so the DOM isn't updated on every message
- Faint simulation traces capped at 500 — beyond that, additional lines add visual noise without statistical value
