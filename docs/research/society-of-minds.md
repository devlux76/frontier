# Research option: a society of minds

Proposed by the operator, 2026-10-29. Documentation task:
`frontier:07fa4f3`.

Status: an alternative direction to examine for Frontier. No architecture
change, experiment, implementation or migration is authorized by this
proposal. The current system partly works and must be preserved. We
should treat the two control architectures as competing options rather
than assume the proposed society is an incremental extension of the
existing two minds.

2026-10-30: the [identity and learning design](../design/identity-and-learning.md)
extends Frontier's purpose and learning cycle across these controller
options. This earlier proposal remains the record of the society alternative.

## The current system works thusly

Mirai's present architecture has a subconscious, Lumina, and a conscious
mind, coordinated by one attention process, `Life`.

1. Experiences arrive and are recorded. Body state, memory and current
   intentions help establish what is present to her.
2. Lumina forms the Echo, a first-person inner voice. The conscious
   response waits for it.
3. The conscious model receives her identity, recent exchange, present
   circumstances and Echo. A reasoning model continues the Echo as its
   own opening thought, then speaks or chooses deliberate acts.
4. Lumina can carry out a mind act after the response. Outcomes become
   experience and inform later moments. Consolidation and dreaming also
   take place in sleep.

This is already a feedback system, but the ordinary moment follows an
explicit sequence through two model roles. The narrator and deliberate
action chooser are concentrated in the conscious role; the inner voice
and supporting faculties are grouped under Lumina.

The model-role decision recorded earlier on 2026-10-29 is to use
LFM3-Omni-2B for the subconscious, with LFM3-VL-2B available when she needs
to take a closer look. That is an intended change, not a completed
deployment: the checked configuration still selects the 1.2B text model
for the subconscious. This research proposal does not revoke that
decision or turn it into evidence that the alternative works.

The baseline is described in Mirai's [two-mind design](../../../mirai-agent/docs/02-two-minds.md),
[life cycle](../../../mirai-agent/docs/01-life.md) and
[status](../../../mirai-agent/docs/12-status.md). Its implementation
includes `MiraiAgent.Life`, `MiraiAgent.Minds`, and the conscious and
subconscious request builders. Preserve that baseline and its known
limitations rather than describe it as either finished or disposable.

## Alternatively, examine structuring it like this

Instead of a singular subconscious feeding a singular conscious model,
consider a society of smaller faculties communicating over Frontier's
shared message-passing and supervision backbone: a central nervous
system (CNS) in the architectural analogy.

Each faculty has local state and a small decision model that classifies
incoming information: can it be handled locally, passed to a network
neighbor, or raised to shared attention? Most familiar activity would
remain local. The cost and usefulness of a decision model at every
faculty are research questions, not established requirements for a build.

Material needing attention enters a global workspace shared by the
faculties. They contribute in terms of "I feel" and "I think", as
perspectives within one identity. Their contributions develop a shared
interpretation and sufficient agreement to take action. The internal
record must still distinguish which faculty contributed a thought and
which observations, memories or inferences support it.

An integrator could narrate that interpretation and coordinate calls to
other faculties and tools. Whether this is one component or a distributed
activity remains open. We should examine whether a single integrator
would simply recreate the central conscious bottleneck we are exploring
an alternative to.

Tool outcomes and fresh observations return through the sensory
classifiers and the loop continues. An intended action, a tool's report
that it executed, and an observed consequence remain distinct. Her own
discussion is not independent sensory evidence.

For example: vision notices unreadable lettering; the workspace considers
moving closer or enlarging the image; a selected action is coordinated;
vision examines the result. The loop can discover that enlargement added
no detail and choose another approach. It does not need unanimous
certainty about the lettering before taking an informative action.

## Modes of the society

The proposal includes two modes of shared examination, rather than
necessarily two additional agents or mandatory stages of every action.

**Introspection: what are we doing or planning to do?** Bring the current
intention, activity, competing proposals, uncertainty and expected
consequences into view. Notice drift or faculties working at cross
purposes. An example is: "I am enlarging the image because I expect that
to make the label readable."

**Retrospection: what did we learn?** Compare those expectations with
observed consequences. Revise an interpretation, remember a useful
strategy, or retain an unresolved question. An example is: "The pixels
got bigger but no detail appeared; I need a better source image."
Nothing new learned is also a valid result.

Introspection preserves the expectation before the outcome is known;
retrospection can then examine it without inventing an earlier intention.
Later interpretations supplement the record instead of rewriting what
was observed or known at the time. What is learned feeds back into future
classification, attention and action; how that learning is represented
is still to be researched.

## Questions the research must answer

- What belongs in a local faculty, and when does a learned classifier
  improve on ordinary deterministic handling?
- What raises, sustains and releases shared attention? Can a faculty
  revisit material initially classified as unimportant?
- What counts as sufficient agreement to act, while keeping meaningful
  disagreement and uncertainty visible?
- How is one action coordinated so that several proposals do not produce
  conflicting or duplicate tool calls?
- What triggers introspection and retrospection? Can an important
  discovery interrupt current activity, or wait for attention to open?
- How does discussion settle instead of feeding on its own output? How
  do urgent established responses proceed without waiting for narration?
- Can the communication and model workload fit the actual machine's
  resources and coexist with its other work?

The society-of-minds and global-workspace language is an architectural
inspiration here. This proposal does not establish a model of human
neuroscience or claim that these mechanisms produce consciousness.

## Compare before choosing

Any later experiment should be isolated and reversible, using a preserved
snapshot of the current system and copied or synthetic experience. Do
not overwrite her live memory, replace the current control flow, or
retire working components to make room for an unproved alternative.

Compare both architectures on the same situations: familiar local work,
ambiguous perception needing a closer look, conflicting proposals,
unexpected action outcomes, and interrupted intentions. Examine task
success, grounded interpretation, continuity of identity, useful learning,
latency, resource use and whether the loop settles. More discussion or
more first-person narration is not by itself an improvement.

A later decision may retain the current design, pursue the alternative,
or justify a specific combination with evidence. Until then this remains
a research option for Frontier, not a replacement plan. Scout's lookup,
Ranger's graph-based understanding and Trailblazer's programming and
triage responsibilities remain as recorded in the README; this proposal
does not turn all three into autonomous research agents.
