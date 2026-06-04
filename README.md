# AI Real-time GYM Trainer

A real-time AI-powered gym coaching application that uses computer vision to detect exercise form, count repetitions, and deliver proactive voice feedback through a large language model. Built with Streamlit and MediaPipe.

---

## Overview

The application streams live webcam video through a WebRTC connection, runs MediaPipe pose landmark detection on every frame, and pipes the resulting body joint data through exercise-specific detectors that measure angles and detect form errors. When errors are found — or when workout milestones are hit — a Groq-hosted LLM generates a terse coaching cue that is converted to speech via Google TTS and played back in the browser without interrupting the camera stream.

Workout history is persisted to a local SQLite database and displayed as an aggregated table after each session.

---

## Architecture

```
Browser (WebRTC)
      │
      ▼
VideoProcessorClass          ← runs in background thread
      │
      ├── MediaPipe PoseLandmarker (VIDEO mode)
      │         └── 33 body landmarks per frame
      │
      ├── Exercise Detector (Squats / Push-ups / Curls / Press / Lunges)
      │         └── angle calculation → rep counting → form metrics
      │
      └── _latest_metrics (thread-safe)
                │
                ▼
        sync_metrics_update()         ← called on every Streamlit rerun (250ms)
                │
                ├── session_state ← metrics, reps, sets
                ├── SQLite DB    ← persist on set completion
                └── VoicePipeline
                          ├── LLMCoach (Groq llama-3.3-70b-versatile)
                          └── TextToSpeech (gTTS → MP3 bytes → browser audio)
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Web framework | Streamlit 1.54 |
| Video streaming | streamlit-webrtc 0.64.5 (WebRTC) |
| Pose estimation | MediaPipe 0.10.14 — PoseLandmarker (full model) |
| Computer vision | OpenCV 4.10 (headless) |
| LLM coaching | Groq API — `llama-3.3-70b-versatile` |
| Text-to-speech | gTTS 2.5.3 (Google TTS) |
| Database | SQLite 3 (via Python stdlib) |
| Data display | Pandas 2.2.3 |

---

## Project Structure

```
MainApp/
├── main.py                              # Streamlit entry point & UI orchestrator
│
├── core/
│   └── base_exercise.py                 # Abstract base class — angle math & detector contract
│
├── detectors/
│   ├── squat.py                         # Squat form detector
│   ├── pushup.py                        # Push-up form detector
│   ├── biceps_curl.py                   # Biceps curl form detector
│   ├── shoulder_press.py                # Shoulder press form detector
│   └── lunges.py                        # Lunge form detector
│
├── ml_models/
│   ├── pose_landmarker_full.task        # MediaPipe full pose model (used in production)
│   └── pose_landmarker_lite.task        # MediaPipe lite pose model (faster, less accurate)
│
├── services/
│   ├── auth/
│   │   └── login_wall.py                # Username-based session auth
│   ├── coaching/
│   │   ├── llm.py                       # Groq LLM client wrapper
│   │   ├── tts.py                       # gTTS wrapper → MP3 bytes
│   │   └── voice_pipeline.py            # Form issue detection + LLM + TTS orchestration
│   ├── config/
│   │   └── workout_config.py            # Exercise options, pose connections, LLM system prompt
│   ├── persistence/
│   │   └── exercise_repository.py       # SQLite CRUD — users and exercise history
│   ├── state/
│   │   └── session_defaults.py          # Streamlit session_state initialization
│   ├── tracking/
│   │   └── metrics.py                   # Video processor → session_state sync + voice triggers
│   ├── ui/
│   │   └── style_loader.py              # CSS injection, font loading, WebRTC iframe patching
│   └── vision/
│       └── exercise_video_processor.py  # VideoProcessorBase subclass — frame pipeline
│
├── static/
│   ├── style.css                        # Dark theme UI overrides
│   └── AdobeClean.otf                   # Custom display font
│
├── data.db                              # SQLite database (auto-created on first run)
├── requirements.txt
└── .streamlit/
    ├── config.toml                      # Theme configuration
    └── secrets.toml                     # GROQ_API_KEY (not committed)
```

---

## Setup & Installation

### Prerequisites

- Python 3.12+
- A [Groq API key](https://console.groq.com/) (free tier available)
- Webcam

### 1. Clone and create virtual environment

```bash
git clone https://github.com/AzhadArshad/ai-realtime-gymtrainer.git
cd ai-realtime-gymtrainer/MainApp
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Configure secrets

Create `.streamlit/secrets.toml`:

```toml
GROQ_API_KEY = "your_groq_api_key_here"
```

Or set as an environment variable:

```bash
export GROQ_API_KEY="your_groq_api_key_here"
```

### 4. Run

```bash
streamlit run main.py
```

Open `http://localhost:8501` in your browser.

---

## How It Works

### 1. Authentication

On first visit, the user enters a unique username. The app calls `get_or_create_user()` which either retrieves the existing SQLite user record or creates a new one. The integer `user_id` and `username` are stored in `st.session_state` for the session.

### 2. Workout Configuration

Before starting, the user selects an exercise, target sets, and reps per set from the sidebar. When "Start Workout" is clicked, these values are committed to `session_state` and the WebRTC camera activates.

### 3. Pose Detection Pipeline

Each incoming video frame goes through `VideoProcessorClass.recv()`:

```
Frame (BGR) → horizontal flip (mirror) → MediaPipe Image (RGB)
    → PoseLandmarker.detect_for_video(frame, timestamp_ms)
    → 33 landmarks with (x, y, z, visibility) per landmark
    → ExerciseDetector.process(landmarks)
    → metrics dict
    → overlay rendering
    → return processed frame
```

The frame timestamp increments by 30ms per frame to satisfy MediaPipe's VIDEO mode requirement (which needs monotonically increasing timestamps).

### 4. Angle Calculation

All detectors inherit `calculate_angle(a, b, c)` from `BaseExercise`. It computes the interior angle at joint `b` using the dot product formula:

```
vector_BA = a - b
vector_BC = c - b

cos(θ) = (BA · BC) / (|BA| × |BC|)
θ = arccos(cos(θ)) in degrees
```

The result is clamped to `[-1.0, 1.0]` before `arccos` to guard against floating-point overflow.

### 5. Rep Counting (State Machine)

Every detector uses a two-stage state machine:

```
Stage = None
    ↓  angle crosses DOWN_THRESHOLD
Stage = "down"
    ↓  angle crosses UP_THRESHOLD (while stage == "down")
Stage = "up"  →  reps += 1
```

The `stage == "down"` guard prevents counting a rep on every frame where the joint is extended — only a complete down-then-up cycle registers.

### 6. Visibility-Aware Side Selection

When both sides of the body are tracked (e.g., both knees), detectors compare MediaPipe visibility scores and use the side with higher confidence. This handles cases where the user is not facing the camera directly.

### 7. Metrics Synchronization

`sync_metrics_update()` is called on every Streamlit rerun (every ~250ms). It:

1. Reads `_latest_metrics` from the video processor via a thread lock
2. Writes all metric values to `session_state` (triggers sidebar refresh)
3. Calculates `sets_completed = total_reps // reps_per_set`
4. On set completion: saves to SQLite, triggers `set_completed` voice event
5. On workout completion: sets `workout_started = False`, triggers `workout_completed` voice event
6. On no pose: triggers `no_pose_detected` voice event
7. On every tick: triggers `ongoing_form_check` (throttled to once per 5 seconds if form issue exists)

### 8. Voice Coaching Pipeline

```
sync_metrics_update detects event
    → VoicePipeline.process_event(event, exercise, metrics)
    → _find_form_issue() checks metric thresholds → returns issue string or None
    → LLMCoach.give_feedback(event, issue)
        → constructs prompt: "Event: {event} Form Issue: {issue}"
        → sends to Groq (llama-3.3-70b-versatile, temperature=0.4)
        → maintains last 10 messages of conversation history
        → returns ≤12-word imperative coaching cue
    → TextToSpeech.speak(text)
        → gTTS encodes to MP3 bytes
    → audio_to_play stored in session_state
    → autoplay_audio() injects <audio> element directly into parent document.body
       (survives Streamlit reruns — not destroyed by React reconciliation)
```

**Throttling:** Non-major events (`ongoing_form_check`, `no_pose_detected`) are silenced if fewer than 5 seconds have elapsed since the last spoken cue. Major events (`workout_started`, `set_completed`, `workout_completed`) always trigger feedback immediately.

### 9. Database Schema

```sql
CREATE TABLE users (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    username   TEXT UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE exercises (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id       INTEGER NOT NULL REFERENCES users(id),
    exercise_name TEXT    NOT NULL,
    reps          INTEGER NOT NULL DEFAULT 0,
    sets          INTEGER NOT NULL DEFAULT 0,
    time          INTEGER NOT NULL DEFAULT 0,
    created_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

`add_exercise()` upserts — if a row already exists for the same user, exercise, and calendar day, it increments the existing reps/sets/time rather than inserting a duplicate.

---

## Supported Exercises & Form Checks

| Exercise | Rep Trigger | Form Checks |
|---|---|---|
| **Squats** | Knee angle: down < 100°, up > 160° | Squat depth, forward lean (back angle < 130°) |
| **Push-ups** | Elbow angle: down < 90°, up > 160° | Body alignment, hip sag, hip pike |
| **Biceps Curls** | Elbow angle: up < 50°, down > 160° | Elbow drift from centerline, torso swing |
| **Shoulder Press** | Elbow angle: down < 90°, up > 160° | Arm extension phase, lower back arch |
| **Lunges** | Front knee: down < 100°, up > 160° | Lateral balance (shoulder-hip alignment) |

---

## Session State Reference

| Key | Type | Description |
|---|---|---|
| `user_id` | int | SQLite user PK |
| `username` | str | Display name |
| `workout_started` | bool | Camera/tracking active |
| `exercise_type` | str | Current exercise name |
| `target_sets` | int | Sets planned |
| `reps_per_set` | int | Reps per set planned |
| `reps` | int | Total reps counted |
| `sets_completed` | int | Completed sets |
| `current_set_reps` | int | Reps in current set |
| `voice_pipeline` | VoicePipeline | Persistent coaching instance |
| `audio_to_play` | bytes | MP3 bytes pending playback |
| `coach_feedback` | str | Last coaching cue text |

---

## Known Limitations

- **Session persistence:** `session_state` is in-memory only. Refreshing the page logs the user out (by design — re-entering the username reconnects to the same SQLite record).
- **Single camera:** Only front-facing pose analysis is supported. Side-on angles are not validated.
- **Lighting sensitivity:** MediaPipe confidence drops significantly in low light. Keep the area well-lit.
- **Network dependency:** Voice coaching requires internet access for both the Groq API and gTTS.
- **WebRTC STUN:** Uses Google's public STUN server (`stun.l.google.com:19302`). For deployment behind a strict firewall, a TURN server may be required.

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `GROQ_API_KEY` | Yes | Groq API key for LLM coaching |

---

## Built With

- [Streamlit](https://streamlit.io/)
- [MediaPipe](https://ai.google.dev/edge/mediapipe/solutions/vision/pose_landmarker)
- [streamlit-webrtc](https://github.com/whitphx/streamlit-webrtc)
- [Groq](https://groq.com/)
- [gTTS](https://gtts.readthedocs.io/)

---

## Author

**Azhad Arshad**

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=flat&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/azhad-arshad-594b64323/)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat&logo=github&logoColor=white)](https://github.com/AzhadArshad)
[![Hugging Face](https://img.shields.io/badge/HuggingFace-FFD21E?style=flat&logo=huggingface&logoColor=black)](https://huggingface.co/Aju360)
[![Kaggle](https://img.shields.io/badge/Kaggle-20BEFF?style=flat&logo=kaggle&logoColor=white)](https://www.kaggle.com/azhadarshad)
