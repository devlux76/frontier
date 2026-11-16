# OpenAI Cookbook: what is worth taking (2026-11-15)

Source: openai/openai-cookbook via our fork devlux76/openai-cookbook (synced to upstream 2026-11-10), read from the repo at depth 1, sparse. The fork will be deleted; these notes are what we keep. Registry: 325 entries, 105 dated 2025-06 or later, newest 2026-11-03. Most of the repo is OpenAI-product how-to, which we skip.

## What it is
OpenAI's examples and articles repo for its API and agent SDK. The useful part for us is the newer material (2025-2026) on running agents under a budget, measuring them, and turning their traces into evals and harness changes. The code is OpenAI-specific; the patterns are not. The notebooks are not runnable offline except where they say so. Nothing here is a model-agnostic library.

## 1. Per-run spending controller
Give each agent run its own budget. Before every model request, count input tokens and reserve the worst-case output cost; after the response, record the actual cost and return the unused reserve. Refuse the next request if it would exceed the balance. Use exact fractions for money, not floats. (articles/per_run_spending_controller_responses_api.md)
**For us:** Frontier should hold a per-task token/time budget in the same reserve-then-settle shape, so a runaway local loop stops at a known cost. The eval lab can use it as a hard stop per run.

## 2. Prompt-cache discipline
Keep the prefix byte-stable: static tool definitions and system text first, per-request restrictions in request metadata, not in the prefix. Use a cache key to improve routing stickiness, and prefer gradual context truncation (retention ratio) over big chunk drops, because big drops bust the prefix every time. (examples/Prompt_Caching_201.ipynb; the notebook's header says it covers models before GPT-5.6.)
**For us:** same rule for our llama.cpp router: Trailblazer and Frontier prompts must put the fixed header first and vary only the tail, or the KV reuse disappears. Log cached-token counts in the eval lab to prove it.

## 3. Shrink the tool surface per step
Expose the full stable tool list but restrict each step to an allowed subset. Cap tool rounds, then make the final call with tool choice "none" so the model must answer. Return only the fields the next decision needs, not whole payloads. (examples/agent_optimization/optimizing_agents_for_cost_and_quality.ipynb; code in live_api.py of the same folder)
**For us:** small local models degrade with a large, ambiguous tool set. Frontier should pass a per-step subset and Trailblazer should return slim results. The "force an answer after N rounds" rule is cheap insurance.

## 4. Quality-gated optimization rounds
Measure a baseline on a fixed eval set; change one knob; accept the saving only if quality holds on the same set. Report each round as a delta against the previous round, not one before/after number. Keep a holdout set. The notebook is a dry run with mock data, so its numbers are illustrative only. (same notebook as #3)
**For us:** this is the eval lab's accept/reject loop for any harness change (router settings, prompt, tool subset). Use the round-by-round table as the format for lab reports.

## 5. Macro evals across many traces
Per-run checks catch single failures. Macro evals cluster the whole population of runs, using a compact structured summary per run (scenario, agent path, handoffs, tool calls, terminal state) as the document to cluster. Then read a Sankey-style chain: case type -> outcome -> finding, to see which scenario types produce which failure. (examples/partners/macro_evals_for_agentic_systems/macro_evals_for_agentic_systems.ipynb)
**For us:** the eval lab runs many coding-agent tasks; cluster their traces this way to find repeated failure patterns instead of reading runs one by one. Trailblazer's graph can supply the agent-path fields.

## 6. Traces -> evals -> harness change, as a loop
Export agent spans to a flat JSONL, attach human and model feedback to the same traces, generate an eval suite from the feedback, gate the current agent on it, and hand the evidence to a coding agent that proposes the next harness change. The harness is a versioned config (model settings, instructions, tool policy, promoted version id), so the optimizer targets the config, not just prompt wording. (examples/agents_sdk/agent_improvement_loop.ipynb)
**For us:** Frontier's agent config should be a versioned record like this, with a promoted-version id that eval results point at. Trailblazer does the implementing; the eval lab supplies the gate.

## 7. Analyze before you grade
Order the flywheel: analyze (annotate failing examples by hand and name the recurring failure modes), then measure (build graders only for those modes), then improve, then repeat. Expect new failure modes after each fix. (examples/evaluation/Building_resilient_prompts_using_an_evaluation_flywheel.md)
**For us:** before the eval lab writes a new grader, annotate a sample of real failed coding runs and name the failure class. This stops us grading things nobody cares about.

## 8. Fidelity ladder for the same test cases
Run one shared test set at three fidelities: direct tool calls with no model (free, deterministic, CI-able), a small application-owned function-calling loop, and the full harness. When results differ between levels, the gap locates the fault: tool, loop or harness. (examples/partners/harness_aware_plugin_evals/harness_aware_plugin_evals.ipynb)
**For us:** Frontier's tool layer gets level-1 tests in CI; the eval lab runs levels 2 and 3 against our local router. A failure at level 3 only is a harness bug, not a model bug.

## 9. Separate compaction from memory
Compaction carries the state of the current run forward when context fills; memory carries reusable lessons into future runs. They answer different questions and must not share a store. Keep the task's facts in workspace files with a machine-readable manifest, not in the prompt. (examples/agents_sdk/building_reliable_agents_memory_compaction.ipynb, comparison table in the notebook)
**For us:** Trailblazer's per-task working state (compaction) and Resonant's cross-task lessons (memory) should be separate writes. Give small models a manifest file to read rather than a long prompt. Note the notebook's warning: compaction can drop facts the next decision needs, so eval it.

## 10. Execution plans as living documents
An AGENTS.md pointer tells the agent when to write a plan. The plan file (PLANS.md) is a self-contained design doc: the reader has only the working tree and the plan, and the plan records progress, decisions and surprises as it goes. The user reviews the plan before the long run starts. (articles/codex_exec_plans.md)
**For us:** this matches "frontier writes exact plans; planner drives; Trailblazer implements". Adopt the self-contained-plan rule and the living progress log for planner tasks. Expect small local models to need shorter plans than the example.

## Also noted
- Crawl / walk / run staging for evals (examples/audio/voice_agent_evaluation.md, examples/Realtime_eval_guide.ipynb): synthetic input, then recorded input, then a simulated multi-turn user. Isolate one dimension per stage. Voice is out of scope for us; the staging order fits the eval lab.
- Output-cap warning: a reasoning model's output cap includes its hidden reasoning tokens, so an incomplete response can be a cap problem, not a model problem. Check for it before blaming the prompt.
- Partner notebooks (AWS, Databricks, Daytona, SchemaFlow) are OpenAI-platform integration demos; the design choices in them are covered by #5 and #8.

## Not worth taking
Responses API parameters and price tables (they expire; the notebooks say so themselves), ChatGPT admin and compliance APIs, Sora and image prompting guides, the gpt-oss hosting guides (we run our own router), Codex-specific workflow guides, and the Agents SDK code itself (we are not coupling to it). The security-scanner notebook (about 700 KB) was not read.
