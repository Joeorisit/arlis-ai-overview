# Architecture

[← Back to overview](../README.md)

Arlis is a TypeScript monorepo with five packages and a marketing site:

| Package | Role |
|---|---|
| **Mobile app** | The product. Expo + React Native, iOS-first, with a custom Swift module for recording. |
| **Backend** | Convex: schema, server functions, HTTP routes, scheduled jobs, auth, billing. |
| **Audio pipeline** | Audio → speaker-labeled transcript. Usable as a library from the backend or as a CLI. |
| **Eval harness** | Offline grading of AI output quality, with a regression gate. Never in the request path. |
| **Dev harness** | A throwaway web UI for exercising the backend during development. Not shipped. |

## System map

```mermaid
flowchart TB
    subgraph Clients
        MOB["📱 iOS app"]
        DEV["🧪 Dev harness (web)"]
    end

    subgraph Convex["Convex backend"]
        direction TB
        Q["Queries<br/>(reactive reads)"]
        M["Mutations<br/>(transactional writes)"]
        A["Actions<br/>(external calls)"]
        H["HTTP routes<br/>(streaming chat, Stripe webhook)"]
        C["Cron jobs"]
        DB[("Database + file storage")]
        Q --> DB
        M --> DB
        A --> M
        H --> A
        C --> M
    end

    PIPE["🎧 Audio pipeline"]
    EVAL["📊 Eval harness"]

    subgraph Ext["External services"]
        ASR["AssemblyAI"]
        LLM["OpenAI GPT-5 mini"]
        JUDGE["Cross-family judge model"]
        STRIPE["Stripe"]
    end

    MOB <--> Q
    MOB --> M
    MOB --> A
    DEV <--> Q
    A --> PIPE --> ASR
    A --> LLM
    STRIPE --> H
    EVAL -. "offline grading" .-> LLM
    EVAL -. "judges output" .-> JUDGE
```

**Why Convex?** It's the database, the server functions, and the realtime sync layer in one place. The app subscribes to queries and re-renders the moment data changes. When a transcript finishes, the visit screen updates by itself, with no polling and no hand-written REST API.

## Request paths

### 1. Recording → transcript

```mermaid
sequenceDiagram
    autonumber
    actor P as Patient
    participant App as iOS app
    participant BE as Backend
    participant Pipe as Audio pipeline
    participant ASR as AssemblyAI

    P->>App: Confirm recording authorization, tap Record
    App->>App: Native Swift recorder (≤ 90 min)
    App->>BE: Upload audio to file storage
    App->>BE: processVisitAudio(visitId)
    BE->>BE: status → transcribing
    BE->>Pipe: Validate, hash, check cache
    Pipe->>ASR: Transcribe (Universal-2)
    ASR-->>Pipe: Turns with speaker A / B
    Pipe->>Pipe: Map speakers → DOCTOR / PATIENT, format
    Pipe-->>BE: Transcript
    BE->>BE: Save transcript, status → transcribed
    BE-->>App: Live update, transcript appears
    App->>App: Delete on-device copy (unless user opted to keep)
```

- **Recording authorization** is captured on *every* recording attempt and stored as an append-only record.
- **Content-hash caching**: the pipeline keys results by the audio's SHA-256, so reprocessing the same file costs nothing.
- **Self-healing**: a watchdog job flips any visit stuck in `transcribing` for more than 30 minutes to `failed`, which unblocks retry.

### 2. Question → grounded answer

See [AI pipeline](ai-pipeline.md) for the full breakdown.

### 3. Appointment → recording

```mermaid
flowchart LR
    A["Home / Calendar"] --> B["Pick doctor"] --> C["Appointment form<br/>time · notes · questions"]
    C --> D[("visit<br/>status: scheduled")]
    D -- "Record now" --> E["Recorder<br/>(same visitId)"]
    E --> F[("visit<br/>status: pending → transcribed")]
```

An appointment isn't a separate object. It's a visit in `scheduled` status. When you record, the audio attaches to that same row, so your prepared questions, the recording, and the transcript stay together.

## Data model

Everything hangs off one central object: the **visit**. Every row is owned by an authenticated user.

```mermaid
erDiagram
    USER ||--o{ VISIT : owns
    USER ||--o{ CHAT_SESSION : owns
    USER ||--o{ MEDICATION : tracks
    USER ||--o{ QUICK_QUESTION : captures
    USER ||--|| ENTITLEMENT : has
    USER ||--|| RATE_LIMIT : has
    VISIT ||--o{ RECORDING_AUTHORIZATION : "authorized by"
    VISIT ||--o{ MEDICATION : "suggested from"
    CHAT_SESSION ||--o{ CHAT_TURN : contains
    CHAT_TURN }o--o{ VISIT : cites
    MEDICATION ||--o{ MEDICATION_COMPLETION : "logged as"

    VISIT {
        string status "scheduled | pending | transcribing | transcribed | missed | failed"
        string doctorName
        string transcript "speaker-labeled"
        object summary "cached key points"
        array scheduledQuestions "appointment prep"
        id audioFile "purged after 7 days"
    }
    CHAT_TURN {
        string question
        string answer
        array citations "visitId + verbatim span + verified flag"
        string refusalReason
        number cost
    }
    MEDICATION {
        string name
        string dose
        array times
        string reviewStatus "for AI suggestions"
    }
```

Design rules:
- **Soft delete everywhere.** Deleted visits stay addressable so old chat citations never break.
- **Every chat turn is logged**: question, loaded visits, raw and final answer, per-citation verification, model, token count, cost. That gives a full audit trail and per-turn cost visibility.
- **Idempotent writes**: client-generated IDs prevent duplicate visits on flaky networks, and the Stripe webhook log drops duplicate or out-of-order events.

## Background jobs

| Job | Frequency | What it does |
|---|---|---|
| Purge audio | Daily | Deletes server audio older than 7 days, in bounded batches that reschedule themselves |
| Transcription watchdog | Every 5 min | Marks visits stuck in `transcribing` for more than 30 min as `failed` |
| Missed appointments | Daily | Marks past-due `scheduled` visits as `missed` after a 1-day grace period |

## Auth & billing

- **Sign-in:** Sign in with Apple (native), Google OAuth, or email + password.
- **Subscription:** Stripe Checkout. A webhook updates the user's entitlement, and paid features check it server-side before running.
- **Rate limits:** per-user burst and daily caps on chat to protect against abuse and runaway cost.
