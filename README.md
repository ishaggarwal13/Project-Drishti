# 👁️ Project Drishti (Drishti Guard)

**AI-powered crowd safety and incident-response platform for large events.**

*Drishti* (दृष्टि) means "vision" in Sanskrit/Hindi. The project turns ordinary camera feeds into early warnings: it counts people, measures crowd density, predicts bottlenecks 15–20 minutes ahead, detects safety anomalies (fire, fights, medical emergencies, stampede risk), and routes alerts and SOS requests to volunteers and administrators in real time.

---

## Table of Contents

1. [The Problem](#-the-problem)
2. [What Drishti Does](#-what-drishti-does)
3. [System Architecture](#-system-architecture)
4. [Repository Structure](#-repository-structure)
5. [Modules in Detail](#-modules-in-detail)
   - [CNS: Central Nervous System](#1-cns--central-nervous-system-cnsprediction)
   - [ADS: Anomaly Detection System](#2-ads--anomaly-detection-system-ads)
   - [UI: Next.js Frontend](#3-ui--nextjs-frontend-ui)
6. [Tech Stack](#-tech-stack)
7. [API Reference](#-api-reference)
8. [Getting Started](#-getting-started)
9. [Configuration](#-configuration)
10. [Docker and Cloud Deployment](#-docker-and-cloud-deployment)
11. [Rate-Limit (HTTP 429) Mitigation](#-rate-limit-http-429-mitigation)
12. [Known Issues and Security Notes](#-known-issues-and-security-notes)
13. [Roadmap](#-roadmap)

---

## 🎯 The Problem

At concerts, exhibitions, festivals, and religious gatherings, crowd disasters (stampedes, crushes at entrances, blocked exits) usually build up over minutes before they become visible to staff. Control rooms cannot watch every camera at once, and volunteers on the ground often learn about incidents too late or through word of mouth.

## ✨ What Drishti Does

| Capability | Description |
|---|---|
| **Live crowd counting & density** | Reads camera frames (webcam, video file, or IP camera) and uses Gemini Vision, with an OpenCV fallback, to count people and score density on a 1–10 scale. |
| **Flow & velocity tracking** | Estimates movement speed, flow direction, queue length and flow efficiency per camera. |
| **Bottleneck prediction (15–20 min ahead)** | Combines recent camera history (5-minute rolling buffer), trend analysis and Gemini reasoning to forecast congestion at specific venue locations. |
| **Anomaly detection** | A separate service classifies video frames into 12 incident categories with 5 severity levels and generates a full incident report (hazards, actions, equipment, de-escalation). |
| **Map-based analysis** | Analyzes venue maps/floor plans and Google Maps data (nearby places, traffic) to identify choke points and risk zones. |
| **SOS & dispatch** | Attendees trigger SOS from their phone with location; volunteers receive it live and acknowledge the dispatch. |
| **Role-based dashboards** | Attendee view (SOS), Volunteer dashboard (dispatches, live anomalies, crowd insights) and an Admin panel (incident management, volunteer assignment). |
| **AI situational chat** | Natural-language questions about the current state of the venue, answered by Gemini using live CNS data. |

---

## 🏗️ System Architecture

```
                         ┌────────────────────────────────────┐
  IP cameras / webcam /  │       CNS  (CNS/prediction)         │
  video files  ────────► │  Central Nervous System (Python)    │
                         │  • per-camera analysis threads      │
                         │  • Gemini Vision + OpenCV fallback  │
                         │  • velocity / flow / bottleneck AI  │
                         │  • 5-min buffer → 15–20 min forecast│
                         └──────────────┬─────────────────────┘
                                        │ cns_data_extractor.py
                                        ▼
┌──────────────────┐   REST + WebSocket  ┌─────────────────────┐
│  Next.js 15 UI   │ ◄─────────────────► │  FastAPI Bridge     │
│  (ui/)           │                     │  fastapi_bridge.py  │
│  • Attendee      │                     │  :8000              │
│  • Volunteer     │                     │  SOS · dispatch ·   │
│  • Login/Signup  │                     │  CNS endpoints · WS │
└──────────────────┘                     └─────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  ADS  (ADS/)  Anomaly Detection System: Flask + SocketIO     │
│  video_anomaly_detector.py → web_interface.py  (:5000)       │
│                           → admin_panel.py     (:5001)       │
│  Gemini 2.0 Flash classifies frames → incident reports →     │
│  admin assigns volunteers                                    │
└─────────────────────────────────────────────────────────────┘
```

**Data flow in short**

1. Cameras feed frames to the **CNS**, which analyzes each one in its own thread.
2. `cns_data_extractor.py` reshapes the raw CNS state into three frontend-ready JSON payloads: *crowd & safety insights*, *live anomaly alerts*, *live feed analysis*.
3. The **FastAPI bridge** exposes those payloads over REST, manages SOS/dispatch state, and pushes events to volunteers, admins and attendees over WebSockets.
4. The **Next.js UI** polls and subscribes to those endpoints and renders dashboards per role.
5. Independently, the **ADS** watches a video source, produces structured incident reports, and feeds an admin dashboard where volunteers are assigned.

---

## 📁 Repository Structure

```
Project-Drishti/
├── CNS/prediction/              # Crowd analysis + prediction backend (Python)
│   ├── fastapi_bridge.py        # REST/WebSocket API consumed by the UI
│   ├── live_crowd_predictor.py  # CentralNervousSystem class (core engine)
│   ├── cns_data_extractor.py    # CNS state → frontend JSON
│   ├── unified_crowd_analyzer.py# Map + video + anomaly combined analysis
│   ├── map_crowd_analyzer.py    # Venue-map / Google Maps analysis
│   ├── crowd_predictor.py       # Simple density prediction
│   ├── ai_situational_chat.py   # Gemini-powered Q&A over live venue data
│   ├── api_mitigation_service.py# Batching, caching, key rotation, 429 handling
│   ├── batch_api_service.py     # Batched Gemini requests
│   ├── local_preprocessing.py   # Local (OpenCV/CV) pre-filtering
│   ├── concert_crowd_mapper.py  # Concert-venue mapping helpers
│   ├── streamlit_crowd_ui*.py   # Streamlit control-room UIs (several iterations)
│   ├── src/                     # Vision, forecasting, maps, density-overlay modules
│   ├── scripts/, deployment/    # GCP setup script, Terraform
│   └── requirements*.txt        # Several dependency sets (fastapi / ui / minimal)
│
├── ADS/                         # Anomaly Detection System (Flask)
│   ├── main.py                  # Entry point / CLI / orchestrator
│   ├── video_anomaly_detector.py# Gemini-based frame → incident report
│   ├── web_interface.py         # Analysis dashboard + upload + logs (:5000)
│   ├── admin_panel.py           # Incident & volunteer management (:5001)
│   ├── config.py                # Env-driven settings
│   └── templates/               # dashboard.html, admin_dashboard.html
│
├── ui/                          # Next.js 15 + React 18 + Tailwind + shadcn/ui
│   └── src/
│       ├── app/                 # / · /login · /signup · /attendee · /volunteer
│       ├── components/          # sos/, volunteer/, attendee/, auth/, home/, ui/
│       ├── ai/                  # Genkit (Gemini) setup
│       └── hooks/, config/, types/
│
├── docker-compose.yml           # backend + frontend + nginx
├── Dockerfile.backend           # Python 3.11 backend image
├── nginx.conf                   # Reverse proxy
├── deploy-gcp.* / deploy-aws.sh / deploy-azure.sh / cloudbuild-*.yaml
├── start_drishti_system.bat     # One-click Windows launcher
├── test_integration.py          # End-to-end API integration tests
└── *.md                         # INTEGRATION_GUIDE, CNS_INTEGRATION_README,
                                 # DOCKER_DEPLOYMENT, GOOGLE_CLOUD_DEPLOYMENT
```

---

## 🔍 Modules in Detail

### 1. CNS — Central Nervous System (`CNS/prediction/`)

The "brain" of the platform. The `CentralNervousSystem` class in `live_crowd_predictor.py` is a multi-threaded engine that:

- **Manages cameras.** Supports local webcams and IP cameras (e.g. the Android *IP Webcam* app at `http://<ip>:8080`). A venue configuration maps each camera to a named location with coordinates, priority and bottleneck risk. The default demo venue is the *Bangalore International Exhibition Center* with cameras at the main entrance, Hall 1/2 entries, food court, parking and main corridor.
- **Analyzes every frame** with a structured Gemini prompt that returns JSON: `people_count`, `density_score`, `movement_speed`, `flow_direction`, `velocity_estimate`, `bottleneck_indicators`, `queue_length_meters`, `safety_risks`, `congestion_points`, `flow_efficiency`, `alert_level`, `immediate_action_needed`.
- **Falls back to OpenCV** (HOG-style people detection and local heuristics) when the API is rate-limited, over quota, or in "OpenCV-first" mode.
- **Runs background loops** for central analysis, bottleneck prediction, alert monitoring, and a 5-minute rolling-buffer prediction system (trend analysis → future crowd state → bottleneck probability → recommended preparation actions).
- **Detects cross-location bottlenecks** by checking whether adjacent zones also show high flow.
- **Protects the API budget** with per-call intervals, a burst limiter (max 8 calls / 10 s), caching and automatic fallbacks (see [429 mitigation](#-rate-limit-http-429-mitigation)).

Supporting pieces:

| File | Role |
|---|---|
| `cns_data_extractor.py` | Converts CNS state into the three UI payloads (insights, anomaly alerts, feed analysis). |
| `unified_crowd_analyzer.py` / `map_crowd_analyzer.py` | Combine venue-map analysis, live video and anomaly detection into one result (`analyze_complete_crowd_situation`). |
| `src/` | Gemini vision processor, forecasting models, Google Maps integration, density overlays on maps. |
| `ai_situational_chat.py` | "Ask the venue" chat for control-room staff. |
| `streamlit_crowd_ui*.py` | Streamlit operator consoles (several iterations of the same UI). |
| `run_analysis.py`, `demo_cns.py` | CLI entry points / demos. |

### 2. ADS — Anomaly Detection System (`ADS/`)

A standalone Flask + Socket.IO service that watches a live camera or an uploaded video and reports **what is wrong and what to do about it**.

**12 anomaly categories:** crowd control, security threat, medical emergency, fire hazard, structural damage, weather emergency, vehicle incident, equipment failure, unauthorized access, violence/aggression, stampede risk, evacuation needed.

**5 severity levels:** Low → Medium → High → Critical → Emergency (Level 4+ triggers an emergency modal in the dashboard).

Each detected anomaly produces an `AnomalyReport` with: situation overview, severity assessment, hazard analysis, immediate actions, equipment required, de-escalation strategy, resolution method, confidence score, and location within the frame.

- `web_interface.py` (port **5000**) — start/stop monitoring, upload a video, view statistics/logs.
- `admin_panel.py` (port **5001**) — receives incidents, de-duplicates repeats, lists volunteers (mock data for demo), assigns one or many volunteers by skill, and resolves incidents.
- `main.py` — CLI: `--web`, `--camera`, `--video <path>`, or an interactive menu.

### 3. UI — Next.js Frontend (`ui/`)

Built with **Next.js 15 (Turbopack), React 18, TypeScript, Tailwind CSS and shadcn/ui (Radix)**.

| Route | Purpose |
|---|---|
| `/` | Landing page describing the platform. |
| `/login`, `/signup` | Role-based sign-in (attendee / volunteer). |
| `/attendee` | Attendee view with an **SOS button** that sends the user's geolocation to the backend. |
| `/volunteer` | Volunteer dashboard: *My Dispatches*, *Live Anomaly Alerts*, *Crowd & Safety Insights*, with a "Start Analysis" control that kicks off CNS monitoring. Falls back to default demo data if the CNS has nothing yet. |

Also included: Firebase and Genkit (Gemini 2.0 Flash) dependencies, PWA `manifest.json`, and a `Dockerfile`/`cloudbuild.yaml` for Cloud Run.

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| AI / Vision | Google **Gemini 2.0 Flash** (vision + reasoning), Google Genkit, OpenCV, NumPy |
| Maps / location | Google Maps APIs |
| Backend (CNS) | Python, **FastAPI**, Uvicorn, WebSockets, Streamlit (operator UIs) |
| Backend (ADS) | Python, **Flask**, Flask-SocketIO, `google-generativeai` |
| Frontend | **Next.js 15**, React 18, TypeScript, Tailwind CSS, shadcn/ui, Radix UI, Recharts, React Hook Form + Zod |
| Infra | Docker, Docker Compose, Nginx, Google Cloud Run / Cloud Build, Terraform; AWS and Azure scripts |

---

## 📡 API Reference

### FastAPI bridge (`CNS/prediction/fastapi_bridge.py`, port 8000)

Interactive docs: `http://localhost:8000/docs`

| Method | Endpoint | Description |
|---|---|---|
| GET | `/` , `/api/health` | Liveness + CNS status |
| POST | `/api/sos/create` | Create an SOS (`user_id`, `location{lat,lng}`, `emergency_type`, `timestamp`); broadcasts to volunteers and admins |
| GET | `/api/sos/active` | List active SOS alerts |
| POST | `/api/dispatch/acknowledge` | Volunteer acknowledges a dispatch |
| GET | `/api/volunteer/{id}/dispatches` | Dispatches for a volunteer |
| POST | `/api/crowd/analyze` | Run crowd analysis for a location / camera URL |
| GET | `/api/crowd/insights` | Latest 10 stored insights |
| POST | `/api/cns/start` · `/api/cns/stop` | Start/stop central monitoring |
| GET | `/api/cns/status` | CNS status |
| POST | `/api/cns/start-analysis` | Start analysis with optional IP-camera config and return initial data |
| GET | `/api/cns/crowd-insights` | → UI "Crowd & Safety Insights" |
| GET | `/api/cns/anomaly-alerts` | → UI "Live Anomaly Alerts" |
| GET | `/api/cns/feed-analysis` | → UI "Live Feed Analysis & Predictions" |
| WS | `/ws/volunteer/{volunteer_id}` | Live events for a volunteer |
| WS | `/ws/admin` | Live events for admins |
| WS | `/ws/attendee/{user_id}` | Live events for an attendee |

### ADS (Flask)

- **:5000** — `POST /api/start-monitoring`, `POST /api/stop-monitoring`, `GET /api/anomalies`, `GET /api/statistics`, `POST /api/upload-video`, `GET /api/logs`, `POST /api/clear-logs`, `POST /api/send-log`
- **:5001** — `POST /api/receive-incident`, `GET /api/incidents`, `GET /api/volunteers`, `GET /api/system-status`, `POST /api/assign-volunteer`, `POST /api/assign-multiple-volunteers`, `POST /api/resolve-incident`, `GET /api/get-risk-categories`

---

## 🚀 Getting Started

### Prerequisites

- Python 3.9+ (3.11 used in Docker)
- Node.js 18+
- A **Gemini API key** ([get one](https://aistudio.google.com/app/apikey)) and optionally a **Google Maps API key**
- A webcam, video file, or phone running an IP-camera app

### 1. Clone

```bash
git clone https://github.com/AryanManojKumar/Project-Drishti.git
cd Project-Drishti
```

### 2. Backend (CNS + FastAPI bridge)

```bash
cd CNS/prediction
python -m venv .venv && source .venv/bin/activate     # Windows: .venv\Scripts\activate
pip install -r requirements-fastapi.txt
export GEMINI_API_KEY=your_key_here                    # Windows: set GEMINI_API_KEY=...
export GOOGLE_MAPS_API_KEY=your_maps_key_here
uvicorn fastapi_bridge:app --reload --port 8000
```

### 3. Frontend

```bash
cd ui
npm install
npm run dev          # dev server runs on port 9002 (see package.json)
```

> The older docs mention port 3000. The `dev` script in `ui/package.json` uses **9002**. Use whichever port you actually run, and make sure it is allowed in the backend CORS list.

### 4. (Optional) Anomaly Detection System

```bash
cd ADS
pip install -r requirements.txt
echo "GEMINI_API_KEY=your_key_here" > .env
python main.py                 # interactive menu
# python main.py --web         # dashboard at http://localhost:5000
# python main.py --video clip.mp4
python admin_panel.py          # admin console at http://localhost:5001
```

### 5. (Optional) Streamlit control-room console

```bash
cd CNS/prediction
pip install -r requirements-ui.txt
streamlit run streamlit_crowd_ui.py
```

### 6. Windows one-click start

`start_drishti_system.bat` launches backend and frontend in separate windows and runs `test_integration.py`. It expects the backend folder at `bhai ka kaam\prediction`, so edit that path to `CNS\prediction` first.

### 7. Try it

1. Open the UI and sign in as a volunteer (demo credentials are defined in the UI's user module).
2. Go to `/volunteer` and click **Start Analysis**; the CNS begins reading the configured camera.
3. Watch *Crowd & Safety Insights* and *Live Anomaly Alerts* update.
4. Sign in as an attendee and press **SOS**; the alert appears for volunteers/admins live.

---

## ⚙️ Configuration

| Variable | Used by | Purpose | Default |
|---|---|---|---|
| `GEMINI_API_KEY` | CNS, ADS | Gemini API access | — |
| `GOOGLE_MAPS_API_KEY` | CNS | Maps / places / traffic | — |
| `GEMINI_MODEL` | ADS | Model name | `gemini-2.0-flash` |
| `ANALYSIS_INTERVAL` | ADS | Seconds between frame analyses | `3` |
| `FRAME_QUALITY` | ADS | JPEG quality (1–100) | `85` |
| `HIGH_PRIORITY_THRESHOLD` | ADS | Severity treated as high priority | `3` |
| `CONFIDENCE_THRESHOLD` | ADS | Minimum confidence to raise an alert | `0.7` |
| `EVENT_TYPE`, `VENUE_CAPACITY`, `SECURITY_LEVEL` | ADS | Event context for prompts | `large_scale_event`, `10000`, `high` |
| `API_ENDPOINT` | ADS | Where incidents are forwarded | `http://localhost:3000/api/anomalies` |
| `NEXT_PUBLIC_API_URL` | UI | Backend base URL | `http://localhost:8000` |

**IP camera config:** provide a JSON file (e.g. `ip_camera_config.json`) or pass a config to `POST /api/cns/start-analysis`. `create_ip_camera_config_template()` in `live_crowd_predictor.py` generates a template.

---

## 🐳 Docker and Cloud Deployment

```bash
cp .env.example .env            # create it and set GEMINI_API_KEY
docker-compose up --build
```

`docker-compose.yml` starts the **backend** (:8000), **frontend** (:3000) and an **Nginx** reverse proxy (:80/:443). Health checks hit `/api/health` and `/`.

Provided deployment helpers: `deploy-gcp.sh/.ps1/.bat` (Cloud Run), `deploy-aws.sh`, `deploy-azure.sh`, `cloudbuild-backend.yaml`, `ui/cloudbuild.yaml`, and Terraform in `CNS/prediction/deployment/terraform`. See `DOCKER_DEPLOYMENT.md` and `GOOGLE_CLOUD_DEPLOYMENT.md`.

---

## 🛡️ Rate-Limit (HTTP 429) Mitigation

Running Gemini against several cameras continuously can exhaust quotas, so the CNS has layered protection (`api_mitigation_service.py`, `batch_api_service.py`, `local_preprocessing.py`; details in `CNS/prediction/RATE_LIMIT_MITIGATION.md`):

1. **Per-task intervals:** frame analysis ≥ 12 s per camera, flow analysis 3 min, bottleneck prediction 5 min.
2. **Burst limiter:** at most 8 API calls per 10-second window.
3. **Request batching and queueing** with priorities.
4. **Multi-level caching** of results.
5. **API-key rotation** with temporary blacklisting of throttled keys.
6. **Local fallback:** OpenCV people detection and rule-based predictions when the API is unavailable.

---

## ⚠️ Known Issues and Security Notes

These came up while reviewing the code and are worth fixing before any real deployment:

1. **🔴 Hardcoded API keys.** Several Google API keys are committed in plain text in `CNS/prediction/` (for example `live_crowd_predictor.py`, `map_crowd_analyzer.py`, `ai_situational_chat.py`, and files under `src/`). Treat them as compromised: **revoke/rotate them in Google Cloud Console**, load keys from environment variables only, and consider purging them from git history (`git filter-repo` or BFG).
2. **Missing frontend files.** The UI imports `@/lib/hooks/useDrishtiAPI` and `@/lib/users`, but `ui/src/lib/` is not in the repository because the root `.gitignore` contains `lib/`. The frontend will not build until those files are added and the ignore rule is narrowed (e.g. `/lib/`).
3. **Docker backend build.** `Dockerfile.backend` copies a root-level `requirements.txt`, which does not exist (requirements live in `CNS/prediction/`). It also runs uvicorn with `--reload` and a `fastapi_bridge` module that is not at the build root. Adjust the build context/paths.
4. **Demo authentication.** Login checks hardcoded dummy users and stores the session in `localStorage`; there is no real authentication. Firebase is a dependency but is not wired in.
5. **Open CORS.** The FastAPI bridge allows `"*"` origins, and the ADS apps use a static `SECRET_KEY` and `cors_allowed_origins="*"`.
6. **In-memory state.** SOS alerts, dispatches and insights are stored in Python dicts and are lost on restart; there is no database.
7. **Dispatch assignment is incomplete.** `/api/sos/create` broadcasts an SOS but does not create a dispatch record or auto-assign the nearest volunteer; `/api/dispatch/acknowledge` only works for dispatches that already exist. The ADS admin panel's volunteer list is mock data.
8. **Duplicated / experimental files.** `CNS/prediction` contains several iterations of the same Streamlit UI, one-off fix scripts (`fix_file.py`, `temp_fix.py`, `fix_syntax.py`), debug images and sample outputs, and `live_crowd_predictor.py` appears to contain duplicated method blocks. Worth cleaning up.
9. **Windows-specific paths.** Several scripts and docs reference a local Windows folder layout (`bhai ka kaam`, `nf2`) that doesn't match the repo.
10. **Privacy.** Analyzing camera feeds of people has legal and ethical implications. Deploy with signage, retention limits, and compliance with local privacy law. The system counts and classifies crowds and should not be used for identifying individuals.

---

## 🗺️ Roadmap

Natural next steps suggested by the current code:

- [ ] Persistent storage (PostgreSQL/Firestore) for incidents, SOS and dispatches
- [ ] Real authentication (Firebase Auth) and role-based access control
- [ ] Automatic nearest-volunteer assignment using volunteer GPS
- [ ] Merge ADS incidents into the FastAPI bridge so the UI shows one unified alert stream
- [ ] Replace Gemini frame-by-frame counting with an on-prem detector (YOLO etc.) to cut cost and latency
- [ ] Heatmap overlays of density on the venue map in the UI
- [ ] Automated tests and CI

---

## 🤝 Contributing

Issues and pull requests are welcome. Please never commit API keys; use environment variables or a secrets manager.

## 📄 License

No license file is currently included in the repository. Add one (e.g. MIT) to clarify usage terms.
