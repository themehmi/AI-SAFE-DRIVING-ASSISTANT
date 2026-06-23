# OCULL — AI Safe Driving Assistant

> A browser-based driver safety copilot powered by real-time computer vision and a voice-activated LLM assistant.

[![Build Status](https://img.shields.io/badge/build-passing-brightgreen)](https://github.com/themehmi/OCULL-AI-DRIVING-ASSISTANT)
[![Version](https://img.shields.io/badge/version-1.0.0-blue)](https://github.com/themehmi/OCULL-AI-DRIVING-ASSISTANT)
[![License](https://img.shields.io/badge/license-MIT-yellow)](https://github.com/themehmi/OCULL-AI-DRIVING-ASSISTANT)
[![Python](https://img.shields.io/badge/python-3.10-blue)](https://github.com/themehmi/OCULL-AI-DRIVING-ASSISTANT)

---

## Table of Contents

1. [Overview](#overview)
2. [Live Demo](#live-demo)
3. [Features](#features)
4. [System Architecture](#system-architecture)
5. [How It Works](#how-it-works)
   - [Drowsiness Detection Loop](#1-drowsiness-detection-loop-apiframe)
   - [Voice Copilot Loop](#2-voice-copilot-loop-apivoice)
   - [Music Playback](#3-voice-controlled-music)
   - [Status Polling](#4-status-polling-apistatus)
6. [Project Structure](#project-structure)
7. [API Reference](#api-reference)
8. [Configuration](#configuration)
9. [Getting Started](#getting-started)
   - [Local Development](#local-development)
   - [Docker Deployment](#docker-deployment)
   - [Hugging Face Spaces](#hugging-face-spaces)
10. [Dependencies](#dependencies)
11. [Known Limitations & Notes](#known-limitations--notes)
12. [Contributing](#contributing)
13. [License](#license)

---

## Overview

**OCULL** is a browser-deployable AI driving assistant named **Lara**. It monitors the driver's eyes through the device webcam in real time, calculates eye closure duration using the Eye Aspect Ratio (EAR) algorithm, and escalates through a tiered alert system — from a soft verbal nudge to a simulated emergency response — if sustained drowsiness is detected.

Alongside safety monitoring, Lara acts as a hands-free voice copilot. Drivers can ask road-related questions or request music by voice without ever touching the screen, keeping attention on the road.

The project demonstrates how lightweight, browser-deployable computer vision (no specialized hardware required) combined with an LLM-backed voice interface can meaningfully reduce distracted and drowsy driving.

---

## Live Demo

A hosted instance is running on Hugging Face Spaces:

**👉 https://huggingface.co/spaces/themehmi/AI-Safe-Driving-Assistant**

Open it in a webcam- and microphone-enabled browser to try the drowsiness monitor and voice copilot directly — no installation needed.

---

## Features

| Feature | Description |
|---|---|
| 😴 Real-time drowsiness detection | Computes EAR from webcam frames via MediaPipe facial landmarks to detect prolonged eye closure |
| 🚦 Tiered alert escalation | Wake-up nudge at 5 s, serious warning at 8 s, tracks repeated drowsy events across the session |
| 🆘 Simulated emergency response | Triggers a simulated 911 call if the driver is unresponsive to a rest prompt for 15 seconds |
| 🌑 Low-light enhancement | Applies CLAHE preprocessing to frames before analysis, improving reliability in dim conditions |
| 📈 EAR signal smoothing | Exponential Moving Average (EMA) filters out jitter caused by low-resolution cameras |
| 🗣️ Wake-word voice copilot | Say "Lara" to activate; the assistant answers driving and road-related questions via Llama 3.1 8B (NVIDIA NIM) |
| 🎵 Voice-controlled music | Search and stream songs by voice via YouTube Music (ytmusicapi), with yt-dlp as a fallback |
| 🎛️ Full playback controls | Play, pause, resume, skip, previous, volume up/down, mute/unmute by voice |
| 👤 User authentication | Signup/login/password-reset flows backed by MongoDB Atlas with bcrypt-hashed passwords |
| 🐳 Docker-ready | Ships with a Dockerfile configured for Gunicorn and Hugging Face Spaces deployment |

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Browser (index.html)                        │
│                                                                     │
│  ┌────────────────┐   ┌───────────────────┐   ┌─────────────────┐  │
│  │  Webcam Loop   │   │  Web Speech API   │   │  Native Audio   │  │
│  │  (base64 POST) │   │  (mic → text)     │   │  Player         │  │
│  └───────┬────────┘   └────────┬──────────┘   └────────▲────────┘  │
│          │                     │                        │           │
└──────────┼─────────────────────┼────────────────────────┼───────────┘
           │ POST /api/frame     │ POST /api/voice        │ video_ids / actions
           ▼                     ▼                        │
┌──────────────────────────────────────────────────────────────────────┐
│                         Flask Backend (app.py)                       │
│                                                                      │
│  ┌─────────────────────────┐   ┌──────────────────────────────────┐  │
│  │  /api/frame             │   │  /api/voice                      │  │
│  │                         │   │                                  │  │
│  │  1. CLAHE enhancement   │   │  1. Wake-word detection ("Lara") │  │
│  │  2. MediaPipe landmark  │   │  2. Dialogue state machine       │  │
│  │     detection           │   │  3. Media command dispatch       │  │
│  │  3. EAR computation     │   │  4. Song search (ytmusicapi /    │  │
│  │  4. EMA smoothing       │   │     yt-dlp fallback)             │  │
│  │  5. Tiered alert logic  │   │  5. LLM Q&A (Llama 3.1 8B via   │  │
│  │  6. Dialogue triggers   │   │     NVIDIA NIM)                  │  │
│  └─────────────────────────┘   └──────────────────────────────────┘  │
│                                                                      │
│  ┌──────────────┐   ┌────────────────────┐   ┌──────────────────┐   │
│  │  /api/status │   │  /api/music_url    │   │  Auth routes     │   │
│  │  (polling)   │   │  (pytubefix        │   │  /login /signup  │   │
│  │              │   │   stream URL)      │   │  /logout etc.    │   │
│  └──────────────┘   └────────────────────┘   └──────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
           │                     │                        │
           ▼                     ▼                        ▼
    ┌──────────────┐   ┌──────────────────┐   ┌──────────────────────┐
    │  MediaPipe   │   │  NVIDIA NIM API  │   │  MongoDB Atlas       │
    │  Face        │   │  (Llama 3.1 8B)  │   │  (users collection)  │
    │  Landmarker  │   │                  │   │                      │
    └──────────────┘   └──────────────────┘   └──────────────────────┘
```

---

## How It Works

### 1. Drowsiness Detection Loop (`/api/frame`)

The browser continuously captures webcam frames, base64-encodes them, and POSTs them to `/api/frame`. On the server:

**Step 1 — Low-Light Enhancement**
The frame is converted to LAB color space and processed with CLAHE (Contrast Limited Adaptive Histogram Equalization) on the L-channel to improve visibility in dim environments before any analysis is performed.

**Step 2 — Landmark Detection**
OpenCV decodes the image and MediaPipe's `FaceLandmarker` (Tasks API, `face_landmarker.task`) extracts 468 facial landmarks from the frame. If the model file is absent at startup, it is automatically downloaded from Google's model registry.

**Step 3 — EAR Computation**
Six landmark indices per eye (mirroring the classic six-point dlib eye model) are used:

```
RIGHT_EYE = [33, 160, 158, 133, 153, 144]
LEFT_EYE  = [362, 385, 387, 263, 373, 380]
```

The Eye Aspect Ratio is calculated as:

```
EAR = (||p2−p6|| + ||p3−p5||) / (2 × ||p1−p4||)
```

where `p1`–`p6` are the six landmark points in pixel space. A high EAR value (~0.3+) indicates open eyes; the value drops sharply toward zero as eyes close.

**Step 4 — EMA Signal Smoothing**
Raw EAR values are smoothed with an Exponential Moving Average to suppress jitter from low-resolution cameras:

```
smoothed_EAR = α × raw_EAR + (1 − α) × smoothed_EAR
```

`α` is set to `0.4` and the detection threshold to `0.20`.

**Step 5 — Tiered Alert Logic**

| Eyes-closed Duration | State | Action |
|---|---|---|
| < threshold | `NORMAL` | No action; counters reset |
| ≥ 5 seconds | `LEVEL_1` | Lara speaks a wake-up nudge |
| ≥ 8 seconds | `LEVEL_3` | Drowsy counter increments; warning issued |
| 3+ drowsy events | `LEVEL_3` (escalated) | Lara asks via voice: "Should I remind you to pull over?" |
| No reply in 15 s | `CRITICAL` | Simulated emergency call (`action: call_emergency`) |

All state is held in a global `system_status` dictionary and a dialogue state machine (`_dialogue_state`). This design keeps the server effectively stateless per-request while maintaining session continuity through shared in-memory state.

---

### 2. Voice Copilot Loop (`/api/voice`)

The browser's Web Speech API transcribes spoken audio to text and POSTs it to `/api/voice`. The backend processes it through a layered state machine:

**Layer 1 — Active Dialogue Override**
If a drowsiness dialogue is active (`asking_rest` or `asking_song`), incoming speech is interpreted as a direct answer to the pending question rather than a new command. Positive responses to a rest prompt lead to a pull-over recommendation; negative responses transition to `asking_song`, prompting the user to name a song.

**Layer 2 — Wake-Word Detection**
Outside of active dialogues, the backend listens for the wake word `"lara"`. If detected, the text following it is extracted as the command. If no command follows (just `"lara"` alone), the assistant responds "Yes?" and enters `active_listening` mode for the next utterance.

**Layer 3 — Media Commands**
Simple playback commands are matched by keyword and dispatched as action strings to the front end:

```
pause → pause_native      resume/play → resume_native
stop  → stop_native       next/skip   → next_native
back  → prev_native       volume up   → vol_up_native
volume down → vol_down_native         mute/unmute → mute/unmute_native
```

**Layer 4 — Music Search**
`"play <song>"` commands trigger a YouTube Music search via `ytmusicapi`. A list of up to 20 video IDs is returned to the browser, which plays audio natively without routing through the server.

**Layer 5 — LLM Q&A**
Anything that doesn't match the above is forwarded to `meta/llama-3.1-8b-instruct` via the NVIDIA NIM API with a strict system prompt restricting responses to driving, road, and music topics. If no `API_KEY` is configured, the app runs in Offline Mode and skips LLM responses.

---

### 3. Voice-Controlled Music

Song searches are resolved by `ytmusicapi`, which queries YouTube Music for the requested track. A custom `curl_cffi` Chrome-impersonating session is injected into `ytmusicapi` to bypass bot-detection on cloud-hosted environments (such as Hugging Face Spaces).

If `ytmusicapi` is blocked or fails, the backend falls back to a `yt-dlp` CLI search:

```
yt-dlp --impersonate chrome --force-ipv4 --get-id --flat-playlist "ytsearch10:<query>"
```

The front end plays audio using `pytubefix` to resolve a direct stream URL from a video ID via `/api/music_url`, rather than embedding a YouTube player — allowing volume and playback control without iframe restrictions.

---

### 4. Status Polling (`/api/status`)

A lightweight endpoint the browser can poll to read the current driver state:

```json
{
  "state": "NORMAL",
  "ear": 0.34,
  "drowsy_counter": 1
}
```

This powers any on-screen safety indicators and the eye-tracking overlay rendered in the browser.

---

## Project Structure

```
OCULL-AI-DRIVING-ASSISTANT/
│
├── app.py                  # Flask application — all backend logic
│   ├── Auth routes         #   /signup, /login, /logout, /reset_password
│   ├── /api/frame          #   Drowsiness detection endpoint
│   ├── /api/voice          #   Voice copilot + dialogue state machine
│   ├── /api/music_url      #   Resolves YouTube stream URLs via pytubefix
│   ├── /api/status         #   System state polling
│   └── /api/chat           #   Text-based chat endpoint (UI fallback)
│
├── templates/              # Jinja2 HTML front end
│   ├── index.html          #   Main dashboard (camera feed, voice UI, music player)
│   ├── login.html          #   Login page
│   ├── signup.html         #   Signup page
│   ├── reset_password.html #   Password reset page
│   └── settings.html       #   User settings page
│
├── face_landmarker.task    # MediaPipe Face Landmarker model (auto-downloaded if missing)
├── requirements.txt        # Python dependencies
├── packages.txt            # System-level packages (for Hugging Face Spaces)
├── Dockerfile              # Container definition (Gunicorn, port 7860)
├── README.md               # Project readme
└── README_DOCKER.md        # Docker-specific deployment notes
```

---

## API Reference

### `GET /`
Renders the main dashboard. Requires authentication; redirects to `/login` if no session.

---

### `POST /api/frame`
Processes a webcam frame for drowsiness detection.

**Request body (JSON)**
```json
{
  "image": "data:image/jpeg;base64,<base64-encoded frame>"
}
```

**Response (JSON)**
```json
{
  "ear": 0.32,
  "state": "NORMAL",
  "drowsy_counter": 0,
  "alert": null,
  "action": null,
  "left_eye": [[x, y], ...],
  "right_eye": [[x, y], ...]
}
```

`state` values: `NORMAL` | `LEVEL_1` | `LEVEL_3` | `CRITICAL`

`action` values: `null` | `"call_emergency"`

`alert` — string spoken aloud by the front end via the Web Speech API when non-null.

---

### `POST /api/voice`
Processes a voice command transcribed to text.

**Request body (JSON)**
```json
{
  "text": "lara play something energetic"
}
```

**Response (JSON)**
```json
{
  "speak": "Playing something energetic.",
  "action": "play_native",
  "video_ids": ["dQw4w9WgXcQ", "..."],
  "title": "something energetic"
}
```

`action` values: `play_native` | `pause_native` | `resume_native` | `stop_native` | `next_native` | `prev_native` | `vol_up_native` | `vol_down_native` | `mute_native` | `unmute_native` | `call_emergency`

---

### `GET /api/music_url?video_id=<id>`
Resolves a YouTube video ID to a direct audio stream URL using `pytubefix`.

**Response (JSON)**
```json
{
  "url": "https://...",
  "title": "Song Title",
  "artist": "Artist Name",
  "thumbnail": "https://..."
}
```

---

### `GET /api/status`
Returns the current driver safety state.

**Response (JSON)**
```json
{
  "state": "NORMAL",
  "ear": 0.31,
  "drowsy_counter": 0
}
```

---

### `POST /api/chat`
Text-based interface to Lara (UI fallback; mirrors `/api/voice` LLM logic).

**Request body (JSON)**
```json
{ "prompt": "What is the speed limit on a highway?" }
```

**Response (JSON)**
```json
{
  "response": "Highway speed limits typically range from 100–130 km/h depending on the country.",
  "speak": "Highway speed limits typically range from 100–130 km/h depending on the country."
}
```

---

### Auth Routes

| Route | Method | Description |
|---|---|---|
| `/signup` | GET, POST | Create a new user account |
| `/login` | GET, POST | Authenticate and start a session |
| `/logout` | GET | Destroy the current session |
| `/reset_password` | GET, POST | Reset password by username |
| `/settings` | GET | User settings page (login required) |

---

## Configuration

Create a `.env` file in the project root:

```env
API_KEY=your_nvidia_nim_api_key_here
SECRET_KEY=your_flask_secret_key_here
MONGO_URI=mongodb+srv://<user>:<password>@<cluster>.mongodb.net/
```

| Variable | Required | Description |
|---|---|---|
| `API_KEY` | Optional | NVIDIA NIM API key for the Llama 3.1 8B model. If omitted, Lara runs in **Offline Mode** (no LLM Q&A). |
| `SECRET_KEY` | Required | Flask session secret. Use a long random string in production. |
| `MONGO_URI` | Required | MongoDB Atlas connection string. Used for user authentication. |

---

## Getting Started

### Local Development

**Prerequisites**
- Python 3.10+
- `build-essential`, `cmake`, and `g++` (required to compile MediaPipe native extensions)
- A webcam-enabled browser
- An NVIDIA NIM API key (optional — Lara works without it in Offline Mode)

**Steps**

```bash
# 1. Clone the repository
git clone https://github.com/themehmi/OCULL-AI-DRIVING-ASSISTANT.git
cd OCULL-AI-DRIVING-ASSISTANT

# 2. Create and activate a virtual environment (recommended)
python3 -m venv venv
source venv/bin/activate       # Windows: venv\Scripts\activate

# 3. Install Python dependencies
pip install -r requirements.txt

# 4. Configure environment variables
cp .env.example .env           # or create .env manually (see Configuration section)

# 5. Run the development server
python app.py
```

The app will be available at **http://localhost:7860**.

Open it in a browser, grant webcam and microphone permissions, say **"Lara"** to activate the voice assistant, and keep the tab open while driving to enable continuous drowsiness monitoring.

> **Note:** The MediaPipe Face Landmarker model (`face_landmarker.task`) is downloaded automatically on first run if it isn't present in the project root.

---

### Docker Deployment

The included `Dockerfile` builds a production image running Gunicorn with multiple workers.

```bash
# Build the image
docker build -t ocull-driving-assistant .

# Run the container
docker run -p 7860:7860 --env-file .env ocull-driving-assistant
```

The app is available at **http://localhost:7860** after the container starts.

---

### Hugging Face Spaces

The repository is pre-configured for Hugging Face Spaces deployment:

- `Dockerfile` exposes port `7860` and uses Gunicorn for production serving.
- `packages.txt` lists required system-level packages (installed by the Spaces build runner).
- `ProxyFix` middleware is applied to correctly handle HTTPS and iframe embedding in the Spaces environment.
- The `pkg_resources` monkey-patch at the top of `app.py` handles Hugging Face's stripped-down Python environment where `pkg_resources` is unavailable.

To deploy, push the repository to a Hugging Face Space with the **Docker** SDK selected and set `API_KEY`, `SECRET_KEY`, and `MONGO_URI` as Space Secrets.

---

## Dependencies

### Python Packages (`requirements.txt`)

| Package | Purpose |
|---|---|
| `Flask 3.0.3` | Web framework and routing |
| `opencv-python-headless 4.9.0.80` | Image decoding, CLAHE preprocessing |
| `numpy 1.26.4` | Landmark coordinate math, EAR computation |
| `mediapipe` | Face Landmarker model (landmark extraction) |
| `openai 1.30.1` | NVIDIA NIM API client (Llama 3.1 8B) |
| `python-dotenv 1.0.1` | `.env` file loading |
| `gunicorn 22.0.0` | Production WSGI server |
| `curl-cffi` | Chrome-impersonating HTTP session for YouTube Music |
| `httpx <0.28.0` | HTTP client (pinned for OpenAI SDK compatibility) |
| `ytmusicapi` | YouTube Music search API |
| `pymongo` | MongoDB Atlas client |
| `requests` | General HTTP requests |
| `yt-dlp` | YouTube search and download fallback |
| `pytubefix` | Resolves direct YouTube audio stream URLs |
| `setuptools` | Build tooling for native extensions |

### System Packages (`packages.txt`)

Required for MediaPipe compilation and runtime on Linux (relevant for Hugging Face Spaces):

```
build-essential
cmake
g++
```

---

## Known Limitations & Notes

**Music playback on cloud hosts**
YouTube actively blocks automated audio requests from cloud IP ranges. The app uses a Chrome-impersonating `curl_cffi` session for `ytmusicapi` and falls back to `yt-dlp` with `--impersonate chrome` when that fails. Playback may still intermittently fail on Hugging Face Spaces due to HTTPS egress sandbox restrictions.

**In-memory global state**
`system_status`, `_dialogue_state`, and tracking variables (`start_closed_time`, `smoothed_ear`, etc.) are stored in Python globals. This means the drowsiness state is shared across all browser sessions connected to the same server process. This is intentional for single-user deployment but would require per-session state management (e.g., via Redis) for a multi-user production deployment.

**EAR threshold tuning**
The default EAR threshold of `0.20` works well for most webcam setups, but eye geometry varies between individuals. Drivers with naturally narrower eyes may need a lower threshold; those with wider eyes may need a higher one. Threshold calibration per user is a potential future improvement.

**No face detected**
If no face is detected in a frame (e.g., driver looks far to the side), the system resets to `NORMAL` state and clears the closure timer. This prevents false negatives from head turns but also means a completely obstructed camera will not trigger alerts.

**Offline Mode**
If `API_KEY` is not set, Lara responds to voice commands that match known media actions (pause, play, skip, volume) but cannot answer open-ended driving questions. Music search still works.

**Session security**
Session cookies are set with `Secure=True` and `SameSite=None` to support cross-origin iframe embedding on Hugging Face Spaces. In a non-iframe deployment, `SameSite=Lax` is more appropriate.

---

## Contributing

Contributions are welcome. To get started:

1. Fork the repository.
2. Create a feature branch: `git checkout -b feature/your-feature-name`.
3. Make your changes with clear, focused commits.
4. Push to your fork and open a Pull Request describing the change and how the relevant behavior (drowsiness detection, voice, music) was verified.

Please keep PRs focused on a single change and include testing notes where applicable.

---

## License

This project is licensed under the **MIT License**. See the `LICENSE` file for details.

---

*Documentation generated from source: [github.com/themehmi/OCULL-AI-DRIVING-ASSISTANT](https://github.com/themehmi/OCULL-AI-DRIVING-ASSISTANT)*
