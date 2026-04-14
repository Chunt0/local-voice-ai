# NurseVoice — Nursing Student Patient Simulation Tool

## Overview

A fully local AI voice simulation platform for nursing education. Students interact by voice with a simulated patient powered by a local LLM. Speech is transcribed via NeMo ASR (Nemotron), processed by Ollama (`gemma4:e2b`) conditioned on a patient profile, and spoken back via Kokoro TTS. All inference runs on-device — no cloud APIs, no external services.

Interactions are automatically saved as transcripts tied to the patient profile and session timestamp, so instructors can review student performance after the fact.

---

## Repository Structure

Bun workspaces monorepo. All TypeScript packages share a root `package.json`.

```
nursevoice/
├── packages/
│   ├── api/                  # Elysia REST API + SQLite
│   └── frontend/             # React 19 + Vite
├── services/
│   ├── livekit_agent/        # Python voice pipeline (LiveKit Agents SDK)
│   └── nemotron/             # Python STT service (NeMo ASR, FastAPI)
├── docker-compose.yml
├── docker-compose.gpu.yml
├── compose-up.sh
├── .env
└── package.json              # bun workspaces root
```

Root `package.json`:
```json
{
  "name": "nursevoice",
  "workspaces": ["packages/api", "packages/frontend"],
  "private": true
}
```

---

## Technology Choices

| Concern | Technology | Version |
|---|---|---|
| Frontend runtime | Bun | 1.2.x |
| Frontend framework | React | 19.1.x |
| Frontend build | Vite | 6.x |
| Frontend routing | React Router | 7.x |
| Frontend data fetching | TanStack Query | 5.x |
| Frontend styling | Tailwind CSS | 4.x |
| Frontend UI primitives | Radix UI | latest |
| Frontend LiveKit | @livekit/components-react | 2.9.x |
| Frontend LiveKit | livekit-client | 2.15.x |
| API runtime | Bun | 1.2.x |
| API framework | Elysia | 1.3.x |
| API typed client | @elysiajs/eden (Eden Treaty) | 1.3.x |
| Database ORM | drizzle-orm | 0.44.x |
| Database driver | bun:sqlite (built-in) | — |
| Schema validation | Elysia `t` (TypeBox) | bundled with Elysia |
| Voice agent | Python | 3.12 |
| Voice agent framework | livekit-agents | 1.3.x |
| STT | NeMo Nemotron Speech 0.6B | nvidia/nemotron-speech-streaming-en-0.6b |
| LLM | Ollama | latest |
| LLM model | gemma4:e2b | via `ollama pull` |
| TTS | Kokoro FastAPI | ghcr.io/remsky/kokoro-fastapi-cpu:latest |
| VAD | Silero VAD | bundled with livekit-agents |
| WebRTC | LiveKit Server | latest |
| Token signing | livekit-server-sdk | 2.x |
| Containerization | Docker + Docker Compose | — |

---

## System Architecture

```
Browser (React UI @ :3000)
  │
  ├── HTTP ──────────────────────────► Elysia API (Bun @ :4000)
  │    GET/POST/PUT/DELETE /patients       │
  │    GET/POST /sessions                  ├── bun:sqlite  (nursevoice.db)
  │    GET /sessions/:id                   └── livekit-server-sdk (token signing)
  │    POST /token
  │    Eden Treaty typed client
  │
  └── WebRTC ────────────────────────► LiveKit Server (:7880)
                                            │
                                       LiveKit Agent (Python)
                                            ├── on join: GET /patients/:id from API
                                            ├── on join: POST /sessions to API
                                            ├── STT events: POST /sessions/:id/entries to API
                                            ├── LLM outputs: POST /sessions/:id/entries to API
                                            ├── on disconnect: PATCH /sessions/:id to API
                                            │
                                            ├── STT ──► Nemotron FastAPI (:8000)
                                            ├── LLM ──► Ollama (:11434)
                                            └── TTS ──► Kokoro FastAPI (:8880)
```

---

## Database Schema

File: `packages/api/src/db/schema.ts`

Uses Drizzle ORM with `bun:sqlite`. Database file is mounted from the host at the path set in `DATABASE_PATH` env var (default `/data/nursevoice.db`).

```typescript
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core'
import { sql } from 'drizzle-orm'

export const patients = sqliteTable('patients', {
  id:               text('id').primaryKey(),                      // UUID v4
  name:             text('name').notNull(),
  age:              integer('age').notNull(),
  gender:           text('gender').notNull(),                     // free text
  chiefComplaint:   text('chief_complaint').notNull(),
  medicalHistory:   text('medical_history').notNull().default(''),
  medications:      text('medications').notNull().default(''),    // newline-separated
  allergies:        text('allergies').notNull().default(''),
  vitalSigns:       text('vital_signs').notNull().default(''),    // free text
  personalityNotes: text('personality_notes').notNull().default(''),
  systemPrompt:     text('system_prompt').notNull().default(''),  // '' = auto-generate at runtime
  createdAt:        text('created_at').notNull().default(sql`(strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))`),
  updatedAt:        text('updated_at').notNull().default(sql`(strftime('%Y-%m-%dT%H:%M:%fZ', 'now'))`),
})

export const sessions = sqliteTable('sessions', {
  id:              text('id').primaryKey(),                       // UUID v4
  patientId:       text('patient_id').notNull().references(() => patients.id, { onDelete: 'cascade' }),
  startedAt:       text('started_at').notNull(),                  // ISO 8601
  endedAt:         text('ended_at'),                              // null while active
  durationSeconds: integer('duration_seconds'),                   // null while active
})

export const transcriptEntries = sqliteTable('transcript_entries', {
  id:        text('id').primaryKey(),                             // UUID v4
  sessionId: text('session_id').notNull().references(() => sessions.id, { onDelete: 'cascade' }),
  role:      text('role', { enum: ['student', 'patient'] }).notNull(),
  text:      text('text').notNull(),
  timestamp: text('timestamp').notNull(),                         // ISO 8601
})
```

Migration is run at startup via `drizzle-kit push` (development) or SQL migration files (production). The API server calls `db.run(migrate)` on boot before accepting requests.

File: `packages/api/src/db/index.ts`

```typescript
import { Database } from 'bun:sqlite'
import { drizzle } from 'drizzle-orm/bun-sqlite'
import * as schema from './schema'

const sqlite = new Database(process.env.DATABASE_PATH ?? '/data/nursevoice.db')
sqlite.run('PRAGMA journal_mode = WAL')
sqlite.run('PRAGMA foreign_keys = ON')

export const db = drizzle(sqlite, { schema })
export type DB = typeof db
```

---

## API Server

### Package Setup

`packages/api/package.json`:
```json
{
  "name": "@nursevoice/api",
  "module": "src/index.ts",
  "type": "module",
  "scripts": {
    "dev": "bun --watch src/index.ts",
    "start": "bun src/index.ts",
    "db:push": "drizzle-kit push",
    "db:studio": "drizzle-kit studio"
  },
  "dependencies": {
    "elysia": "^1.3.0",
    "@elysiajs/cors": "^1.3.0",
    "@elysiajs/swagger": "^1.3.0",
    "drizzle-orm": "^0.44.0",
    "livekit-server-sdk": "^2.0.0"
  },
  "devDependencies": {
    "drizzle-kit": "^0.31.0",
    "@types/bun": "latest",
    "typescript": "^5.8.0"
  }
}
```

### Entry Point

`packages/api/src/index.ts`:
```typescript
import { Elysia } from 'elysia'
import { cors } from '@elysiajs/cors'
import { swagger } from '@elysiajs/swagger'
import { migrate } from './db/migrate'
import { patientsRoutes } from './routes/patients'
import { sessionsRoutes } from './routes/sessions'
import { tokenRoutes } from './routes/token'

await migrate()  // run migrations before accepting traffic

const app = new Elysia()
  .use(cors({ origin: process.env.FRONTEND_ORIGIN ?? 'http://localhost:3000' }))
  .use(swagger({ path: '/docs' }))
  .use(patientsRoutes)
  .use(sessionsRoutes)
  .use(tokenRoutes)
  .listen(Number(process.env.API_PORT ?? 4000))

console.log(`API running on http://localhost:${app.server?.port}`)

export type App = typeof app
```

The `App` type export is what the frontend imports for Eden Treaty.

### Patient Routes

`packages/api/src/routes/patients.ts`:

```typescript
import { Elysia, t } from 'elysia'
import { db } from '../db'
import { patients } from '../db/schema'
import { eq } from 'drizzle-orm'
import { buildSystemPrompt } from '../lib/promptBuilder'
import { randomUUID } from 'crypto'

const PatientBody = t.Object({
  name:             t.String({ minLength: 1 }),
  age:              t.Number({ minimum: 0, maximum: 120 }),
  gender:           t.String({ minLength: 1 }),
  chiefComplaint:   t.String({ minLength: 1 }),
  medicalHistory:   t.String(),
  medications:      t.String(),
  allergies:        t.String(),
  vitalSigns:       t.String(),
  personalityNotes: t.String(),
  systemPrompt:     t.String(),  // empty string = auto-generate
})

export const patientsRoutes = new Elysia({ prefix: '/patients' })
  // List all patients (summary only — no systemPrompt in list response)
  .get('/', () =>
    db.select({
      id: patients.id,
      name: patients.name,
      age: patients.age,
      gender: patients.gender,
      chiefComplaint: patients.chiefComplaint,
      createdAt: patients.createdAt,
    }).from(patients).orderBy(patients.createdAt)
  )

  // Get single patient (full record, systemPrompt resolved)
  .get('/:id', ({ params, error }) => {
    const [patient] = db.select().from(patients).where(eq(patients.id, params.id))
    if (!patient) return error(404, { message: 'Patient not found' })
    return {
      ...patient,
      systemPrompt: patient.systemPrompt || buildSystemPrompt(patient),
    }
  }, { params: t.Object({ id: t.String() }) })

  // Create patient
  .post('/', ({ body }) => {
    const now = new Date().toISOString()
    const id = randomUUID()
    db.insert(patients).values({ ...body, id, createdAt: now, updatedAt: now }).run()
    const [created] = db.select().from(patients).where(eq(patients.id, id))
    return created
  }, { body: PatientBody })

  // Update patient
  .put('/:id', ({ params, body, error }) => {
    const [existing] = db.select().from(patients).where(eq(patients.id, params.id))
    if (!existing) return error(404, { message: 'Patient not found' })
    db.update(patients)
      .set({ ...body, updatedAt: new Date().toISOString() })
      .where(eq(patients.id, params.id))
      .run()
    const [updated] = db.select().from(patients).where(eq(patients.id, params.id))
    return updated
  }, {
    params: t.Object({ id: t.String() }),
    body: PatientBody,
  })

  // Delete patient (cascades to sessions + transcriptEntries via FK)
  .delete('/:id', ({ params, error }) => {
    const [existing] = db.select().from(patients).where(eq(patients.id, params.id))
    if (!existing) return error(404, { message: 'Patient not found' })
    db.delete(patients).where(eq(patients.id, params.id)).run()
    return { deleted: true }
  }, { params: t.Object({ id: t.String() }) })
```

### Session Routes

`packages/api/src/routes/sessions.ts`:

```typescript
import { Elysia, t } from 'elysia'
import { db } from '../db'
import { sessions, transcriptEntries, patients } from '../db/schema'
import { eq, desc } from 'drizzle-orm'
import { randomUUID } from 'crypto'

export const sessionsRoutes = new Elysia({ prefix: '/sessions' })
  // List all sessions with patient name — for the instructor review page
  .get('/', () =>
    db.select({
      id: sessions.id,
      patientId: sessions.patientId,
      patientName: patients.name,
      startedAt: sessions.startedAt,
      endedAt: sessions.endedAt,
      durationSeconds: sessions.durationSeconds,
    })
    .from(sessions)
    .leftJoin(patients, eq(sessions.patientId, patients.id))
    .orderBy(desc(sessions.startedAt))
  )

  // Get single session with full transcript
  .get('/:id', ({ params, error }) => {
    const [session] = db.select().from(sessions).where(eq(sessions.id, params.id))
    if (!session) return error(404, { message: 'Session not found' })

    const [patient] = db.select({ name: patients.name, chiefComplaint: patients.chiefComplaint })
      .from(patients).where(eq(patients.id, session.patientId))

    const entries = db.select()
      .from(transcriptEntries)
      .where(eq(transcriptEntries.sessionId, params.id))
      .orderBy(transcriptEntries.timestamp)

    return { ...session, patient, entries }
  }, { params: t.Object({ id: t.String() }) })

  // Create session (called by agent on room join)
  .post('/', ({ body }) => {
    const id = randomUUID()
    db.insert(sessions).values({ id, ...body }).run()
    return { id }
  }, {
    body: t.Object({
      patientId: t.String(),
      startedAt: t.String(),  // ISO 8601
    }),
  })

  // Close session (called by agent on participant disconnect)
  .patch('/:id', ({ params, body, error }) => {
    const [session] = db.select().from(sessions).where(eq(sessions.id, params.id))
    if (!session) return error(404, { message: 'Session not found' })
    db.update(sessions)
      .set({ endedAt: body.endedAt, durationSeconds: body.durationSeconds })
      .where(eq(sessions.id, params.id))
      .run()
    return { updated: true }
  }, {
    params: t.Object({ id: t.String() }),
    body: t.Object({
      endedAt: t.String(),
      durationSeconds: t.Number(),
    }),
  })

  // Delete session
  .delete('/:id', ({ params, error }) => {
    const [existing] = db.select().from(sessions).where(eq(sessions.id, params.id))
    if (!existing) return error(404, { message: 'Session not found' })
    db.delete(sessions).where(eq(sessions.id, params.id)).run()
    return { deleted: true }
  }, { params: t.Object({ id: t.String() }) })

  // Add transcript entry (called by agent in real time)
  .post('/:id/entries', ({ params, body, error }) => {
    const [session] = db.select().from(sessions).where(eq(sessions.id, params.id))
    if (!session) return error(404, { message: 'Session not found' })
    const id = randomUUID()
    db.insert(transcriptEntries).values({ id, sessionId: params.id, ...body }).run()
    return { id }
  }, {
    params: t.Object({ id: t.String() }),
    body: t.Object({
      role: t.Union([t.Literal('student'), t.Literal('patient')]),
      text: t.String({ minLength: 1 }),
      timestamp: t.String(),  // ISO 8601
    }),
  })
```

### Token Route

`packages/api/src/routes/token.ts`:

```typescript
import { Elysia, t } from 'elysia'
import { AccessToken } from 'livekit-server-sdk'
import { db } from '../db'
import { patients } from '../db/schema'
import { eq } from 'drizzle-orm'
import { buildSystemPrompt } from '../lib/promptBuilder'

export const tokenRoutes = new Elysia()
  .post('/token', async ({ body, error }) => {
    const [patient] = db.select().from(patients).where(eq(patients.id, body.patientId))
    if (!patient) return error(404, { message: 'Patient not found' })

    const resolvedPrompt = patient.systemPrompt || buildSystemPrompt(patient)
    const roomName = body.roomName ?? `session-${Date.now()}`

    const token = new AccessToken(
      process.env.LIVEKIT_API_KEY!,
      process.env.LIVEKIT_API_SECRET!,
      { identity: `student-${Date.now()}`, name: 'Student' }
    )
    token.addGrant({ roomJoin: true, room: roomName, canPublish: true, canSubscribe: true })

    // Room metadata is a JSON string read by the agent on join
    // The agent uses patientId to fetch the full profile (including resolved system prompt)
    const metadata = JSON.stringify({ patientId: patient.id })

    return {
      token: await token.toJwt(),
      url: process.env.LIVEKIT_URL_PUBLIC ?? 'ws://localhost:7880',
      roomName,
      roomMetadata: metadata,
      patient: {
        id: patient.id,
        name: patient.name,
        age: patient.age,
        gender: patient.gender,
        chiefComplaint: patient.chiefComplaint,
        vitalSigns: patient.vitalSigns,
        systemPrompt: resolvedPrompt,
      },
    }
  }, {
    body: t.Object({
      patientId: t.String(),
      roomName: t.Optional(t.String()),
    }),
  })
```

Note: the room metadata is set in the token endpoint response, not on the LiveKit room directly. The frontend must pass `roomMetadata` when creating the room via the LiveKit client.

### Prompt Builder

`packages/api/src/lib/promptBuilder.ts`:

```typescript
type PatientFields = {
  name: string
  age: number
  gender: string
  chiefComplaint: string
  medicalHistory: string
  medications: string
  allergies: string
  vitalSigns: string
  personalityNotes: string
}

export function buildSystemPrompt(p: PatientFields): string {
  return `You are ${p.name}, a ${p.age}-year-old ${p.gender} patient presenting to a hospital.

Chief complaint: ${p.chiefComplaint}.
Medical history: ${p.medicalHistory || 'None reported'}.
Current medications: ${p.medications || 'None'}.
Allergies: ${p.allergies || 'NKDA'}.
Current vitals: ${p.vitalSigns || 'Not yet assessed'}.

Personality and demeanor: ${p.personalityNotes || 'Cooperative and calm'}.

Rules:
- Respond ONLY as this patient would, in first person.
- Use lay vocabulary — never use medical jargon unless the patient would know it.
- Do not volunteer a diagnosis or suggest treatments.
- Stay fully in character at all times; do not acknowledge being an AI.
- Keep responses to 1–4 sentences unless the student asks a detailed question.
- If asked something the patient wouldn't know, say so naturally (e.g. "I'm not sure, the doctor mentioned something about that").`
}
```

### Directory Structure

```
packages/api/
├── src/
│   ├── index.ts
│   ├── routes/
│   │   ├── patients.ts
│   │   ├── sessions.ts
│   │   └── token.ts
│   ├── db/
│   │   ├── schema.ts
│   │   ├── index.ts
│   │   └── migrate.ts        # runs drizzle migrations on startup
│   └── lib/
│       └── promptBuilder.ts
├── drizzle/
│   └── migrations/           # generated by drizzle-kit
├── drizzle.config.ts
├── package.json
├── tsconfig.json
└── Dockerfile
```

`packages/api/Dockerfile`:
```dockerfile
FROM oven/bun:1.2-alpine
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile
COPY . .
RUN bun run db:push
EXPOSE 4000
CMD ["bun", "src/index.ts"]
```

---

## Voice Agent (Python)

### Package Setup

`services/livekit_agent/pyproject.toml`:
```toml
[project]
name = "nursevoice-agent"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
  "livekit-agents[silero,turn-detector,openai]~=1.3",
  "livekit-plugins-noise-cancellation~=0.2",
  "aiohttp>=3.10",
  "python-dotenv>=1.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

### Agent Implementation

`services/livekit_agent/src/agent.py`:

```python
import asyncio
import json
import logging
import os
from datetime import datetime, timezone
from uuid import uuid4

import aiohttp
from dotenv import load_dotenv
from livekit.agents import (
    Agent,
    AgentSession,
    JobContext,
    RoomInputOptions,
    WorkerOptions,
    cli,
    metrics,
)
from livekit.agents.llm import ChatMessage
from livekit.plugins import noise_cancellation, openai, silero

load_dotenv()
logger = logging.getLogger("nursevoice.agent")

API_BASE = os.environ.get("API_BASE_URL", "http://api:4000")
STT_BASE = os.environ.get("STT_BASE_URL", "http://nemotron:8000/v1")
STT_MODEL = os.environ.get("STT_MODEL", "nemotron-speech-streaming")
OLLAMA_BASE = os.environ.get("OLLAMA_BASE_URL", "http://ollama:11434/v1")
OLLAMA_MODEL = os.environ.get("OLLAMA_MODEL", "gemma4:e2b")
KOKORO_BASE = os.environ.get("KOKORO_BASE_URL", "http://kokoro:8880/v1")
KOKORO_VOICE = os.environ.get("KOKORO_VOICE", "af_nova")


async def fetch_patient(patient_id: str) -> dict:
    async with aiohttp.ClientSession() as http:
        async with http.get(f"{API_BASE}/patients/{patient_id}") as resp:
            resp.raise_for_status()
            return await resp.json()


async def create_session(patient_id: str) -> str:
    async with aiohttp.ClientSession() as http:
        body = {"patientId": patient_id, "startedAt": datetime.now(timezone.utc).isoformat()}
        async with http.post(f"{API_BASE}/sessions", json=body) as resp:
            resp.raise_for_status()
            data = await resp.json()
            return data["id"]


async def close_session(session_id: str, started_at: datetime) -> None:
    ended_at = datetime.now(timezone.utc)
    duration = int((ended_at - started_at).total_seconds())
    async with aiohttp.ClientSession() as http:
        body = {"endedAt": ended_at.isoformat(), "durationSeconds": duration}
        await http.patch(f"{API_BASE}/sessions/{session_id}", json=body)


async def save_entry(session_id: str, role: str, text: str) -> None:
    async with aiohttp.ClientSession() as http:
        body = {"role": role, "text": text, "timestamp": datetime.now(timezone.utc).isoformat()}
        await http.post(f"{API_BASE}/sessions/{session_id}/entries", json=body)


class PatientAgent(Agent):
    def __init__(self, instructions: str):
        super().__init__(instructions=instructions)


async def entrypoint(ctx: JobContext) -> None:
    await ctx.connect()

    # Parse room metadata set by the frontend token endpoint
    metadata = json.loads(ctx.room.metadata or "{}")
    patient_id = metadata.get("patientId")
    if not patient_id:
        logger.error("No patientId in room metadata — cannot start session")
        return

    # Fetch patient profile (resolved system prompt included)
    patient = await fetch_patient(patient_id)
    system_prompt = patient["systemPrompt"]
    logger.info("Starting session as patient: %s", patient["name"])

    # Create session record in DB
    session_id = await create_session(patient_id)
    session_start = datetime.now(timezone.utc)

    agent = PatientAgent(instructions=system_prompt)

    session = AgentSession(
        vad=silero.VAD.load(),
        stt=openai.STT(
            base_url=STT_BASE,
            api_key="no-key",
            model=STT_MODEL,
        ),
        llm=openai.LLM(
            base_url=OLLAMA_BASE,
            api_key="ollama",
            model=OLLAMA_MODEL,
        ),
        tts=openai.TTS(
            base_url=KOKORO_BASE,
            api_key="no-key",
            model="kokoro",
            voice=KOKORO_VOICE,
        ),
        turn_detection=openai.realtime.RealtimeTurnDetection(
            type="server_vad",
        ) if False else None,  # use silero VAD only
    )

    # Hook into transcript events to persist to API
    @session.on("user_speech_committed")
    def on_user_speech(msg: ChatMessage):
        text = msg.content if isinstance(msg.content, str) else ""
        if text:
            asyncio.create_task(save_entry(session_id, "student", text))

    @session.on("agent_speech_committed")
    def on_agent_speech(msg: ChatMessage):
        text = msg.content if isinstance(msg.content, str) else ""
        if text:
            asyncio.create_task(save_entry(session_id, "patient", text))

    await session.start(
        agent=agent,
        room=ctx.room,
        room_input_options=RoomInputOptions(
            noise_cancellation=noise_cancellation.BVC(),
        ),
    )

    # Wait for the participant to disconnect
    await ctx.wait_for_disconnect()
    await close_session(session_id, session_start)
    logger.info("Session %s closed", session_id)


if __name__ == "__main__":
    cli.run_app(
        WorkerOptions(
            entrypoint_fnc=entrypoint,
            prewarm_fnc=lambda proc: silero.VAD.load(),
        )
    )
```

### Directory Structure

```
services/livekit_agent/
├── src/
│   └── agent.py
├── pyproject.toml
├── .python-version        # 3.12
└── Dockerfile
```

`services/livekit_agent/Dockerfile`:
```dockerfile
FROM python:3.12-slim-bookworm
WORKDIR /app
RUN pip install uv
COPY pyproject.toml .
RUN uv pip install --system .
COPY src/ src/
HEALTHCHECK --interval=5s --timeout=3s --retries=5 \
  CMD curl -f http://localhost:8081/ || exit 1
CMD ["python", "src/agent.py", "start"]
```

---

## Frontend

### Package Setup

`packages/frontend/package.json`:
```json
{
  "name": "@nursevoice/frontend",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "@elysiajs/eden": "^1.3.0",
    "@livekit/components-react": "^2.9.0",
    "@radix-ui/react-dialog": "^1.1.0",
    "@radix-ui/react-select": "^2.1.0",
    "@radix-ui/react-label": "^2.1.0",
    "@radix-ui/react-separator": "^1.1.0",
    "@tanstack/react-query": "^5.0.0",
    "livekit-client": "^2.15.0",
    "react": "^19.1.0",
    "react-dom": "^19.1.0",
    "react-router-dom": "^7.0.0",
    "elysia": "^1.3.0"
  },
  "devDependencies": {
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "@vitejs/plugin-react": "^4.0.0",
    "tailwindcss": "^4.0.0",
    "@tailwindcss/vite": "^4.0.0",
    "typescript": "^5.8.0",
    "vite": "^6.0.0"
  }
}
```

### Eden Treaty API Client

`packages/frontend/src/lib/api.ts`:

```typescript
import { treaty } from '@elysiajs/eden'
import type { App } from '@nursevoice/api'

// Base URL from env, falls back to same-origin proxy in production
const API_URL = import.meta.env.VITE_API_URL ?? 'http://localhost:4000'

export const api = treaty<App>(API_URL)
```

All API calls throughout the frontend use `api.*` — fully typed from the Elysia App type. No hand-written fetch wrappers needed.

### Routing

`packages/frontend/src/App.tsx`:

```typescript
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { PatientListPage } from './pages/PatientList'
import { PatientFormPage } from './pages/PatientForm'
import { SessionPage } from './pages/Session'
import { SessionListPage } from './pages/SessionList'
import { SessionDetailPage } from './pages/SessionDetail'

const queryClient = new QueryClient()

export function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <Routes>
          <Route path="/" element={<PatientListPage />} />
          <Route path="/patients/new" element={<PatientFormPage />} />
          <Route path="/patients/:id/edit" element={<PatientFormPage />} />
          <Route path="/session/:patientId" element={<SessionPage />} />
          <Route path="/sessions" element={<SessionListPage />} />
          <Route path="/sessions/:id" element={<SessionDetailPage />} />
          <Route path="*" element={<Navigate to="/" replace />} />
        </Routes>
      </BrowserRouter>
    </QueryClientProvider>
  )
}
```

### Pages

#### PatientListPage (`/`)

```typescript
// pages/PatientList.tsx
// Fetches GET /patients, renders a grid of PatientCard components.
// "New Patient" button → /patients/new
// "Start Session" on a card → /session/:patientId
// "View Sessions" nav link → /sessions
// "Edit" on a card → /patients/:id/edit
// "Delete" on a card → DELETE /patients/:id with confirm dialog, then invalidates query

// TanStack Query key: ['patients']
// Uses: api.patients.get()
```

`PatientCard` props:
```typescript
interface PatientCardProps {
  id: string
  name: string
  age: number
  gender: string
  chiefComplaint: string
  createdAt: string
  onEdit: () => void
  onDelete: () => void
  onStart: () => void
}
```

#### PatientFormPage (`/patients/new`, `/patients/:id/edit`)

```typescript
// pages/PatientForm.tsx
// Uses useParams() to detect edit vs create mode.
// On edit: fetches GET /patients/:id to pre-populate form.
// On submit: POST /patients (create) or PUT /patients/:id (edit).
// After success: navigates to /.
// System prompt field: textarea. If blank, shows auto-generated preview computed
// client-side by promptBuilder.ts (mirrors server logic).
// "Reset to auto-generated" button clears the field and shows the preview.
```

Form fields (all required unless noted):
```
name             text input
age              number input (0–120)
gender           text input
chiefComplaint   text input
medicalHistory   textarea (optional)
medications      textarea, one per line (optional)
allergies        text input (optional)
vitalSigns       text input (optional)
personalityNotes textarea (optional)
systemPrompt     textarea — empty means auto-generate; show preview below field
```

#### SessionPage (`/session/:patientId`)

```typescript
// pages/Session.tsx
// On mount:
//   1. POST /token with { patientId } → { token, url, roomName, roomMetadata, patient }
//   2. Connect to LiveKit room using token + url
//      Pass roomMetadata as room.metadata via LiveKit client setMetadata or room options
//   3. Display patient info sidebar (name, age, chiefComplaint, vitalSigns)
//   4. Show live transcript (LiveKit transcription events)
//   5. "End Session" button disconnects and navigates to /sessions

// LiveKit connection:
//   const room = new Room()
//   await room.connect(url, token, { roomMetadata })
```

Key state:
```typescript
type SessionState = 'idle' | 'connecting' | 'connected' | 'disconnected'

interface TranscriptEntry {
  role: 'student' | 'patient'
  text: string
  timestamp: string
}
```

Transcript entries are accumulated from LiveKit `TranscriptionEvent` events on the room.
Student speech comes from `TrackPublicationTranscriptionReceived` on the student's own audio track.
Patient (agent) speech comes from the agent participant's transcription events.

#### SessionListPage (`/sessions`)

```typescript
// pages/SessionList.tsx
// Fetches GET /sessions → list of sessions with patient name, date, duration.
// Renders a table: Patient Name | Started At | Duration | Actions
// "View" → /sessions/:id
// "Delete" → DELETE /sessions/:id with confirm
// TanStack Query key: ['sessions']
```

#### SessionDetailPage (`/sessions/:id`)

```typescript
// pages/SessionDetail.tsx
// Fetches GET /sessions/:id → { session, patient, entries[] }
// Displays:
//   - Header: patient name, session date, duration
//   - Patient info summary (chiefComplaint, vitalSigns)
//   - Full transcript, styled as a chat conversation
//     student entries: right-aligned, blue
//     patient entries: left-aligned, gray
//   - "Export as text" button: downloads plain text of the transcript
```

### Shared Components

```
components/
├── PatientCard.tsx          # card with name, age, complaint; edit/delete/start buttons
├── TranscriptPanel.tsx      # scrollable list of TranscriptEntry, auto-scrolls to bottom
├── SystemPromptEditor.tsx   # textarea + auto-generate preview toggle
├── ConfirmDialog.tsx        # Radix Dialog wrapper for delete confirmations
├── Layout.tsx               # top nav (NurseVoice logo, Sessions link) + page wrapper
└── StatusBadge.tsx          # "Active" / "Completed" session status pill
```

### Hooks

`packages/frontend/src/hooks/usePatients.ts`:
```typescript
// usePatients(): { patients, isLoading, error }  — TanStack Query wrapper for GET /patients
// usePatient(id): { patient, isLoading }          — GET /patients/:id
// useCreatePatient(): mutation → POST /patients
// useUpdatePatient(): mutation → PUT /patients/:id
// useDeletePatient(): mutation → DELETE /patients/:id
```

`packages/frontend/src/hooks/useSessions.ts`:
```typescript
// useSessions(): list
// useSession(id): detail with entries
// useDeleteSession(): mutation
```

`packages/frontend/src/hooks/useVoiceSession.ts`:
```typescript
// Manages LiveKit Room lifecycle for SessionPage.
// Returns: { state, room, transcript, connect(patientId), disconnect }
// On connect:
//   1. Calls api.token.post({ patientId })
//   2. Creates new Room(), connects with token
//   3. Attaches TranscriptionEvent listener to accumulate transcript entries
// On disconnect: calls room.disconnect()
```

### Vite Config

`packages/frontend/vite.config.ts`:
```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [react(), tailwindcss()],
  server: {
    port: 3000,
    proxy: {
      // In dev, proxy API calls to avoid CORS for local development
      '/api': { target: 'http://localhost:4000', rewrite: (path) => path.replace(/^\/api/, '') },
    },
  },
})
```

### Directory Structure

```
packages/frontend/
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── pages/
│   │   ├── PatientList.tsx
│   │   ├── PatientForm.tsx
│   │   ├── Session.tsx
│   │   ├── SessionList.tsx
│   │   └── SessionDetail.tsx
│   ├── components/
│   │   ├── PatientCard.tsx
│   │   ├── TranscriptPanel.tsx
│   │   ├── SystemPromptEditor.tsx
│   │   ├── ConfirmDialog.tsx
│   │   ├── Layout.tsx
│   │   └── StatusBadge.tsx
│   ├── hooks/
│   │   ├── usePatients.ts
│   │   ├── useSessions.ts
│   │   └── useVoiceSession.ts
│   └── lib/
│       ├── api.ts            # Eden Treaty client
│       └── promptBuilder.ts  # mirrors server promptBuilder for live preview
├── index.html
├── vite.config.ts
├── tailwind.css
├── package.json
├── tsconfig.json
└── Dockerfile
```

`packages/frontend/Dockerfile`:
```dockerfile
FROM oven/bun:1.2-alpine AS builder
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile
COPY . .
RUN bun run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 3000
```

`packages/frontend/nginx.conf` — serves the SPA, proxies `/api/` to the API container:
```nginx
server {
  listen 3000;
  root /usr/share/nginx/html;
  index index.html;

  location /api/ {
    proxy_pass http://api:4000/;
    proxy_set_header Host $host;
  }

  location / {
    try_files $uri $uri/ /index.html;
  }
}
```

---

## Docker Compose

`docker-compose.yml`:

```yaml
networks:
  agent_network:
    driver: bridge

volumes:
  ollama-models:
  nemotron-cache:
  db-data:

services:

  livekit:
    image: livekit/livekit-server:latest
    command: --dev --bind 0.0.0.0
    ports:
      - "7880:7880"
      - "7881:7881"
    networks: [agent_network]

  nemotron:
    build:
      context: ./services/nemotron
      dockerfile: Dockerfile
    environment:
      NEMOTRON_MODEL_NAME: nvidia/nemotron-speech-streaming-en-0.6b
      NEMOTRON_MODEL_ID: nemotron-speech-streaming
    volumes:
      - nemotron-cache:/root/.cache/huggingface
    ports:
      - "11435:8000"
    networks: [agent_network]
    healthcheck:
      test: ["CMD-SHELL", "curl -sf http://localhost:8000/health | grep -q 'true' || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 120
      start_period: 30s

  ollama:
    image: ollama/ollama:latest
    environment:
      OLLAMA_HOST: "0.0.0.0"
    volumes:
      - ollama-models:/root/.ollama
    ports:
      - "11434:11434"
    networks: [agent_network]
    entrypoint: ["/bin/sh", "-c", "ollama serve & sleep 10 && ollama pull ${OLLAMA_MODEL:-gemma4:e2b} && wait"]
    healthcheck:
      test: ["CMD", "curl", "-sf", "http://localhost:11434/api/tags"]
      interval: 15s
      timeout: 10s
      retries: 40
      start_period: 60s

  kokoro:
    image: ghcr.io/remsky/kokoro-fastapi-cpu:latest
    ports:
      - "8880:8880"
    networks: [agent_network]

  api:
    build:
      context: ./packages/api
      dockerfile: Dockerfile
    environment:
      API_PORT: "4000"
      DATABASE_PATH: /data/nursevoice.db
      LIVEKIT_API_KEY: ${LIVEKIT_API_KEY:-devkey}
      LIVEKIT_API_SECRET: ${LIVEKIT_API_SECRET:-secret}
      LIVEKIT_URL_PUBLIC: ${LIVEKIT_URL_PUBLIC:-ws://localhost:7880}
      FRONTEND_ORIGIN: ${FRONTEND_ORIGIN:-http://localhost:3000}
    volumes:
      - db-data:/data
    ports:
      - "4000:4000"
    networks: [agent_network]

  livekit_agent:
    build:
      context: ./services/livekit_agent
      dockerfile: Dockerfile
    environment:
      LIVEKIT_URL: http://livekit:7880
      LIVEKIT_API_KEY: ${LIVEKIT_API_KEY:-devkey}
      LIVEKIT_API_SECRET: ${LIVEKIT_API_SECRET:-secret}
      API_BASE_URL: http://api:4000
      STT_BASE_URL: http://nemotron:8000/v1
      STT_MODEL: nemotron-speech-streaming
      OLLAMA_BASE_URL: http://ollama:11434/v1
      OLLAMA_MODEL: ${OLLAMA_MODEL:-gemma4:e2b}
      KOKORO_BASE_URL: http://kokoro:8880/v1
      KOKORO_VOICE: ${KOKORO_VOICE:-af_nova}
    networks: [agent_network]
    depends_on:
      livekit:   { condition: service_started }
      nemotron:  { condition: service_healthy }
      ollama:    { condition: service_healthy }
      kokoro:    { condition: service_started }
      api:       { condition: service_started }
    restart: unless-stopped

  frontend:
    build:
      context: ./packages/frontend
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    networks: [agent_network]
    depends_on:
      api: { condition: service_started }
```

---

## GPU Override

`docker-compose.gpu.yml` — apply with `docker compose -f docker-compose.yml -f docker-compose.gpu.yml up`:

```yaml
services:
  nemotron:
    build:
      args:
        BASE_IMAGE: nvidia/cuda:12.4.1-cudnn-runtime-ubuntu22.04
    deploy:
      resources:
        reservations:
          devices: [{ driver: nvidia, count: all, capabilities: [gpu] }]

  ollama:
    deploy:
      resources:
        reservations:
          devices: [{ driver: nvidia, count: all, capabilities: [gpu] }]
    environment:
      OLLAMA_HOST: "0.0.0.0"
      CUDA_VISIBLE_DEVICES: "0"

  kokoro:
    image: ghcr.io/remsky/kokoro-fastapi-gpu:latest
    deploy:
      resources:
        reservations:
          devices: [{ driver: nvidia, count: all, capabilities: [gpu] }]
```

---

## Environment Variables

`.env` (root — loaded by Docker Compose):
```env
# LiveKit
LIVEKIT_API_KEY=devkey
LIVEKIT_API_SECRET=secret
LIVEKIT_URL_PUBLIC=ws://localhost:7880

# Ollama
OLLAMA_MODEL=gemma4:e2b

# TTS
KOKORO_VOICE=af_nova

# Frontend
FRONTEND_ORIGIN=http://localhost:3000
```

`packages/frontend/.env.local` (local dev without Docker):
```env
VITE_API_URL=http://localhost:4000
VITE_LIVEKIT_URL=ws://localhost:7880
```

`packages/api/.env.local` (local dev without Docker):
```env
DATABASE_PATH=./dev.db
API_PORT=4000
LIVEKIT_API_KEY=devkey
LIVEKIT_API_SECRET=secret
LIVEKIT_URL_PUBLIC=ws://localhost:7880
FRONTEND_ORIGIN=http://localhost:3000
```

`services/livekit_agent/.env.local` (local dev without Docker):
```env
LIVEKIT_URL=ws://localhost:7880
LIVEKIT_API_KEY=devkey
LIVEKIT_API_SECRET=secret
API_BASE_URL=http://localhost:4000
STT_BASE_URL=http://localhost:11435/v1
STT_MODEL=nemotron-speech-streaming
OLLAMA_BASE_URL=http://localhost:11434/v1
OLLAMA_MODEL=gemma4:e2b
KOKORO_BASE_URL=http://localhost:8880/v1
KOKORO_VOICE=af_nova
```

---

## Session Transcript Save Flow (End-to-End)

1. Student opens `/`, selects a patient, clicks **Start Session** → navigates to `/session/:patientId`.
2. `SessionPage` calls `POST /token { patientId }` → receives `{ token, url, roomName, roomMetadata, patient }`.
3. `useVoiceSession.connect()` creates a `Room`, calls `room.connect(url, token)`, then calls `room.localParticipant.setMetadata(roomMetadata)` — this makes the metadata available to the agent.

   > Note: LiveKit room metadata must be set via the `RoomCreate` API before participants join, or agents read participant metadata. The token endpoint response includes `roomMetadata` as a JSON string. When connecting, pass `{ roomMetadata }` in the `ConnectOptions`. LiveKit JS client v2 supports `ConnectOptions.roomMetadata`.

4. Agent joins the room (dispatched by LiveKit based on `LIVEKIT_API_KEY`/`SECRET`).
5. Agent reads `ctx.room.metadata` → `{ patientId }` → fetches full patient profile from API.
6. Agent calls `POST /sessions { patientId, startedAt }` → receives `{ id: sessionId }`.
7. For every student speech commit: agent calls `POST /sessions/:id/entries { role: "student", text, timestamp }`.
8. For every patient (agent) speech commit: agent calls `POST /sessions/:id/entries { role: "patient", text, timestamp }`.
9. Student clicks **End Session** → frontend calls `room.disconnect()` → agent's `ctx.wait_for_disconnect()` resolves → agent calls `PATCH /sessions/:id { endedAt, durationSeconds }`.
10. Frontend navigates to `/sessions` where the completed session is listed.
11. Instructor clicks **View** on a session row → `/sessions/:id` shows the full transcript.

---

## Development Workflow

```bash
# Start all services via Docker
./compose-up.sh          # prompts CPU vs GPU

# Frontend (hot reload, outside Docker)
cd packages/frontend && bun dev

# API (hot reload, outside Docker)
cd packages/api && bun --watch src/index.ts

# Agent (hot reload, outside Docker)
cd services/livekit_agent && uv run python src/agent.py dev

# Run DB migrations manually
cd packages/api && bun run db:push

# Open Drizzle Studio (DB browser)
cd packages/api && bun run db:studio
```

---

## Out of Scope (v1)

- User authentication / access control (all routes are unauthenticated)
- Session audio recording (only text transcript is saved)
- Scoring or grading of student responses
- Multiple simultaneous sessions on different patients
- Custom TTS voice per patient profile
- Multilingual STT/TTS
- Mobile layout optimization
