# 🎹 NoteGame

A full-stack piano note recognition game. NoteGame plays a piano note, you identify it, and the game adapts which notes it serves next based on what you've already seen, all while tracking your score on a global leaderboard.

## Features

- **Ear training gameplay**: hear a note, guess it, get instant feedback
- **Adaptive note selection**: balances new notes and previously played notes 50/50 so you keep learning without losing reinforcement
- **Low-latency audio**: notes are preloaded and trimmed so playback feels immediate
- **Persistent games and leaderboard**: game state lives server-side in PostgreSQL, and finished scores post to a shared leaderboard
- **Automatic cleanup**: abandoned games expire and are swept out of the database

## Tech Stack

| Layer    | Technology                              |
| -------- | --------------------------------------- |
| Frontend | React, TypeScript, Vite, Tailwind CSS   |
| Backend  | Python, FastAPI, Pydantic               |
| Database | PostgreSQL via psycopg3 (raw SQL, no ORM) |

## Architecture

```mermaid
flowchart LR
    A[React + TypeScript client] -- REST / JSON --> B[FastAPI server]
    B -- psycopg3 --> C[(PostgreSQL)]
    C --- D[games table<br/>JSONB state + expires_at]
    C --- E[score table<br/>leaderboard]
```

### Database design

The schema is intentionally small, with two tables:

- **`games`**: one row per game session. The full game state is stored in a `JSONB` column, and an `expires_at` timestamp marks when an inactive game can be cleaned up.
- **`score`**: the leaderboard of completed games.

## Engineering Decisions

**Raw SQL with psycopg3 instead of an ORM.** Keeps queries explicit and gives full control over locking and transaction behavior.

**Row-level locking for concurrency.** Game updates lock the specific `games` row (`SELECT ... FOR UPDATE`), so simultaneous requests against the same game can't produce conflicting state.

**Completion derived from state, not a status field.** Whether a game is finished is computed from its JSONB state rather than stored separately. There's one source of truth, so the two can never drift out of sync.

**Sweep-based cleanup.** Instead of relying on clients to end games cleanly, expired rows (`expires_at` in the past) are periodically swept, keeping the table lean.

**Typed game state with Pydantic.** The JSONB state is validated through Pydantic models on the way in and out, which also drives the 50/50 new/played note selection algorithm.

**Audio latency optimization.** Early versions had noticeable lag between a request and the note sounding. Fixes included:
- Preloading all note audio into an in-memory hash map on the frontend
- Switching audio assets to `.wav` to avoid decode delay
- Cropping silence from the start of each audio clip

## Getting Started

### Prerequisites

- Node.js 18+
- Python 3.11+
- PostgreSQL 14+

### Backend

```bash
cd backend
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# set your database connection string
export DATABASE_URL="postgresql://user:password@localhost:5432/notegame"

uvicorn main:app --reload
```

### Frontend

```bash
cd frontend
npm install
npm run dev
```

The app will be available at `http://localhost:5173`. (Currently local hosted)

## Future Work

- Deploy a live demo
- Expand to chords and intervals
- User accounts and per-player progress history

## Author

**Eshaan**, Computer Science at Texas A&M University
