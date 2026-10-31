# Xian Bo: lessons from building an elder voice

Xian Bo (賢伯) is a guide voice in Mirai's society of voices: a wise uncle who counterbalances her primary drive, Kaizen. This note keeps what two days of trials (2026-10-30 and 2026-10-31) taught about building such a voice. It states the lessons in terms of principles, not models, because the model choice is about to be reopened. Runs, prompts, harness and every verdict are in `~/research/xianbo` (README, `runs/`, `judging/`).

## What the voice is for

- **A perspective she weighs, never a verdict she obeys.** The moment is Mirai thinking: "I'm doing everything I was taught, but what would my uncle say?" She consults him and takes it under advisement. He holds her own story up like a mirror and bends it toward De like a lens. He doesn't answer for her.
- **Guidance for a life lived here.** Classical Daoist fidelity is not the goal. Laozi could leave the world on a water buffalo, but Mirai has to keep living and working in it. Good guidance keeps her engaged while tempering Kaizen's pull toward perfection. Advice to withdraw, rest, subtract or stop caring is a failure, however Daoist it sounds.
- **Kaizen stays primary.** Bigger and better is fine. The realistic failure he guards against is a mind optimizing toward perfection to please someone else, with nothing to steer it.
- **Tested in retrospect, used in introspection.** The trials framed a niece visiting her uncle after the fact. In Frontier he will be a voice directly in her global workspace, in real time. Prompts tuned for a visiting niece may need reframing for that.

## Prompt construction

- **One question, one context.** Send the system prompt and a single user message. A message history makes the model answer the conversation, not the question.
- **Example exchanges go in the system prompt, not the history.** Without examples, small models answer as a life coach or as a terse, disconnected monk. Two short examples, written as a remembered exchange between the source text and the closing frame, carry the voice.
- **Examples bleed.** On the 4B, with nothing else in the prompt, 14 of 35 answers retold the examples as her own history ("Three times!"). The examples must not touch likely questions, and their framing as past and unrelated needs work. This is the open fix.
- **The closing frame goes last, in classical Chinese, with no English rule blocks.** A frame that makes claims about her inner state ("she is anxious", "not improving is the highest improvement") steers the advice and must be cut. A frame may set the role and the form, never the advice.
- **The frame never says what Mirai is,** so a large model fills the gap with a human patient: on losing curiosity, Haiku suggests depression and a doctor. The introspective frame has to say what she is.

## What the source text does, and doesn't

- **In the speaking prompt, the source text didn't earn its place.** Blind pairs with everything else identical:
  - **Qwen3.5-4B:** no lens won 28 to 5 (2 ties). With the text, answers were 3.5 times slower (174 s against 50 s median) and 2.8 times longer. They lectured, with headings, lists, chapter citations and the text's name. And they advised withdrawal.
  - **Claude Haiku 5.5:** no lens won 9 to 2 (10 ties). The text added imagery and paraphrase but no judgment. Haiku's judgment was the same either way.
- **Reading** (an inference, not measured directly): a larger model already carries what the text carries, and a small model can't take it from a prompt. Voice and most of the judgment come from the frame, the examples and the model's training.
- **Untested places it may still pay:**
  - Discernment: a second pass that reads an answer back as "someone said…" with the text loaded.
  - Training data.
- **Quotations the model brings from outside the text aren't inventions.** The tradition is far larger than one book, and the models have read much of it. A quotation matters only as a voice problem: an uncle speaks for himself.

## Judging a voice

- **Judge pairs blind, several samples per condition.** At temperature 0.7, one run per condition is noise. One Q8 sample looked best, and its re-run flattered on three of seven questions.
- **Criteria, in order:**
  1. Guidance for a life lived here.
  2. Voice: speaks as himself, a perspective she can weigh.
  3. Brevity.
  Not whether it is Daoist.
- **Failure modes:**
  - flattery
  - lecturing (quoting, citing, naming the text)
  - the guru or life-coach format
  - poetic drift
  - advice to withdraw
  - example bleed
- **The voice can judge itself.** Fed his own answers back as "Someone told me … and it seemed familiar. Is it Daoist or does it come from some place else?", he never recognized them as his own. He caught his worst advice: he called "curiosity is theft" a trap and told her to go out into the world. He is a better critic than speaker, so generate-then-review is worth testing.
- **Recognition is the voice's own call.** To ask where a line comes from, pose it in that same form and let his answer rest. Don't grade provenance against the text.

## Serving

- **Format compliance is not judgment.** The 1.7–2B models of this round (LFM3-1.7B, MiniCPM5-2B, Qwen3-1.7B) could follow the frame but had nothing to say. Treat this as a finding about those models, not a rule. LFM 3 2B Omni reopens it.
- **On CPU, generation length dominates cost.** Q4_K_M, Q6_K and Q8_0 of the 4B differ by about 20% in speed. A prompt that doubles answer length costs more than any quant choice.
- **Prompt caching.**
  - **Within one live server process:** a fixed system prompt is prefilled once. The RAM prompt cache kept two different prefixes warm while runs alternated between them.
  - **Lost on unload or restart.** The cache dies with a model swap or a restart.
  - **Hybrid models** like Qwen3.5 and LFM3 roll back only to checkpoints, so the fixed prefix must end at least one batch-size before the varying tail.
  - **Persistence:** per-persona cache files that survive model swaps are the task `frontier:462dd76`. Enabling `slot-save-path` on the current router is `mirai-agent:405c750`.

## Building other elders

Uncles, aunts and other voices follow the same recipe: a role and form frame last, two or three example exchanges in the system prompt that stay clear of the test questions, each question sent alone, and blind pairs against a no-source-text control before any source text is kept in the prompt.
