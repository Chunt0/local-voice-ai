# NurseVoice — Nursing Student Patient Simulation Tool

## Overview

A fully local AI voice simulation platform for nursing education. Students interact by voice with a simulated patient powered by a local LLM. The system converts student speech to text, processes it through an LLM conditioned on a patient profile, and speaks the response back using a local TTS engine. All inference runs on-device — no cloud APIs.

Built on the same LiveKit-based voice pipeline as the existing `local-voice-ai` project, with the LLM backend swapped to Ollama and an entirely new TypeScript/React web UI that supports patient profile management.

---

## Core User Flow

1. Instructor creates a patient profile (name, age, chief complaint, medical history, personality notes, system prompt).
2. Student opens the web UI, selects a patient from the list, and clicks **Start Session**.
3. Student speaks → STT (Nemotron/NeMo) → Ollama LLM (conditioned on patient profile) → Kokoro TTS → audio plays back.
4. Transcript is shown in real time. Session can be ended at any time.
5. Instructor can edit or delete patient profiles at any time.

---

## Technology Choices

| Concern | Technology | Notes |
|---|---|---|
| Frontend | React 19 + TypeScript | Vite for dev server/bundling |
| Backend API | Bun + TypeScript + Elysia | REST API for patient profiles & LiveKit tokens |
| Voice Agent | Python 3.12 + LiveKit Agents SDK | Most mature SDK for voice pipeline; Python is acceptable here |
| STT | NVIDIA Nemotron Speech 0.6B | Same as existing repo; FastAPI wrapper, OpenAI-compatible |
| LLM | Ollama (`gemma4:e2b`) | Replaces llama.cpp; OpenAI-compatible API |
| TTS | Kokoro (FastAPI) | Same as existing repo; lightweight, local |
| VAD | Silero VAD | Same as existing repo |
| WebRTC | LiveKit Server | Same as existing repo |
| Database | SQLite via Drizzle ORM | Embedded, zero-dependency persistence for patient profiles |
| Styling | Tailwind CSS v4 | |
| Containerization | Docker + Docker Compose | All services in one compose file |

---

## System Architecture

```
Browser (React UI)
  │
  ├── REST/HTTP ──────────────────→ API Server (Bun + Elysia)
  │                                    ├── GET/POST/PUT/DELETE /patients
  │                                    ├── POST /token  (LiveKit room token)
  │                                    └── SQLite DB (patient profiles)
  │
  └── WebRTC ─────────────────────→ LiveKit Server
                                         │
                                    LiveKit Agent (Python)
                                         ├── STT: Nemotron (HTTP)
                                         ├── LLM: Ollama (HTTP) ← patient profile injected as system prompt
                                         └── TTS: Kokoro (HTTP)
```

### Docker Services

| Service | Image / Build | Internal Port | Purpose |
|---|---|---|---|
| `livekit` | `livekit/livekit-server:latest` | 7880, 7881 | WebRTC signaling |
| `nemotron` | Build from `./inference/nemotron` | 8000 | STT (NeMo ASR) |
| `ollama` | `ollama/ollama:latest` | 11434 | LLM inference |
| `kokoro` | `ghcr.io/remsky/kokoro-fastapi-cpu:latest` | 8880 | TTS |
| `livekit_agent` | Build from `./livekit_agent` | — | Voice pipeline orchestrator |
| `api` | Build from `./api` | 4000 | Patient profile REST API + token endpoint |
| `frontend` | Build from `./frontend` | 3000 | React web UI |

All services are on a shared Docker bridge network `agent_network`. Only `frontend` (3000) and `api` (4000) are exposed to the host for normal use; LiveKit (7880) is exposed for the browser's WebRTC connection.

---

## Patient Profile Data Model

```typescript
// api/src/schema.ts
export interface PatientProfile {
  id: string;            // UUID
  name: string;          // e.g. "Maria Gonzalez"
  age: number;
  gender: string;        // "female" | "male" | "other" | free text
  chiefComplaint: string;     // e.g. "chest pain for 2 hours"
  medicalHistory: string;     // free text, markdown OK
  medications: string;        // free text list
  allergies: string;          // free text
  vitalSigns: string;         // free text, e.g. "BP 145/92, HR 88, RR 18, SpO2 97%"
  personalityNotes: string;   // e.g. "anxious, tends to minimize symptoms, asks many questions"
  systemPrompt: string;       // full system prompt — editable, auto-generated from fields above if blank
  createdAt: string;          // ISO timestamp
  updatedAt: string;
}
```

**Auto-generated system prompt template** (used if `systemPrompt` field is blank):

```
You are {{name}}, a {{age}}-year-old {{gender}} patient.
Chief complaint: {{chiefComplaint}}.
Medical history: {{medicalHistory}}.
Current medications: {{medications}}.
Allergies: {{allergies}}.
Current vitals: {{vitalSigns}}.

Personality: {{personalityNotes}}.

Respond only as this patient would. Stay in character. Answer questions about your symptoms honestly but with the vocabulary of a layperson. Do not volunteer diagnoses. Do not break character. Keep responses concise (1–4 sentences) unless the student asks a detailed question.
```

---

## API Server (Bun + Elysia)

### Endpoints

```
GET    /patients           → list all profiles (id, name, age, chiefComplaint)
POST   /patients           → create new profile
GET    /patients/:id       → get full profile
PUT    /patients/:id       → update profile
DELETE /patients/:id       → delete profile

POST   /token              → generate LiveKit room token
  body: { patientId: string, roomName?: string }
  returns: { token: string, url: string, patientProfile: PatientProfile }
```

### Token Generation

When the frontend requests a token it includes the selected `patientId`. The API server:
1. Fetches the patient profile from SQLite.
2. Creates a LiveKit room token with the patient's `id` embedded in room metadata.
3. Returns the token, the LiveKit server URL, and the full patient profile so the frontend can display patient info.

### Tech Stack

- **Bun** — runtime and package manager
- **Elysia** — fast, type-safe web framework built for Bun
- **Drizzle ORM** + `bun:sqlite` — type-safe SQLite queries
- **zod** — request validation
- **livekit-server-sdk** — token signing

### Directory Structure

```
api/
├── src/
│   ├── index.ts        # Elysia app entry point
│   ├── routes/
│   │   ├── patients.ts
│   │   └── token.ts
│   ├── db/
│   │   ├── schema.ts   # Drizzle schema
│   │   └── index.ts    # DB connection
│   └── lib/
│       └── promptBuilder.ts  # auto-generates system prompt from profile fields
├── drizzle/
│   └── migrations/
├── package.json        # Bun project
└── Dockerfile
```

---

## Voice Agent (Python)

Minimal changes from the existing agent. Key differences:

1. **Ollama instead of llama.cpp** — use `livekit-plugins-openai` pointed at Ollama's OpenAI-compatible endpoint, or the `livekit-plugins-ollama` plugin if available:
   ```python
   llm=openai.LLM(
       base_url="http://ollama:11434/v1",
       api_key="ollama",          # Ollama ignores this but the client requires it
       model="gemma4:e2b",
   )
   ```

2. **Dynamic system prompt from patient profile** — on session start, the agent reads room metadata to get the `patientId`, fetches the full profile from the API server (`http://api:4000/patients/{id}`), and injects the system prompt:
   ```python
   async def on_session_start(session: AgentSession, ctx: JobContext):
       patient_id = ctx.room.metadata  # set by frontend via room metadata
       profile = await fetch_patient_profile(patient_id)
       session.agent.instructions = profile["systemPrompt"]
   ```

3. **STT and TTS** remain the same (Nemotron, Kokoro).

### Directory Structure

```
livekit_agent/
├── src/
│   └── agent.py
├── pyproject.toml
└── Dockerfile
```

---

## Frontend (React + TypeScript + Vite)

### Pages / Views

```
/                    → Patient selection screen
/session/:patientId  → Active voice session with patient
/patients            → Instructor: list all patient profiles
/patients/new        → Instructor: create patient profile
/patients/:id/edit   → Instructor: edit patient profile
```

### Component Tree

```
App
├── PatientListPage
│   ├── PatientCard (name, age, chief complaint, select button)
│   └── NewPatientButton
│
├── PatientFormPage          (create / edit)
│   ├── BasicInfoFields      (name, age, gender)
│   ├── ClinicalFields       (chief complaint, history, meds, allergies, vitals)
│   ├── PersonalityField
│   └── SystemPromptEditor   (textarea, shows auto-generated or custom prompt)
│
└── SessionPage
    ├── PatientInfoPanel     (name, vitals, chief complaint — sidebar)
    ├── VoiceControls        (connect/disconnect, mic mute)
    ├── TranscriptPanel      (scrolling chat-style transcript)
    └── StatusBar            (connection state, STT/TTS activity)
```

### Key Libraries

- `@livekit/components-react` — voice/video UI components
- `livekit-client` — WebRTC client
- `@tanstack/react-query` — data fetching / cache for patient profiles
- `react-router-dom` v7 — client-side routing
- `tailwindcss` v4 — styling
- `@radix-ui/*` — accessible primitives (dialog, select, etc.)
- `zod` — shared schema validation

### Directory Structure

```
frontend/
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── pages/
│   │   ├── PatientList.tsx
│   │   ├── PatientForm.tsx
│   │   └── Session.tsx
│   ├── components/
│   │   ├── PatientCard.tsx
│   │   ├── SystemPromptEditor.tsx
│   │   ├── TranscriptPanel.tsx
│   │   └── VoiceControls.tsx
│   ├── hooks/
│   │   ├── usePatients.ts
│   │   └── useSession.ts
│   ├── lib/
│   │   ├── api.ts         # typed fetch wrapper for /api/*
│   │   └── promptBuilder.ts  # mirrors server-side auto-generator
│   └── types/
│       └── patient.ts
├── index.html
├── vite.config.ts
├── package.json           # Bun project
└── Dockerfile
```

---

## Ollama Service

Uses the official `ollama/ollama` Docker image. On first start, a startup script pulls `gemma4:e2b`:

```dockerfile
# inside docker-compose healthcheck / entrypoint
ollama serve &
sleep 5
ollama pull gemma4:e2b
wait
```

Or via a sidecar init container pattern in compose.

### Environment Variables (`.env`)

```env
OLLAMA_MODEL=gemma4:e2b
OLLAMA_BASE_URL=http://ollama:11434/v1
OLLAMA_HOST=0.0.0.0   # bind to all interfaces inside container
```

---

## Environment Configuration

### `.env` (root, loaded by all services via compose)

```env
# LiveKit
LIVEKIT_URL=http://livekit:7880
LIVEKIT_API_KEY=devkey
LIVEKIT_API_SECRET=secret
NEXT_PUBLIC_LIVEKIT_URL=http://localhost:7880

# Ollama / LLM
OLLAMA_MODEL=gemma4:e2b
OLLAMA_BASE_URL=http://ollama:11434/v1

# STT (Nemotron)
STT_BASE_URL=http://nemotron:8000/v1
STT_MODEL=nemotron-speech-streaming

# TTS (Kokoro)
KOKORO_BASE_URL=http://kokoro:8880/v1
KOKORO_VOICE=af_nova

# API Server
API_PORT=4000
DATABASE_PATH=/data/nursevoice.db

# Frontend (Vite)
VITE_API_URL=http://localhost:4000
VITE_LIVEKIT_URL=ws://localhost:7880
```

---

## Docker Compose Overview

```yaml
services:
  livekit:       # WebRTC signaling
  nemotron:      # STT — same as existing repo
  ollama:        # LLM — replaces llama_cpp
  kokoro:        # TTS — same as existing repo
  livekit_agent: # Python voice agent
    depends_on:
      nemotron: { condition: service_healthy }
      ollama:   { condition: service_healthy }
      kokoro:   { condition: service_started }
      livekit:  { condition: service_started }
  api:           # Bun REST API
    depends_on:  []  # no inference deps, starts immediately
  frontend:      # React UI (served by Vite in dev or nginx in prod)
    depends_on:
      api: { condition: service_started }
```

**Volumes**:
- `ollama-models:/root/.ollama` — persists pulled models across restarts
- `nemotron-cache:/root/.cache/huggingface` — same as existing
- `db-data:/data` — SQLite database

---

## GPU Support

Same pattern as existing repo: a `docker-compose.gpu.yml` overlay that adds:
- `runtime: nvidia` to `nemotron` and `ollama`
- `OLLAMA_NUM_GPU=1` (or appropriate env var for Ollama GPU layers)
- GPU-enabled Kokoro image

---

## Development Workflow

```bash
# Start everything
./compose-up.sh

# Frontend hot-reload (outside Docker)
cd frontend && bun dev

# API hot-reload (outside Docker)
cd api && bun --watch src/index.ts

# Agent hot-reload (outside Docker)
cd livekit_agent && uv run python src/agent.py dev
```

For local development outside Docker, service URLs are overridden via `.env.local` files in each subdirectory.

---

## Out of Scope (v1)

- User authentication / access control
- Session recording or playback
- Scoring / grading of student performance
- Multiple simultaneous sessions
- Custom voice selection per patient
- Multilingual support
