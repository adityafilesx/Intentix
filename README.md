## System Architecture

This section describes Intentix's architecture with actionable details about components, data flow, concurrency, failure modes, and deployment patterns. The goal is to make responsibilities, integration points and trade-offs explicit so contributors and operators can reason about reliability, privacy, and scaling.

### High-level overview

Intentix is intentionally lightweight and privacy-first. It runs a local Flask web UI (control surface) together with a background automation/ML subsystem that:

- Reads camera frames and runs face/gaze/blink detection (MediaPipe).
- Accepts text and commands (UI, voice).
- Dispatches non-blocking automation tasks (typing to focused window, WhatsApp/Twilio flows) via worker threads or background workers.
- Optionally integrates a message queue (Redis) to support reliability and horizontal scaling.

Design goals:
- Local-first processing for privacy (MediaPipe, pyautogui run locally).
- Minimal blocking of the web server — background workers handle side effects.
- Configurable for single-machine use and optionally scalable to a distributed worker model.

### Architecture diagram

> Paste into a renderer that supports Mermaid (GitHub may render mermaid in some contexts).

```mermaid
flowchart LR
  subgraph Client
    FE[Web UI (Flask templates + JS)]
    BrowserCam[Optional Browser Camera UI]
  end

  subgraph Server
    API[Flask App (main.py)]
    Scheduler[(Job Scheduler)]
  end

  subgraph Workers
    LocalWorker[Local worker threads/processes]
    QueueWorker[Queued worker (Redis/RQ) - optional]
  end

  subgraph Local
    Cam[Camera + MediaPipe]
    Input[pyautogui (keyboard/mouse)]
    Voice[Speech capture / recognizer]
  end

  subgraph External
    Twilio[Twilio (optional)]
    Browser[Web Browser (WhatsApp)]
    S3[S3 / Object Storage (optional)]
  end

  FE -->|HTTP / WebSocket| API
  API -->|enqueue / spawn| Scheduler
  Scheduler --> LocalWorker
  Scheduler --> QueueWorker
  LocalWorker --> Input
  QueueWorker --> Input
  Cam -->|frames| API
  Voice -->|commands| API
  API --> Twilio
  API --> Browser
  API --> S3
```

### Component responsibilities

- Web UI (templates + static)
  - Editable "Text Tag" textarea and controls (start/stop gaze, TYPE EXTERNAL, emergency).
  - Sends `POST /perform_action` and opens WebSocket or SSE for live status updates.
  - Minimal business logic: compose commands and display status.

- Flask App (main.py)
  - Serves UI, static assets and small REST API endpoints (actions, health, config).
  - Accepts commands and enqueues background tasks (local thread or persist to queue).
  - Lightweight validation and auth for local deployment.

- Scheduler & Worker(s)
  - Short-lived automation: spawn OS-focused automation in background threads with clear timeouts.
  - Long-running or reliability-critical jobs: push to Redis-backed queue workers (RQ/Celery) for retries and observability.
  - Workers own side effects: pyautogui writes, opening browser, calling Twilio; the API only triggers.

- Camera + ML (MediaPipe)
  - Runs in its own thread/process (or a separate process to avoid GL conflicts).
  - Emits events (gaze, blink) into the scheduler/worker via an internal pub/sub interface or in-memory queue.

- Automation (pyautogui, browser control)
  - All UI automation occurs in worker context.
  - Workers must verify a short countdown and explicit user confirmation before typing sensitive data.

- Optional external systems
  - Twilio for calls/SMS, WhatsApp Web via browser automation or URL open + pyautogui press Enter.
  - Redis for durable job queues & rate-limiting.
  - S3 for storing artifacts (screenshots, logs) if enabled.

### Data flows & API contract

Primary endpoints (examples):
- POST /perform_action
  - body: { action: string, text?: string, options?: {} }
  - API validates and either: (a) creates a background thread, or (b) enqueues a job (if queue configured).
  - Returns job id or immediate status.

- GET /health
  - returns component statuses: camera, model present, queue connected, overlay availability.

- WebSocket / SSE / Server-Sent Events (optional)
  - Push live status updates to the UI: job progress, gaze events, errors.

Event flows:
1. User clicks TYPE EXTERNAL or triggers command via gaze/voice.
2. Browser sends /perform_action to Flask.
3. Flask enqueues the job and returns job id.
4. Scheduler picks job and spawns worker (local thread or queue worker).
5. Worker performs countdown → pyautogui.write(text) → reports success/failure.
6. Worker pushes status updates back via persistence or pub/sub to API for UI streaming.

### Threading, concurrency & safety

- Keep Flask request handling synchronous and non-blocking: offload side-effects to workers.
- Recommended safe worker patterns:
  - Local dev: use concurrent.futures.ThreadPoolExecutor with bounded worker pool and timeouts.
  - Production/robust: use a durable queue (Redis + RQ/Celery) and dedicated worker processes.
- MediaPipe and camera capture often require being run in their own process (avoid conflicts with GUI event loops and global interpreters). Consider multiprocessing or a small dedicated service.
- pyautogui is inherently stateful and interacts with OS input — serialize automation tasks and avoid overlapping writes. Use a per-machine lock to ensure only one active typing flow at a time.

### Reliability, retries & failure modes

- Short failures (timing out typing because focus lost): implement retries with backoff and user-visible error messages.
- External API failures (Twilio): use exponential retry and circuit-breaker pattern; surface errors to UI.
- MediaPipe model missing: fall back with a clear error and optional manual controls.
- Queue and worker observed failure modes:
  - If worker crashes, ensure jobs are re-queued or marked failed.
  - Use a heartbeat/monitor to restart crashed workers automatically.

### Scaling & deployment patterns

Single-machine (default):
- Flask + worker threads + camera + pyautogui all on same host. Best for privacy and simple setups.

Robust/distributed:
- Run Flask web server statelessly (behind a reverse proxy).
- Use Redis-backed job queue for automation tasks. Workers run on the same LAN host that has display access (if running automation) or on dedicated machines where UI automation is permitted.
- For camera/MediaPipe heavy workloads, run a dedicated process per camera, or aggregate frames on an edge device and send summarized events to the API.

Docker notes:
- If containerizing automation with pyautogui, you'll need X11/device passthrough and appropriate permissions for synthetic input. Prefer host deployments for desktop automation.

Example deployment options:
- Local-only: python main.py (with venv)
- Containerized UI + host worker: UI in Docker, worker on host with DISPLAY/X11 access
- Horizontally scaled: Flask + Redis + Workers (workers must have access to display if they use pyautogui)

### Observability & testing

- Telemetry:
  - Add structured logs for actions and worker events.
  - Expose /metrics (Prometheus) for worker queue depth, job success/fail counts, camera frame rate.
- Health checks:
  - `GET /health` should include camera, model, queue connectivity and overlay status.
- Testing:
  - Unit test detection logic and command parsing by isolating MediaPipe wrappers and using recorded frames.
  - Mock pyautogui and webbrowser in unit tests for automation behavior.
  - Integration tests for queue behavior: run local Redis, enqueue jobs, assert expected worker side-effects via mocks or stubs.

### Security & privacy considerations (architecture-specific)

- Local-first: keep MediaPipe and processing local by default. Only upload artifacts when explicitly configured.
- Limit sensitive actions:
  - Require an explicit user-confirmed countdown for TYPE EXTERNAL or any flow that types secrets.
  - Provide an "emergency-only" configuration toggle for automatic dialing/sending.
- Protect API access:
  - For local deployments, bind to localhost by default. If exposing the API, add authentication (API keys) and TLS.
- Secrets:
  - Read Twilio credentials from environment variables or a secure secrets store; do not persist to repo.

### Practical recommendations (quick checklist)

- Enforce a bounded worker pool or queue to avoid resource exhaustion.
- Isolate camera/ML into a process to avoid blocking the web server or causing rendering conflicts.
- Use a job queue (Redis) when you need reliable retries, monitoring, or horizontal workers.
- Keep all OS input side-effects behind workers with single-writer guarantees (locks, job serialization).
- Add /health and basic metrics early — they make debugging and automation much easier.

---
If you want, I can:
- Produce an updated README patch replacing the current System Architecture block in the repo, or
- Create a separate architecture.md with diagrams, sequence diagrams and actionable runbook steps.
Which would you prefer and would you like the Mermaid sequence diagram (user → API → worker → pyautogui) added as well?
