# Intentix

Assistive eye-, blink-, and voice-controlled toolkit for hands-free computer control, lightweight automation, and emergency workflows.

Status: WIP / Alpha — update to "Beta" or "Production" when appropriate.

---

## Table of Contents

- [About](#about)
- [Tech Stack](#tech-stack)
- [Key Features](#key-features)
- [System Architecture](#system-architecture)
  - [Architecture Diagram (Mermaid)](#architecture-diagram-mermaid)
  - [Components & Data Flow](#components--data-flow)
- [Repository Structure](#repository-structure)
- [Text Tag (HTML textarea) — how text flows in Intentix](#text-tag-html-textarea---how-text-flows-in-intentix)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Clone](#clone)
  - [Quick Install & Run](#quick-install--run)
  - [Docker (optional)](#docker-optional)
- [Configuration & Environment Variables](#configuration--environment-variables)
- [Usage Examples & Workflows](#usage-examples--workflows)
  - [TYPE EXTERNAL (end-to-end)](#type-external-end-to-end)
  - [Emergency flows (WhatsApp / Twilio)](#emergency-flows-whatsapp--twilio)
- [Testing & CI Recommendations](#testing--ci-recommendations)
- [Permissions & Platform Notes](#permissions--platform-notes)
- [Troubleshooting](#troubleshooting)
- [Security & Privacy Considerations](#security--privacy-considerations)
- [Development & Contributing](#development--contributing)
- [License & Contact](#license--contact)

---

## About

Intentix provides a local web UI and automation backend to enable hands-free interactions using webcam-based gaze/blink detection (MediaPipe), voice commands, and scripted UI automation (pyautogui). It bundles assistive features and emergency helpers (WhatsApp/Twilio) and is intended for research, accessibility prototypes, and small deployments.

Repository: https://github.com/adityafilesx/Intentix

---

## Tech Stack

Core languages, frameworks and libs used in the project:

- Python 3.8+
  - Flask — web server and UI templates
  - OpenCV (cv2) — camera capture and image utilities
  - MediaPipe (tasks/python, vision, face_landmarker) — face, gaze & blink detection
  - pyautogui — UI automation (typing, key presses)
  - SpeechRecognition / voice processing modules — voice command capture
  - Twilio (optional) — programmatic calls/SMS
  - PyQt5 (optional) — overlay cursor UI
- Frontend
  - HTML / CSS / JavaScript (vanilla) — templates under `templates/` and `static/`
- Optional/Supporting
  - Docker (for containerized development)
  - Redis / message queue (recommended for scale)
  - S3 / object storage (for artifacts)

---

## Key Features

- Real-time gaze and blink detection (MediaPipe) for hands-free control
- Virtual on-screen keyboard + editable textarea as main text input ("Text Tag")
- "Type external" automation: type textarea contents into any focused OS application
- Voice command capture and execution (start/stop, confirm/cancel)
- Emergency helpers: open WhatsApp Web with pre-filled message, Twilio call helper
- Lightweight Flask UI for local control and configuration
- Background threads for non-blocking automation

---

## System Architecture

Intentix is organized into UI, server/API, worker threads for automation, and local device integrations (camera & input automation). The design favors simplicity and local execution (privacy-first).

### Architecture Diagram

> Paste this diagram into a renderer that supports Mermaid (GitHub may render mermaid in some contexts).

```mermaid
flowchart LR
  subgraph Client
    FE[Web UI (Flask templates + JS)]
  end

  subgraph Server
    API[Flask App (main.py)]
  end

  subgraph Workers
    Worker[Background threads / automation]
  end

  subgraph Local
    Cam[Camera + MediaPipe]
    Input[pyautogui (keyboard/mouse)]
  end

  subgraph External
    Twilio[Twilio (optional)]
    Browser[Web Browser (WhatsApp flow)]
  end

  FE -->|HTTP fetch / JS| API
  API -->|spawns| Worker
  Worker --> Input
  Cam --> API
  API --> Twilio
  API --> Browser
```

### Components & Data Flow

- Web UI (templates + static)
  - Renders virtual keyboard and `#output-area` textarea.
  - Provides buttons triggering actions (type, search, emergency).
- Flask App (`main.py`)
  - Serves UI and handles `/perform_action` POST endpoint.
  - Parses `action` and `text`, dispatches jobs.
- Background workers (threads)
  - Execute `execute_type_external`, `auto_send_whatsapp`, `make_twilio_call`, etc.
  - Avoid blocking the web server while performing IO or UI automation.
- Camera + ML (MediaPipe)
  - Detects gaze and blinks to trigger automated UI interactions.
- pyautogui
  - Sends keystrokes or presses Enter to the currently focused window (used for TYPE EXTERNAL and WhatsApp flows).
- External tools
  - Twilio for calls (optional)
  - Browser to open Web WhatsApp pre-filled URL

---

## Repository Structure (typical / recommended)

A simplified view (adjust if your code layout differs):

- main.py — Flask app, action handlers, automation helpers
- templates/
  - index.html — main UI
- static/
  - styles.css, scripts
- core/ or modules:
  - gaze_tracker.py, blink_detector.py, voice_processor.py, fatigue_monitor.py
- automation/
  - executor.py, email_service.py, browser_control.py
- utils/
  - helpers.py, logger.py
- requirements.txt
- config.json / .env (optional)
- README.md

---

## Text Tag (HTML textarea) — how text flows in Intentix

Central element: `<textarea id="output-area" placeholder="> SELECT A MODE..."></textarea>`

- This is the primary editable field where users compose text or where text arrives from voice commands or processing pipelines.
- Client-side helpers (JS) manipulate this textarea (`add()`, `back()`, `clr()`, `moveCursor()`).
- The "TYPE EXTERNAL" action sends the textarea content to the server (`/perform_action`) with `action: "type_external"`. The server spawns a thread that executes `pyautogui.write(text, interval=0.1)` after a short countdown to allow the user to focus a target app.

---

## Getting Started

### Prerequisites

- Python 3.8 or later
- pip
- Webcam (for gaze/blink features)
- (Optional) Microphone for voice features
- (Optional) PyQt5 for overlay features
- (Optional) Twilio account & credentials for call flows

### Clone

```bash
# HTTPS
git clone https://github.com/adityafilesx/Intentix.git
cd Intentix

# or SSH
git clone git@github.com:adityafilesx/Intentix.git
cd Intentix
```

### Quick Install & Run

1. Create virtual environment and install dependencies:

```bash
python -m venv .venv
# macOS/Linux
source .venv/bin/activate
# Windows
# .venv\Scripts\activate

pip install -r requirements.txt
```

2. (Optional) Copy environment template:

```bash
cp .env.example .env
# Edit .env with your values (if .env.example exists)
```

3. Run the app:

```bash
python main.py
```

4. Open the UI in a browser:

```
http://127.0.0.1:5000/
```

Notes:
- If the MediaPipe face landmarker task file is not present, `main.py` will attempt to download it to the repository directory on first run.
- If PyQt5 is not installed, overlay features will be disabled (the app prints a warning).

### Docker (optional)

You can containerize the app. Example Dockerfile / Compose are not included here but recommended steps:

- Build a Python image with required system packages (ffmpeg, libsm6, libxext6, etc.)
- Install requirements.txt
- Expose port 5000
- Mount device access for webcam if needed (use `--device` on docker run, or use host networking)

---

## Configuration & Environment Variables

Important settings (where/how they're stored may vary — update to match project):

- `TWILIO_SID` — Twilio Account SID (optional)
- `TWILIO_AUTH` — Twilio Auth Token (optional)
- `TWILIO_PHONE` — Twilio source phone number (optional)
- `SETTINGS["emergency_contact"]` — emergency phone number used by WhatsApp/Twilio flows
- `BLINK_SENSITIVITY` or similar — blink threshold setting (UI exposes a setting)
- `OVERLAY_ENABLED` — enable/disable cursor overlay
- Logging level / file path

Store secrets securely (local `.env` not committed, use GitHub Secrets or Vault in CI/CD).

---

## Usage Examples & Workflows

### TYPE EXTERNAL (end-to-end)

1. Focus the external application window (Notepad, text editor, web form).
2. In Intentix UI, compose or paste text into the textarea `#output-area`.
3. Click "TYPE EXTERNAL".
4. The UI will trigger a short countdown to let you focus the target window.
5. After countdown, server thread calls `pyautogui.write(text, interval=0.1)` — the text is typed into the focused window.

### Emergency flows (WhatsApp / Twilio)

- WhatsApp:
  - The app opens a `https://web.whatsapp.com/send?phone=<number>&text=<encoded_message>` URL.
  - If WhatsApp Web session is logged in, pyautogui presses Enter to send (after a short wait).
- Twilio:
  - If Twilio config is present, `make_twilio_call()` can place a call using TwiML to speak an emergency message.

---

## Testing & CI Recommendations

- Unit tests: add tests for core modules (gaze logic, command parsing, automation wrappers).
- Integration tests: use a headless environment or mocks for `pyautogui` and `webbrowser`.
- CI pipeline (example):
  - lint → unit tests → build artifact → (optional) publish
- For pyautogui-related tests, mock `pyautogui.write` and other side-effecting calls.

---

## Permissions & Platform Notes

- Accessibility / input control:
  - On macOS you may need to grant Accessibility permission to the Python interpreter to allow pyautogui to send keystrokes.
  - On Linux/X11, ensure appropriate X11 access or use a compositor that permits synthetic input.
- Webcam access:
  - Browser-based camera demos use the local webcam via OpenCV; ensure drivers and permissions are set.
- Running as root is not recommended for desktop automation.

---

## Troubleshooting

- pyautogui does not type into window:
  - Ensure the target window is focused and accepts keyboard input.
  - Check OS-level permissions (Accessibility).
- WhatsApp send fails:
  - Ensure web.whatsapp.com is logged in and not blocked by authentication prompts.
  - Timing may vary — increase wait times before pressing Enter.
- MediaPipe model download fails:
  - Check network, available disk space, and retry. You can download the model manually and place it at the path printed by the app.
- Webcam not found:
  - Confirm device index, drivers, and that another process is not using the camera.

---

## Security & Privacy Considerations

- Intentix performs local processing by default (MediaPipe, pyautogui). Be careful if sharing or uploading logs that contain PII.
- Do not commit credentials into the repository. Use environment variables or secret stores.
- When using Twilio, secure API credentials and restrict their usage to required endpoints/IPs.
- Be aware that pyautogui will type on whichever window is focused — ensure you do not inadvertently type sensitive data into unintended places.

---

## Development & Contributing

- Fork the repo, create a feature branch, and open a Pull Request with description and tests.
- Keep changes modular and add/update docs for new behavior.
- Suggested pre-commit checks:
  - flake8/black for Python
  - eslint / prettier for front-end assets (if applicable)
- If adding integrations (new services), include configuration instructions and environment variables in README.

---

## License & Contact

- License: Add your LICENSE file (e.g., MIT or Apache-2.0).
- Maintainer / Contact: adityafilesx
- Repository: https://github.com/adityafilesx/Intentix

---

If you want, I can:
- Generate a `.env.example` with the environment variables above,
- Produce a sample Dockerfile + docker-compose.yml for local dev (with webcam passthrough notes),
- Add CI workflow (GitHub Actions) template for linting and tests.
