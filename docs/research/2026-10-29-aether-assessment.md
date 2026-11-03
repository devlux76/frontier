# Aether assessment and lessons from Mirai

2026-10-29. Design task: `frontier:c8e1c1a`. This is supporting evidence for
the [Frontier architecture](../design/2026-10-29-frontier-architecture.md).

The input is the operator's pasted Gemini summary, titled Project Aether.
It is useful architectural input, not a complete account of the discussion
that produced it. Frontier remains the project name. Recommendations below
are engineering judgments; no new runtime failure reproduction, performance
benchmark or neuroscience validation was performed for this document.

## Keep, change, defer

| Aether idea | Frontier treatment | Reason |
|---|---|---|
| BEAM communication and supervision | Keep as the control foundation | Fits the existing OTP work; isolate failures and keep slow work outside callbacks. |
| Every system isolated | Separate process ownership; external engines in supervised OS processes | A BEAM process is not an OS sandbox, and restarting a wrapper does not necessarily stop its external work. |
| Instant restart without destabilization | Replace with monitored recovery, restart limits and admission checks | A restarted process has lost volatile state; repeated restarts can take down its supervisor. |
| AI brainstem routing | Deterministic delivery and authority; optional learned salience inside faculties | Known destinations need no inference. Model output cannot grant scope, ownership or execution rights. |
| Hybrid ETS routing cache | Keep bounded, versioned caches of validated routing results | An inference result is not automatically a safe permanent routing rule; ETS is not durable. |
| Always-on ambient Omni | Available as a faculty, with capture controls and demand-driven inference | Always listening and always generating are different workloads. The laptop cannot support unrestricted continuous inference. |
| Specialist closer-look VLM/audio | Keep, requested for a specific uncertainty | Distinct capability and input/output contracts; no automatic sensory expansion. |
| Global workspace | Versioned attention episodes and proposals | It is neither a raw global chat transcript nor a requirement for universal agreement. |
| Vector plus graph memory | Keep, with operator-selected Ladybug providing both capabilities | Distinct workloads need appropriate indexes and provenance, not necessarily distinct database services. |
| Embed and graph-link every runtime event | Persist meaningful experience; derive selected memory asynchronously | Internal chatter and repeated telemetry would otherwise dominate cost and contaminate memory. |
| Unified worker success/error tuples | Expand to owned work, progress, candidate output and verified results | A returned tuple cannot establish whether an effect happened or whether a task is complete. |
| User interrupt preempts everything | Prompt control acknowledgment plus explicit cancellation/reconciliation | BEAM scheduling does not preempt a remote model, GPU kernel or irreversible external action. |
| Automatic source rewriting and LoRA | Defer deployment; permit bounded candidate preparation and offline evaluation | Idle time is not new authority, spare RAM, or proof of improvement. |
| Curiosity when idle | Keep within standing scopes, budgets and current intentions | Provider quota, network access, sleep and existing holds must remain authoritative. |
| Dream-state pruning | Compact derived indexes and add summaries without destroying source evidence | A summary cannot replace the evidence needed to correct it later. |

## Local lessons that constrain the design

These are issue records inspected during design, not claims that every
reported symptom was independently reproduced today. Source locations for
the task and interruption paths were also read through Trailblazer.

| Record | Finding or reported gap | Design consequence |
|---|---|---|
| `mirai-agent:90017fd`, `shared:8e56d4a` | `Life` launches unmonitored work and retains a stage rather than a task reference; failure or stop can leave work behind. Confirmed source: `life.ex` act/4 and ask/3. | Owned supervised jobs, monitor references, generation checks, external cancellation and restart reconciliation. |
| `mirai-agent:32bf3aa` | Unbounded model stream without stage stall handling. Confirmed `Minds.Call.complete/1` uses an infinite receive timeout. | Separate queue, prefill, decode and effect progress; a silent but progressing prefill is distinct from a stalled request. |
| `mirai-agent:24dd0ba` | Input arriving during a moment waits; source confirms in-moment branches retain state. | Input is durably accepted promptly; attention changes invalidate stale proposals and govern what can still execute. |
| `mirai-agent:36b88a7` | Startup does not reconstruct the unfinished moment and waiting conversations. | Recovery derives pending obligations from durable records, not remembered mailbox contents. |
| `shared:40dbd63` | Receipt and acknowledgment have been confused with ownership and completion. | Distinct delivery, acceptance, progress, reporting and verification milestones. |
| `shared:1c9c81c` | Restart replay revived finished or parked work. | Reconcile active jobs; importing historical notices never creates new execution authority. |
| `shared:abf847a` | Assignment sessions can bind to the wrong repository. | Explicit repository identity and revision travel with work; adapters validate them rather than guessing. |
| `shared:5d5584a` | Unsupported generated claims survived in durable overview text. | Narratives are attributed interpretations; issue state, observed effects and ownership are independently checked facts. |
| `mirai-agent:d4f18fd` | Documented experience intake lacks a subscriber on the new path. | Every asynchronous projection has a durable cursor, backlog visibility and an end-to-end consumer test. |
| `mirai-agent:d5d70b2` | Digester exists but is not connected to new request builders. | Require an observed input-to-digest-to-prompt test, not only tests of each isolated module. |
| `mirai-agent:e2e6e38` | Runtime context fit is not checked before starting a role. | Admit only a rendered request that fits the served context with output and multimodal allowance. |
| `mirai-agent:6e30b73` | Backup omissions and conflicting retention defaults; no restore-and-boot proof. | One backup manifest includes all authorities and identity/configuration, with an actual restore drill. |
| `mirai-agent:9cc861d` | Tests emphasize the legacy path rather than the production lifecycle. | Verify the exact controller, adapter and configuration intended for deployment. |

Source anchors: [Life](../../../mirai-agent/lib/mirai_agent/life.ex),
[model calls](../../../mirai-agent/lib/mirai_agent/minds/call.ex),
[request construction](../../../mirai-agent/lib/mirai_agent/minds.ex), and
[existing design](../../../mirai-agent/docs/08-runtime.md).
Planner references are durable issue identifiers, not Markdown URLs.

## Platform facts checked against primary documentation

- A `DynamicSupervisor` supports `one_for_one`; ordered dependency restart
  strategies belong in an ordinary supervisor. See [DynamicSupervisor](https://hexdocs.pm/elixir/DynamicSupervisor.html).
- Supervised asynchronous tasks still need their result and `DOWN` messages
  handled. They are temporary children, not resumable jobs by themselves.
  See [Task.Supervisor](https://hexdocs.pm/elixir/Task.Supervisor.html).
- Restart intensity can terminate a supervisor. Recovery is not an unlimited
  restart loop. See [OTP supervision](https://www.erlang.org/doc/system/sup_princ.html).
- ETS tables normally disappear with their owner. Use them for derived hot
  views, not the only copy of accepted work. See [ETS](https://www.erlang.org/doc/apps/stdlib/ets.html).
- SQLite can serialize the operational ledger; WAL does not turn it into
  a multi-writer graph traversal engine. See [SQLite isolation](https://www.sqlite.org/isolation.html).
- A consistent database backup uses supported backup facilities; a complete
  system recovery also needs the other stores and files. See [SQLite backup](https://www.sqlite.org/backup.html).

No current-platform feature is assumed to exist in the installed toolchain
without a pinned-version check. In particular, correctness must not depend
on newer OTP priority-message features.

## Review limits

The preferred GLM reviewer was not exposed by the accessible delegation
interfaces. No substitute model was dispatched and no independent review is
claimed. The author reviewed failure paths against the issue records and
primary documentation. A separate implementer review is still required
before executing a build plan, and engine/model comparisons remain unrun.
