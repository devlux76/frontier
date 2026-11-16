# freeact: what is worth taking (2026-11-15)

Source: gradion-ai/freeact (upstream, active, ~131★, pushed 2026-07), read from its docs. Our unmodified fork was deleted in the GitHub cleanup; these notes are what we keep.

## What it is
A CodeAct agent (same family as smolagents): the model acts by writing code that runs in a stateful IPython kernel (`ipybox`), not by emitting one JSON tool call per inference. Python, shell (`!cmd`, `%%bash`) and tool calls mix in one action. Pydantic-AI for models, Anthropic `sandbox-runtime` for OS-level sandboxing. Nothing in the loop itself is new; four mechanisms are.

## 1. MCP "code mode" (typed tool APIs)
MCP server schemas are compiled into importable Python modules (`mcptools/<server>/<tool>.py`, `run(Params(...))`). The agent reads the typed signature, then composes several tool calls plus processing in one action. Fewer LLM round trips; no intermediate tool output pulled into context. (Credit: Cloudflare "Code Mode" post.)
**For us:** same direction as our "no code in JSON" rule for local models. A generated typed façade over MCP tools is a cheap win for small local models that are bad at multi-turn JSON tool use.

## 2. Output parsers: the agent as toolsmith
Most MCP tools have no output schema (e.g. GitHub's MCP returns a JSON string). freeact has the agent call the tool with sample inputs, observe the output, and write a parser that adds `run_parsed()` returning a typed result. Persisted, so later actions can process output without first reading it into context.
**For us:** a concrete, testable "improve your own tools" loop. Fits the Trailblazer/planner verify step: a parser is accepted only if it round-trips observed outputs.

## 3. Saved code actions → tool library (interface / implementation split)
A successful action can be saved, parameterized, as `gentools/<category>/<tool>/{api.py, impl.py}`. `api.py` holds the signature, Pydantic models and docstring; `impl.py` is lazily imported. The agent tests it after saving. Discovery reads only `api.py`, so a library grows without growing the context.
**For us:** the api/impl split is the key idea: it is what keeps a skill library cheap to search. Maps onto Resonant's "capture experience as executable knowledge" and onto how our skills are already loaded (description first, body on demand).

## 4. Tool discovery
`basic`: browse categories → list tools. `hybrid`: BM25 + vector fusion over tool APIs (`pytools_search_tools`), configurable weights, file-watched index. Only task-relevant tool APIs are loaded.
**For us:** Trailblazer already does hybrid retrieval over code; tool APIs are just another corpus it could index.

## Also noted
- Approval is per sub-action: shell lines and tool calls inside a code action are intercepted and approved individually, with tiered permission rules, including from subagents. Matches the agent-guard idea at a finer grain.
- Kernel cwd is reset after every action (prints a notice), which removes a whole class of drift bugs.
- Sessions persist incrementally and double as eval/debug records.

## Not worth taking
The CLI/SDK shell, Pydantic-AI coupling, Gemini-embedding default.
