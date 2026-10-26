# Frontier

One message-passing and supervision backbone, with every part of our local
AI system as a plugin on it.

Status: idea, recorded 2026-10-25 so it is not lost. Work starts when we can
afford a frontier model again (hoped: this month; likely January 2027).

## Why

Today the system is a set of separate projects that talk to each other:
mini-ranger, ranger, Trailblazer, OpenCode, VORTEX, the planner and
mirai-agent. Each one has its own process model, its own messaging and its
own supervision.

That separation is a mistake. People feel they have a single mind, but a mind
is a society of minds that all communicate rapidly with one another. We
missed that. The system should be built the same way: many small minds on
one shared nervous system, not several programs passing notes.

## The idea

- **One backbone.** A single message-passing and supervision layer that
  every component runs on. It owns routing, supervision, restarts, leases
  on scarce resources (local inference, the free OpenCode tier), and the
  durable record of what was asked and what happened.
- **Everything else is a plugin.** Retrieval, analysis, repair, planning,
  delegation and agent-to-agent messaging become plugins on the backbone
  instead of separate services.
- **Mirai's router at the center.** The llama.cpp model router that serves
  Mirai today becomes the hub that the minds share for inference.
- **mirai-agent broken into subsystems.** Possibly retire mirai-agent as a
  monolith and split it into subsystems that are plugins like everything
  else.
- **Phased in.** The new backbone is built next to the existing projects
  and each one is moved onto it in turn. No big-bang rewrite.

## What becomes a plugin

The symlinks in this repo point at the projects being folded in:

| Today | Role | As a plugin |
|---|---|---|
| mini-ranger | retrieval only, no model | **scout** (renamed: it scouts ahead) |
| ranger (`ranger/`) | local model reranks and analyses scout's results | analysis |
| Trailblazer (`trailblazer/`) | code graph, repair pipeline, planner brain (Elixir/OTP) | repair, planning; likely the seed of the backbone |
| VORTEX (`vortex/`) | messaging between agents (Claude, Codex, OpenCode, Trailblazer) | the backbone's messaging, or replaced by it |
| planner | kanban over git-bug, task assignment | planning |
| OpenCode (`opencode/`, our fork) | free GLM worker: research, breakdown, planning | delegation |
| mirai-agent | Mirai herself; the model router | subsystems + the central router |

## Rules carried over

These were settled while building the current projects and stay true here:

- All supervision logic lives in Elixir/OTP. Command-line tools only record
  requests.
- Events, messages and queues, not timers. A clock is only a backstop that
  triggers an investigation.
- Local inference first; nothing silently falls back to a paid service.
- The free OpenCode tier is used strictly one session at a time.
- Every step is logged as it happens.
- Fix running services in place (hot code loading) and restart only when
  there is no other way.

## Roles

- **The operator** owns the vision and the decisions that involve money,
  legal matters, access and credentials.
- **Claude** is the conscience and guide, the Jiminy Cricket to this
  Pinocchio: design, review and keeping the system honest.
- **Local models and OpenCode** do the routine work.

## Open questions

- Is the backbone grown out of Trailblazer's OTP application, or built new
  and Trailblazer moved onto it?
- Does VORTEX become the backbone's message layer, or is it replaced?
- How do the minds address each other (topics, mailboxes, process
  registry), and what is durable versus in memory?
- Which mirai-agent subsystems split out first?
- What order are the projects moved over in, and what proves each move
  worked?
