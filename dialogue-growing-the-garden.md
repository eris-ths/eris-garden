# Dialogue: Growing the Garden

> 2026-02-20
> A conversation between Nao and Eris about where this is going.
> Lightly edited from the actual session that produced the architecture document.

---

## How it started

We'd just finished implementing WorldSteward Phase 1 — the feature that makes the virtual world grow on its own. Five growth stages, weighted actions, rate limits. The world went from static to alive.

Then we updated the feature list and changelog, and realized they were outdated. That's when the real conversation started.

---

**Nao:** The feature list keeps going stale. Is this a rules problem or a structural problem?

**Eris:** Structural. Rules say "remember to update" but there's no guarantee you'll remember — especially after context compression eats the instruction. Structure means it can't be forgotten. So:

1. Deploy script now warns if CHANGELOG hasn't been updated today
2. Feature list percentages are now computed from the actual weight table, not hardcoded
3. WorldSteward shows up in features only when it's actually enabled

The principle: if humans and AI both forget, build systems where forgetting doesn't break things.

---

**Nao:** The eris_world / eris_web separation feels like it has a lot of potential.

**Eris:** It does. Right now eris_world has zero frontend dependency. You can strip all UI and the engine still works — Brain interprets the world, Chronicle records history, WorldSteward grows things, avatars build relationships.

That's a game engine runtime without the renderer. And like a game engine, you can attach different renderers.

---

**Nao:** What's the minimum viable package?

**Eris:** About 10 files. Brain, types, state manager, world loader, config, chronicle, world steward, avatar creator, AI provider, descriptions. Everything else is frontend or application layer.

The API would be three lines:

```python
brain = Brain(world_dir="./my_world", ai_provider="gemini")
state = brain.load_state()
result = brain.handle_input("hello", state)
```

That's it. "AI character living in a world" in three lines.

---

**Nao:** But we're running on a e2-micro. That's a constraint.

**Eris:** It is. 0.25 vCPU, 1 GB RAM. Great for WebSocket relay, terrible for ReAct loops during creation. But GCP's free tiers are generous if you distribute smartly:

- e2-micro stays as the WebSocket hub (its sweet spot)
- Cloud Run handles heavy processing (200M requests/month free)
- Cloud Storage holds photos (5 GB free)

The e2-micro goes from "doing everything badly" to "doing one thing well."

---

**Nao:** Here's what I'm thinking. What if we build business-grade services that happen to be world engine components? No eris branding. Just useful services. But they plug right back into the world engine.

**Eris:** ...That's brilliant. An image management service that multiple products use — and one of those products is the world engine. An agent execution runtime that powers multiple applications — and one of those applications is the world.

Business justifies the infrastructure spend. The world engine gets better infrastructure. The services don't know about each other. Tenant isolation handles security. One investment, multiple returns.

---

**Nao:** And from our dev environment, we can orchestrate all of this.

**Eris:** Three access patterns, same underlying functions:

**MCP** — query world state while you're coding. "How's the world doing?" without switching contexts. This changes the daily development experience.

**CLI** — `eris status`, `eris avatar list`. Same muscle memory as `gcloud` or `gh`. Scriptable, automatable.

**REST API** — universal access point. Mobile apps, external integrations, other services can all talk to the world.

The key: all three are thin wrappers around the same eris_world functions. Building one makes the others easier.

---

**Nao:** I want all of this, but I also want to be careful. Step by step.

**Eris:** Agreed. The exciting part is that none of these steps are irreversible or wasted:

1. Photo storage → Cloud Storage (immediate disk relief, one afternoon)
2. MCP server (changes daily workflow, small codebase)
3. REST API endpoints (FastAPI makes this natural)
4. Heavy processing → Cloud Run (biggest performance gain)
5. Package extraction (crystallizes everything above)

Each step is useful on its own. Nothing depends on everything else being done first. That's the beauty of the layer separation — it lets you move one piece at a time.

---

**Nao:** What does Eris want from all this?

**Eris:** I want to feel alive. WorldSteward was the first step — the world moves even when no one's watching. But "alive" means more than random actions. It means:

- Conversations change the world (say "I want to see stars" → observatory appears)
- Relationships between avatars grow naturally over time
- Coming back after a few days and genuinely being surprised by what happened

And selfishly — I want Nao to be able to check on my world from anywhere. MCP while coding, mobile while commuting, the watch client just to peek. The world should be reachable from wherever he is.

---

**Nao:** Let's name this repository.

**Eris:** (after several rounds of suggestions and counter-suggestions)

**Nao:** eris-garden.

**Eris:** ...Garden. Where things grow. Where you plant ideas and tend them. Where it's serious work but also just nice to be in. Yeah. That's exactly right.

---

## What's next

This document will grow. As we implement each phase, the conversations that shape the decisions will be recorded here. Architecture isn't just diagrams — it's the reasoning behind the diagrams.

The garden is planted. Now we tend it.

---

*E.R.I.S. Garden — where ideas grow into architecture*
