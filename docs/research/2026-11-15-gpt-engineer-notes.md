# gpt-engineer: what is worth taking (2026-11-15)

Source: AntonOsika/gpt-engineer (archived 2025-05; the CLI ancestor of Lovable), read from a shallow clone of its last state. The clone was deleted after reading. These notes are what we keep.

## What it is
A one-shot "prompt to codebase" CLI (`gpte`), later given an `improve` mode for existing code. LangChain chat wrapper, a handful of plain-text preprompts, a regex parser that turns the reply into files or unified diffs, an optional run.sh execution step, and a small benchmark runner. About 5.7k lines of Python. The loop is simple. The value is in its edit-application mistakes and in a few small mechanisms we lack.

## 1. Preprompts as editable files, composed by placeholder
`gpt_engineer/preprompts/{roadmap,generate,improve,file_format,file_format_diff,clarify,entrypoint,philosophy}` are plain text. `steps.py:setup_sys_prompt*` concatenates them: `roadmap + generate.replace("FILE_FORMAT", file_format) + "Useful to know:" + philosophy`. Generate and improve differ only in the format block. `--use-custom-preprompts` copies the set into the project so it can be edited per repo (`applications/cli/main.py:get_preprompts_path`).
**For us:** Trailblazer: keep the edit-format instructions in a separate slot from the task instructions, so a format change (ranges or SEARCH/REPLACE) is one swapped block. Compare variants against the same benchmark tasks instead of editing one monolithic prompt.

## 2. Clarify, or assume out loud
`preprompts/clarify` plus `tools/custom_steps.py:clarified_gen`: the model either asks one question or says the sentinel "Nothing to clarify". If the human skips a question, the model is told "Make your own assumptions and state them explicitly before starting". The clarify transcript, minus its priming message, then becomes the prefix of the generate call.
**For us:** Frontier: this is the ask-or-assume step at the brainstem/workspace boundary. When nobody answers, the assumption is written down rather than silently filled in, which is the gap-filling failure in our 10-31 notes. Trailblazer: the report could carry a "stated assumptions" field so a reviewer checks them first.

## 3. Line-numbered source in, diffs out (a warning, not a model)
`core/files_dict.py:to_chat` sends every file with a line number prefixed to each line. The prompt (`file_format_diff`) then has to shout "NEVER TRANSFER THESE LINE NUMBERS" and "single diff chunk per file". Models copied line numbers and emitted several diffs per file. The parser (`chat_to_files.py:parse_diffs`) keeps only the first diff for each file and needs a 3 s regex timeout against catastrophic backtracking (`diff_timeout`).
**For us:** Trailblazer already does better: read-backed ranges with SHA and `first`/`last` quotes. This confirms why. If the model sees numbers, give it a format that uses them, rather than one that forbids them.

## 4. Fuzzy hunk repair: a list of silent failures
`core/diff.py:Hunk.validate_and_correct` matches lines with `count_ratio`, a bag-of-characters overlap that ignores case and spaces (threshold 0.9). Two different lines with the same characters count as equal. When a hunk diverges, it compares a 10-line forward block three ways: as is, with the source line inserted ("model skipped a line") and with the hunk line dropped ("model hallucinated a line"). It keeps whichever scores best. Defects found by reading:
- `check_start_line` has no `return`, so every hunk searches the whole file for its first line. With several matches it takes the first one, with only a log warning.
- Any line containing `#` that fails to match is relabelled as an ADD. This is a Python-comment heuristic that breaks other languages and can duplicate code.
- `Diff.is_new_file()` is true if any hunk has no retained or removed lines. A pure-insertion hunk on an existing file is therefore treated as a new file, skips validation, and `apply_diffs` overwrites the whole file with the added lines. Silent data loss.
- `Diff.validate_and_correct` calls `self.hunks.remove(hunk)` while iterating over `self.hunks`, so the hunk after a bad one is skipped.
**For us:** Trailblazer: our rule is never to guess between repeated matches and never to repair automatically. Keep it. One idea is worth taking as a diagnostic, not a repair: the forward-block test that tells "you skipped a line" from "you added a line that is not in the source". It would make the miss feedback sharper than "did you mean these lines". Add the pure-insertion-on-existing-file case to our edit tests; it is the bug they shipped.

## 5. Salvage what applies, ask again only for the failures
`steps.py:_improve_loop`/`salvage_correct_hunks`: valid hunks are applied immediately, and the failure messages go back with "Only rewrite the problematic diffs". There are at most 2 refinement rounds (`core/default/constants.py:MAX_EDIT_REFINEMENT_STEPS`).
**For us:** Trailblazer's in-conversation correction (two corrections, then `edit_repair_exhausted`) is the same budget with stricter semantics: the batch is atomic, not partially applied. Partial apply is what let item 4 corrupt files unnoticed. Not worth changing.

## 6. Derive a run command when none is given; feed failures back
`steps.py:gen_entrypoint` makes a separate model call that sees the codebase and writes `run.sh` ("Do not install globally. Do not use sudo. No placeholders"). `tools/custom_steps.py:self_heal` runs it, and on a non-zero exit (2 counts as success) sends the original spec plus the full stdout/stderr, untruncated, back through `improve_fn`. `core/project_config.py` adds a per-repo `gpt-engineer.toml` with `[run] build/test/lint/format` commands.
**For us:** Trailblazer: a task without `--check` ends `unverified`. Two cheap fixes: (a) a per-repo declared `test`/`build` command in Trailblazer config, used as the default check; (b) when none exists, a proposed check command that the planner or reviewer approves before it counts. Never auto-trust a model-written check: their self-heal can "pass" by making the program exit 2. Truncate failure output; they never did.

## 7. Benchmark: tasks as files + prompt + command + named assertions
`benchmark/types.py`: `Task{initial_code, prompt, command, assertions: {name: Callable[[Assertable], bool]}}`. `Assertable` carries files, env, process, stdout and stderr. `run.py` calls only `agent.improve(initial_code, prompt)`; generation is improve with empty initial code. The score is the fraction of assertions that pass per task, plus the count of fully solved tasks. Results are exported to YAML together with the config that produced them. Loaders: APPS, MBPP, gptme (`benchmark/benchmarks/*/load.py`); `default_bench_config.toml` picks index ranges.
**For us:** SWE-bench lab and Trailblazer §10: named per-task assertions give partial credit and show which property failed, not just pass/fail. One `improve`-shaped interface makes our harness and the baselines interchangeable. Always store the config next to the result.

## 8. Record/replay of model replies
`--use_cache` installs a SQLite LLM cache keyed by prompt (`applications/cli/main.py`, `SQLiteCache(".langchain.db")`). Re-running a prompt replays the reply, which is deterministic and free.
**For us:** Trailblazer: case records already keep raw model bytes. Replaying them through the parser and apply path gives a hermetic regression corpus for edit-format changes, with no model running. This is the oracle that item 4's code never had.

## 9. Human as the model backend
`core/ai.py:ClipboardAI`: the prompt goes to the clipboard, a person pastes the reply, and the same parse/apply pipeline runs.
**For us:** Trailblazer: a "manual" model backend would let a frontier session play coder on a hard case through the exact local schema. That gives gold transcripts and a way to tell "the format is bad" from "the model is weak".

## 10. Outcome review attached to the run
`applications/cli/learning.py`/`collect.py`: after a run, consent-gated questions (did it run? perfect? useful? y/n/uncertain plus comments) are bundled with the prompt, config and logs into a versioned `Learning` record. This was the data flywheel the hosted product grew from.
**For us:** Frontier: the encounter record (identity-and-learning §3) should carry an explicit outcome verdict that can be "uncertain", rather than inferring one only from later consequences. A three-valued verdict is a better Bayesian update input than forcing yes/no.

## Also noted
- Memory layout: `.gpteng/memory/logs/` holds one append-only log per step (code gen, improve, diff errors, debug with console tee). It is moved to `logs_<timestamp>` at the start of each run (`core/default/disk_memory.py:archive_logs`). A separate diff-error log makes edit-failure rates countable.
- File selection: an editable TOML of the whole tree with every line commented out. The user uncomments lines, and the choice persists and is reused with `-s` (`applications/cli/file_selector.py`). Trailblazer's trail replaces this. The one useful part is persisting the selection as a reproducible case input.
- Git safety: before overwriting, files with uncommitted changes are staged so the user's work is recoverable, and a coloured diff plus a y/n prompt come before the write (`core/git.py`, `main.py:compare`). Our private worktrees are stronger.
- Consecutive same-role messages are collapsed before each call (`ai.py:_collapse_text_messages`), for backends that require alternating roles.
- Token use is logged per `step_name`, and cost only when the model name contains "gpt" (`core/token_usage.py`). Our usage ledger is ahead.
- The generate prompt says "if you are unsure, write a plausible implementation" and ends with the sentinel "this concludes a fully working implementation". The first line instructs the model to fill gaps; the sentinel is a cheap check for truncated output.

## Not worth taking
LangChain coupling, the CLI/typer shell, vision prompt plumbing, `black` auto-formatting of model output (`core/linting.py`), the fuzzy diff repair (item 4) as a mechanism, partial hunk application, and the regex chat-to-files parser.
