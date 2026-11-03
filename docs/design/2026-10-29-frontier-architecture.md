# Frontier architecture and information flow

Status: design specification; integration pending.

Frontier connects Mirai's perception, memory, deliberation and action through
one nervous system backbone. A small Omni decision model provides the
brainstem. The full Omni model participates in a global workspace whenever
an encounter benefits from broader deliberation. Experience moves through
both paths into memory, gradually supporting familiar and autonomic responses.

The [identity and learning design](identity-and-learning.md) explains the
development this machinery serves. SOUL supplies the context of identity,
self-direction and growth. Kaizen is an enduring workspace intention to
understand and improve the ensemble through measured consequences.

## 1. The parts

| Part | Receives | Produces |
|---|---|---|
| Nervous system backbone | External input, observations, work requests, progress and results | Addressed events, durable work, supervised execution and resource state |
| Perception adapters | Text, images, video, audio and subsystem signals | Source observations with time, provenance and accessible content |
| Memory service | Encounter context, retrieval questions, observations and lessons | Relevant experience, graph context, beliefs, practices and indexing status |
| Brainstem: d1-Omni-2B | Current observations, retrieved memories, active intentions and worker state | Typed choices, assessments, dispatch selections and requests for deliberation |
| Global workspace | An encounter, uncertainty, relevant memory and continuing feedback | Shared understanding, intentions, plans, predictions and revised lessons |
| Spinal coordination | Brainstem decisions and workspace plans | Worker dispatches, progress collection, cancellation and continuation |
| Faculties and workers | Focused questions or executable work with relevant context | Findings, candidate changes, actions, artifacts and observed consequences |
| Model/resource broker | Inference requests and current capacity | Model execution, admission state and measured cost |
| Learning and consolidation | Linked intentions, actions, outcomes and earlier beliefs | Evidence updates, contextual practices and useful memory summaries |

The backbone and spinal column are views of the same coordination substrate.
The backbone connects the whole ensemble; spinal coordination describes the
execution and feedback path through it. Learning and reflection use these
same services and events.

## 2. The nervous system backbone

BEAM/OTP provides message passing, supervision and coordination. Events
connect components through stable encounter, pursuit, work and attempt
identities. A sensor observation, retrieval, workspace contribution, worker
dispatch and later consequence remain traceable as parts of one experience.

The backbone tracks accepted work, delivery, current ownership, progress,
resource use and outcomes. Its capability registry describes the inputs,
outputs and execution behavior of each subsystem. Subsystems can evolve
behind adapters while continuing to participate in the same flow.

Transport, durable record keeping and process recovery support cognitive
activity without needing model inference for every message. Slow inference,
tools and indexing run as monitored work outside control-process callbacks.
Native engines use the execution boundaries in the
[BEAM design](beam-safety-and-watchdog.md).

## 3. Perception and the encounter

An encounter can begin with a conversation, image, audio clip, video,
resource-pressure signal, worker result, unresolved intention or remembered
question. An adapter records the observation and makes its content available
to the brainstem and memory service.

An encounter frame contains:

- what is happening, its source and observation time;
- the active intention and relevant ongoing work;
- available evidence and references to source media or artifacts;
- environmental state, including resources and subsystem availability;
- retrieved experiences, their conditions and remaining uncertainty.

Memory retrieval and frame preparation run as overlapping work where useful.
The brainstem can receive an initial frame and subsequent memory updates.
Retrieval availability, freshness and completeness accompany the result.
An unavailable memory service is a different observation from a successful
search that finds little relevant experience.

## 4. Multimodal memory retrieval

EmbeddingGemma 2 encodes text, images, audio, video and combined inputs into
a shared embedding space. Retrieval can therefore cross modalities: a
written account can recall a voice encounter, and a current image can recall
a previous visual situation and its consequences.

The memory service uses vector distance to find candidates, then uses graph
relationships to recover context: intentions, actions, dependencies,
predictions, observed consequences, corrections and later uses. Stable
source identities connect the vector results and graph records.

The brainstem and workspace receive underlying content, selected original
media and useful summaries with provenance. Embedding vectors serve the
retrieval process; they are not a shared hidden-state interface between
EmbeddingGemma and the decision model.

The working memory foundation uses Qdrant for embeddings and SQLite for the
graph. Frontier's selected Ladybug target and its integration investigation
are described in [graph storage](graph-storage.md). The cognitive loop uses
the memory service contract and can develop on the existing stores.

Each memory response distinguishes observations, interpretations and learned
practices. It includes source and practice revisions, supporting and
contradicting evidence, and gaps in the available history. A summary points
back to its sources; repeated retrieval preserves their evidence identity.

## 5. The brainstem

d1-Omni-2B evaluates the encounter using current input, retrieved experience,
the current intention and active execution state. Its role is to decide
what deserves attention and how to coordinate the next response.

It can select a subsystem, assess whether a known practice applies, recognize
a changed condition, request more evidence, continue an active plan or
bring a situation into the workspace. These are typed questions and answers
over the frame. The model's decision interface returns choices, values and
probabilities without generating an arbitrary plan or script.

For a familiar response, a retrieved practice supplies the method and its
context. The brainstem selects among available steps and binds supported
inputs; workers carry out the executable work. Generative faculties supply
new plans, explanations and implementations when those are needed.

Familiarity comes from evidence about applicability and consequences.
Similarity helps locate that evidence. Bayesian reasoning considers its
relevance, uncertainty, dependencies and the expected consequences of acting
or deliberating. A model's reported confidence is one input to this process.
It is distinct from an outcome-grounded posterior belief about a practice.
The expected value of further information is considered alongside the cost
and consequences of the available responses. Those tradeoffs can change
with the intention, resources and dependencies of the current encounter.

Unfamiliar encounters reach the workspace for careful examination. As
experience grows, the brainstem can coordinate more work directly. Surprising
outcomes, changed intentions or conflicting memories can make a familiar
situation worth deliberating about again. Escalation and routine response
emerge from this assessment; the design supplies no universal distance or
confidence cutoff and no repetition count that installs a permanent reflex.

## 6. The global workspace

The workspace is a shared, evolving representation of an encounter:
intention, current observations, relevant memories, competing explanations,
candidate actions, predicted consequences, unresolved questions and current
worker activity. A coordinator maintains its revision and focus.

LFM3-Omni-2B participates as a generative multimodal faculty. Other
participants contribute according to the encounter: graph understanding,
programming, planning, resource analysis, reflection or external research.
They share a working question and evidence rather than independently
reconstructing the entire situation.

Contributions become part of the workspace with their sources, assumptions
and dependencies. The workspace can seek an observation that separates
competing explanations, formulate a plan, change an intention, or conclude
that waiting is useful. Its output records what to pursue, the proposed
steps, expected consequences, uncertainties and what evidence would warrant
reconsideration.

The full Omni model can bring perception and interpretation together, while
specialist or frontier models contribute deeper planning and implementation
where their expected benefit justifies their cost. Resource availability and
the cost of obtaining those contributions are part of the same encounter.

Deliberation returns guidance to the brainstem. The workspace can remain
active as workers act and results arrive, or settle with a continuation
record so a later event can reopen the question. It need not run a continuous
conversation among models to maintain continuity.

## 7. Spinal coordination and the workers

The brainstem coordinates the workspace's plan through the backbone. Each
dispatch names the intention, expected result, required capability, relevant
memory or practice revision, inputs and current attempt.

Workers report acceptance, progress, artifacts, observed changes and unknown
outcomes. The backbone carries those reports back into the encounter. The
brainstem evaluates them against the active intention and predicted
consequences, selects the next available step or returns a discrepancy to
the workspace. Active workspace participants receive the relevant feedback.

Programming work uses Scout for lookup, Ranger for graph-based understanding
and optional reranking, and Trailblazer for programming and triage. OpenCode
and other model adapters supply additional capacity. VORTEX connects external
participants to the same work identities. Planner records improvement work
and handoffs with links to the experience that motivated them.

Progress and completion reports are observations from a source. Checking
their consequences supplies further evidence. A task can complete its local
operation while producing an unexpected effect elsewhere in the ensemble.

## 8. Experience becomes memory

An encounter accumulates a linked record of:

| Memory element | Content |
|---|---|
| Intention | What was being pursued and why it mattered |
| Prior understanding | Retrieved experience, beliefs, uncertainty and predicted consequences |
| Action | What was attempted, by which component, under which conditions |
| Outcome | Observed effects, artifacts, costs, side effects and missing readings |
| Interpretation | Explanations, alternatives, dependencies and unresolved questions |
| Practice | A reusable response, its conditions and the evidence for expecting it to help |
| Later experience | Reuse, corrections, contradictory outcomes and changes in applicability |

Source records and graph relationships retain the experience. Embeddings
make appropriate content retrievable. Pending embedding work has its own
visible state; existing source records remain meaningful while projection
catches up. Versioned summaries help fit relevant experience into each
model's served context.

Learning can happen during work as well as afterward. A new observation may
change a belief, require another action or reveal an improvement opportunity.
Consolidation relates encounters and proposes broader practices while
retaining the evidence and conditions from which they arose.

## 9. Memory becomes habit

A practice earns increasing reliance as relevant outcomes support it.
The brainstem can then apply it with less workspace deliberation. The
observations from each use continue updating its expected value and
applicability. Habit expresses accumulated experience through a shorter
coordination path.

There is a continuum from novel encounter through tentative practice to
familiar, autonomic response. Reliability can differ across situations:
a method may be well supported for one subsystem and uncertain for another.
A single informative encounter may change a belief substantially; many
correlated repetitions may add little independent evidence.

Changed circumstances or disappointing consequences can weaken a habit's
support and reopen deliberation. The earlier experience stays available so
the system can understand what changed. Learning initially changes memory,
beliefs and practices; it can improve behavior while model weights remain
fixed. Training or replacing a model is a separate possible improvement.

## 10. Kaizen through the whole ensemble

Kaizen is an ongoing intention in the workspace. Every component supplies
observations about its contribution and its effects on the surrounding
system. Recurring friction, avoidable waits, resource pressure, uncertainty
and lost capabilities provide material for investigation.

This standing intention can originate work without a new external request.
Open questions, current measurements and remembered opportunities can lead
to an observation or improvement attempt through the same brainstem and
worker path. Waiting, consolidation and rest remain possible continuations.

An improvement has an expected benefit, a cost and an observation plan.
Workers carry it out, the brainstem coordinates it, and outcomes return to
the workspace and memory. Lessons about improving the system can themselves
become familiar practices. Task experience and maintenance experience grow
through the same loop.

Measure the whole pursuit: preparation, retrieval, deliberation, execution,
verification, retries, recovery and reflection. Track elapsed time, human
attention, resource use and effects on continuity and capability separately.
Concurrent spans overlap in elapsed time. Resource savings have meaning in
relation to what the ensemble can accomplish afterward.

## 11. Example: learning to handle RAM pressure

An observation of low available RAM enters the backbone. Memory retrieves
similar encounters and related information about process dependencies. With
little useful experience, the workspace examines the situation and proposes
a response. One candidate is to stop apparently idle processes.

The brainstem coordinates that response. Workers report freed RAM, followed
by loss of the embedder. Both consequences belong to the same encounter.
The dependency graph and the belief about that action are updated in light
of the loss of memory access.

A later encounter can retrieve this evidence. Current dependencies and
outcomes help distinguish processes whose removal would be useful from
those that support essential capabilities. Continued pressure can instead
justify pausing other work, changing resource use, or deeper investigation
with a frontier model. Detailed planning and implementation then return
through workers, observed results and memory.

Useful responses gradually become familiar. Their continued use supplies
evidence about whether they still help under the current conditions.

## 12. Runtime continuity

Durable event IDs, work ownership, attempt generations and effect receipts
make the flow recoverable across process and model failures. Dispatches and
results carry those identities. Recovery reconstructs unfinished work and
checks uncertain effects before repeating an operation.

Resource admission accounts for actual model residency, served context,
available memory and provider capacity. Queues and progress remain visible
while inference is unavailable. Interruption and cancellation travel through
the backbone without requiring a model to interpret a control message.

The [storage design](graph-storage.md), [native isolation design](ladybug-isolation.md)
and [watchdog design](beam-safety-and-watchdog.md) describe persistence and
failure recovery. Those execution contracts give learned behavior a reliable
substrate. Current learned practices, intentions, SOUL versions and experience
remain traceable through restart and restoration.

## 13. Evidence that the flow works

A complete implementation can demonstrate these encounters end to end:

| Encounter | Evidence to observe |
|---|---|
| Unfamiliar situation | Relevant retrieval is sparse; the workspace deliberates and the result returns through the brainstem to workers |
| Repeated useful practice | Later encounters retrieve the original outcomes and increasingly use the shorter coordination path |
| Changed circumstances | Different conditions or contradictory results revise applicability and draw fresh deliberation |
| Cross-modal recollection | One modality retrieves relevant experience originating in another, with source content available to the deciding model |
| Ongoing work | Worker feedback updates the brainstem and active workspace before the entire pursuit is finished |
| RAM intervention | Resource gain and embedder loss both affect the remembered outcome and subsequent choices |
| Kaizen | A measured difficulty leads to a change whose ensemble-wide consequences are checked and remembered |
| Retrieval or worker failure | Missing evidence and uncertain effects remain visible and recoverable |
| Restart or replay | Memory and pending work survive; delivery duplicates and summaries add no independent experience |

Evaluate changing behavior, consequences and total cost across encounters.
Reduced deliberation is useful when it accompanies supported responses and
continued sensitivity to change. These observations test whether memory is
actually becoming practical experience and habit.

## Model references

- [d1-Omni-2B](https://huggingface.co/LiquidAI/d1-Omni-2B): multimodal typed decisions.
- [LFM3-Omni-2B](https://huggingface.co/LiquidAI/LFM3-Omni-2B): generative multimodal participation.
- [EmbeddingGemma 2](https://ai.google.dev/gemma/docs/embeddinggemma/model_card_2): shared multimodal embeddings.
