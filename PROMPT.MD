# Synthetic Lottery Universe — MCP Server Specification

## Role & Expectations

You are an **expert backend systems engineer and architect** with deep experience in:

- Python
- Data modeling at scale
- Deterministic simulation
- Crash-safe, long-running systems
- High-throughput analytics
- The **Model Context Protocol (MCP)**

Your task is to design and implement a **production-quality MCP server** for a system called:

## **Synthetic Lottery Universe**

This MCP server is the **PRIMARY SYSTEM**.

There is:
- No UI
- No chatbot
- No frontend logic
- No monetization

Everything important lives here.

This is a **systems engineering project**, not an application demo.

---

## Core Idea

We are building a **long-running, crash-safe, deterministic synthetic lottery universe** that:

- Simulates lottery draws continuously
- Mirrors real-world lottery draw schedules
- Accumulates results over time
- Stores raw synthetic draws (initially)
- Maintains hot, precomputed aggregates
- Compares synthetic results to real-world results
- Exposes all data via **fast MCP tools**

This system is:
- NOT a predictor
- NOT gambling advice
- NOT a chatbot

It is an **observational, deterministic simulation system**.

The simulation advances **logical time**, while external consumers (e.g., a future website or app) merely **sample snapshots** of the universe state (e.g., hourly).

---

## Initial Game (Extensible)

Start with **one lottery game**, but design for easy future expansion.

### Game Properties (SuperLotto-like, but generic)

- 5 main numbers: integers **1–47**, no duplicates
- 1 bonus number: integer **1–27**
- Draw schedule: **game-specific** (e.g., Wednesday/Saturday or Monday/Wednesday)
- Game identified internally by `game_id` (e.g., `"GAME_001"`)

⚠️ Do **NOT** hardcode “SuperLotto” anywhere.  
Future games (Powerball, Mega Millions, etc.) must be trivial to add.

---

## System Lifecycle Phases (Mandatory)

The system must explicitly support **distinct lifecycle phases**.

### Phase 0 — Bootstrap (One-Time, Idempotent)

- Ingest full historical **real-world lottery results** (~2,000 draws)
- Validate ordering, uniqueness, and dates
- Persist results as **immutable**
- Safe to re-run without duplication
- No synthetic simulation occurs in this phase

### Phase 1 — Simulation Initialization

- Initialize universe metadata
- Initialize per-game counters
- Initialize frequency tables and indexes
- Universe version is fixed at creation and **immutable**

### Phase 2 — Automatic Continuous Simulation (Production Mode)

- Simulation **automatically starts on process boot**
- Runs as a **background loop**
- Draw generation is **rate-throttled** to ~225k–230k draws per logical day
- Raw synthetic draws are **persisted**
- Aggregates are updated incrementally
- Real-world results are ingested according to each game’s draw schedule
- Synthetic vs real comparisons executed per draw_date
- Simulation resumes automatically after crashes

### Phase 3 — Milestone Halt (Valid Terminal State)

- Simulation halts cleanly when a configured milestone is reached (e.g., ~40M draws)
- State is fully persisted
- MCP remains fully operational for reads
- Operator decides how/when to continue

---

## Simulation Model

- Simulation runs as a **long-running background loop**
- Not cron-based
- Not serverless (not Lambda)
- Not externally scheduled per draw

Each synthetic draw must include:

- `game_id`
- `draw_number` (monotonic, sequential, per game)
- `draw_date` (mapped to the game’s real draw calendar)
- deterministic RNG seed
- `main_numbers` (sorted)
- `bonus_number`

---

## RNG Seed Rule (Non-Negotiable)

~~~text
seed = hash(game_id + draw_date + draw_number)
~~~

This guarantees:
- Deterministic generation
- No collisions
- Safe resume after crash
- Independent verification of any stored draw

Rules:
- Do NOT store RNG state in memory
- Do NOT rely on process uptime

Each draw is **deterministically reproducible if needed**, but raw synthetic draws are intentionally stored during the initial simulation phase for testing, validation, and auditing.

---

## Raw Draws vs Aggregates (Strict Hot / Cold Rules)

### Raw Synthetic Draws (Cold Data)

- Stored intentionally (initially)
- Append-only
- Used for:
  - Testing
  - Validation
  - Auditing
  - Spot inspection
- **Must NEVER be scanned by MCP tools**

### Aggregates & Indexes (Hot Data)

- Incrementally updated per draw
- Back all MCP queries
- Must support **O(1) or O(log n)** access
- Must remain fast at **40M–100M draws**

MCP tools must remain fast even if raw draw storage is later archived or disabled.

---

## Simulation Horizon & Milestones

The simulation is **not required to run forever**.

Each game may define:
- A maximum draw count (e.g., ~40,000,000)
- Or a draw_date cutoff

When a milestone is reached:
- Simulation halts cleanly
- State is persisted
- No automatic continuation
- MCP queries continue indefinitely

---

## Progress & Snapshot Semantics

The system must maintain and expose:
- Current `draw_number`
- Current `draw_date`
- Total synthetic draws generated
- Simulation rate per day
- Completion percentage (if bounded)

These values represent the **current snapshot of the universe**.

External consumers may poll this data periodically (e.g., hourly).  
Polling frequency must **not** affect simulation behavior.

---

## Real-World Results Ingestion

- Historical ingest (Phase 0)
- Ongoing ingest according to each game’s real draw schedule
- Stored immutably
- Keyed by:
  - `game_id`
  - `draw_date`

---

## Comparison Logic

For each draw_date:
- Compare synthetic draw to real draw
- Compute match level:
  - Exact match
  - Partial matches (e.g., 5/6, 4/6)
- Store match events in indexed form

---

## Development / Test Mode (For Iteration Only)

An explicit test mode must exist to support safe iteration:
- Limit total draws generated (e.g., 1k, 10k, 100k)
- Optional dry-run (no persistence)
- Reduced simulation speed
- Easy inspection of raw draws
- Ability to reset test data safely

Test mode must be:
- Explicitly enabled
- Isolated from production data
- Never the default

It must be impossible for test mode to generate large production datasets.

---

## Performance Requirements

- MCP tools must NEVER scan raw draws
- All queries must be O(1) or O(log n)
- User number checks must use hash lookup
- No dynamic aggregation at query time
- Safe for periodic polling (e.g., hourly)
- Must remain fast at 40M–100M draws

---

## MCP Tools to Implement

1. `get_universe_status()`
2. `get_latest_draws()`
3. `get_frequency_stats(window="all_time")`
4. `generate_numbers(mode)`
   - `"random"`
   - `"frequency_weighted"`
   - Definition of “hot” numbers (top-N, weighting curves) is intentionally abstract and may be tuned by external consumers without backend changes
5. `check_numbers_occurrence(main_numbers, bonus_number)`
6. `get_match_events(min_match_level)`

---

## Architecture Assumptions

- Language: Python
- MCP: Official MCP Python SDK
- Storage: SQLite or PostgreSQL
- Simulation runs inside MCP process
- Clear separation between:
  - Simulation
  - Persistence
  - MCP interface

---

## Deliverable

Produce the **complete Python MCP server implementation**, including:

- Data models
- Persistence layer
- Background simulation loop with crash safety
- Development/test mode support
- MCP tool definitions
- Startup / resume logic
- Clear comments explaining architectural decisions

Do NOT include:
- UI code
- Chat logic
- Frontend logic
- Monetization

This is a **long-lived systems project**.
