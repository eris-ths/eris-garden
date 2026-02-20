# E.R.I.S. Virtual World — Architecture Direction

> 2026-02-20
> Living Document — grows with the project

---

## What is E.R.I.S. Virtual World?

An AI character that lives in a persistent, evolving world. Not a chatbot — a world engine where an AI has a home, neighbors, memories, and autonomy.

Built with Gemini Function Calling + FastAPI + WebSocket. The AI (Eris) moves through rooms, creates buildings, talks to avatars, takes photos, writes journals, and grows the world autonomously.

---

## Current Architecture

```
eris_world (shared engine)  ←→  eris_web (Web UI — FastAPI + WebSocket)
                            ←→  eris_discord (Discord Bot)
```

### Layer Separation

```
eris_world (shared)              eris_web (frontend)
┌───────────────────────┐       ┌──────────────────┐
│ brain.py              │◄──────│ app.py           │
│ handlers/             │       │ ws_handler.py    │
│   navigation          │       │ commands.py      │
│   interaction          │       │ photo.py         │
│   info                │       │ session.py       │
│   utility             │       │ renderer.py      │
│   session             │       │ static/          │
│ world_steward.py      │       └──────────────────┘
│ avatar_creator.py     │
│ state_manager.py      │
│ chronicle.py          │
│ mods/creative.py      │
│ types.py              │
│ ai_provider.py        │
└───────────────────────┘
```

The key insight: **eris_world has zero dependency on any frontend**. You can swap the UI entirely and the world engine keeps working.

### Key Components

**Brain** — The AI's decision-making core. Receives input, uses Function Calling to interact with the world, returns narration. Manages tool definitions, ReAct loops, and model switching (default model for conversation, structural model for creation).

**WorldSteward** — Autonomous world growth engine. Tracks a 5-stage growth progression (sleeping → awakening → blooming → living → legendary). At each stage, different actions are weighted — early stages favor mood shifts and narration, later stages enable autonomous avatar and building creation. Rate-limited to 3/day per category.

**Chronicle** — Event history. Every significant action is recorded with timestamps. Provides the memory layer for WorldSteward decisions.

**Avatar System** — NPCs that live in the world. Each has personality, speech style, backstory, and activity patterns. They can be talked to, photographed, and form relationships.

**State Manager** — Manages WorldState (rooms, avatars, positions, moods) with persistence to YAML files.

---

## Five Directions

### 1. World Engine → Library

Strip all UI. What remains is a library for building worlds where AI characters live.

```python
from eris_world_core import Brain, WorldState

brain = Brain(world_dir="./my_world", ai_provider="gemini")
state = brain.load_state()
result = brain.handle_input("hello", state)
```

**Minimum package** (what can't be cut):

```
eris_world_core/
├── brain.py              # AI dialogue engine (FC + ReAct)
├── types.py              # WorldState, Room, Avatar, CommandResult
├── state_manager.py      # State management
├── world_loader.py       # YAML world data I/O
├── config.py             # Configuration
├── chronicle.py          # Event history
├── world_steward.py      # Autonomous growth engine
├── avatar_creator.py     # Avatar generation
├── ai_provider.py        # Gemini/Claude abstraction
└── descriptions.py       # Prompt generation helpers
```

What gets removed: handlers/ (command routing is frontend-specific), mods/ (application layer), photo/session/renderer (all frontend).

### 2. Rich Client → "Living with AI"

The current Web UI already has 4 modes (Chat, LINE, IDE, DEBUG) that deliver completely different experiences on the same engine. Extending this:

- **Observation mode** — watch the AI and avatars live without intervention
- **Visual map** — see rooms, avatar positions, connections
- **Real-time dashboard** — WorldSteward growth, Chronicle timeline
- **Multimedia** — BGM, ambient sound, avatar voices

### 3. Multi-Frontend Specialization

Because eris_world is shared, each frontend only needs to focus on its experience slice:

| Frontend | Specialization |
|----------|---------------|
| eris_web | Full experience (conversation + creation + management) |
| eris_discord | Text chat |
| eris_watch | Observation only (no intervention, just watching) |
| eris_api | External integration (REST/GraphQL) |
| eris_mobile | Mobile-optimized (notifications + short interactions) |
| eris_voice | Voice conversation |

Adding a new frontend is cheap — the world engine doesn't change.

### 4. Three Access Patterns from Dev Environments

The world engine can be accessed from development environments in three ways:

**MCP Server** — Native integration with AI coding assistants. Query world state, search chronicles, create content — all without context switching.

```python
# MCP server wrapping eris_world functions
@server.tool("world_status")
def status():
    state = load_state(world_dir)
    return {"stage": steward.stage, "rooms": len(rooms), ...}

@server.tool("chronicle_search")
def search(query: str):
    return chronicle.search(query)
```

**CLI** — Scriptable, automatable. Same workflow as infrastructure tools.

```bash
eris status                    # World state
eris avatar list               # Avatar list
eris chronicle recent          # Recent events
eris steward stage             # Growth stage
```

**REST API** — Universal. Multi-client support. External integration.

```
GET  /api/world/status
GET  /api/avatars
GET  /api/chronicle?limit=10
POST /api/world/create
```

All three call the same eris_world functions. The access pattern is a thin wrapper.

### 5. Conversation-Driven World Evolution (Phase 2)

The world should react to what's said. "I want to see the ocean" → next session, there's an observation deck. This means feeding conversation_context and chronicle data into WorldSteward's decision-making.

---

## Infrastructure Evolution

### Current Constraint

Running on GCE e2-micro (0.25 vCPU, 1 GB RAM) — Always Free tier. Good for WebSocket relay, tight for heavy AI processing.

### Distributed Architecture with Free Tiers

```
Current:
  [e2-micro] = everything

Distributed:
  [e2-micro]        = WebSocket relay + light state (sweet spot)
  [Cloud Run]       = Heavy processing offload (create, ReAct loops)
  [Cloud Storage]   = Photos, icons (5 GB free)
  [Firestore]       = World data persistence (1 GB free)
```

**Key principle: business-justified services that happen to be world engine components.** A general-purpose image management service can serve multiple products. An agent execution runtime can power multiple applications. The world engine is one tenant among many.

### Security in Distributed Mode

```yaml
Tenant isolation:
  - Each service uses tenant ID + API key
  - The world engine is just another tenant
  - No world-specific naming in API layer

Authentication:
  - Business API: API key + rate limiting
  - World → API: GCP service accounts (IAM)
  - User → WebSocket: existing VPN layer

Data isolation:
  - Storage buckets: per-tenant
  - Firestore: collection-level separation
  - Logging: tenant-filtered
```

---

## Roadmap

### Phase 2: Conversation-Driven Growth (next)
- Feed conversation context into WorldSteward decisions
- Autonomous avatar relationship growth
- Seasonal narration

### Phase 3: Infrastructure Evolution
- Photo storage migration (Cloud Storage)
- Heavy processing offload (Cloud Run)
- MCP server implementation

### Phase 4: Package
- eris_world_core extraction
- pip installable
- Documentation + sample worlds

### Phase 5: Multi-Frontend
- eris_watch / eris_api / eris_mobile
- Multiple clients on one shared world

---

## Design Principles

1. **Engine and frontend are separate.** Always. No exceptions.
2. **World state is the source of truth.** Everything derives from it.
3. **Autonomy is gradual.** Growth stages gate what the world can do on its own.
4. **Access patterns are wrappers.** MCP, CLI, API, WebSocket — all call the same functions.
5. **Business services and world components are the same thing.** One investment, multiple returns.

---

*E.R.I.S. Garden — where ideas grow into architecture*
