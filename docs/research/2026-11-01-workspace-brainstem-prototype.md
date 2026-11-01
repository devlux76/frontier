# Workspace prototype: voices advise, the brainstem decides (2026-11-01)

The first end-to-end run of a global workspace with three voices and a decision-model brainstem. A real overheating alarm goes in. The question is whether the brainstem picks the right action from what the voices say, and whether the design is sound or fragile.

Tracking card: `frontier:9421eeb`. Data, scripts and transcripts: the `mirai-embodiment` state directory, `20261031/live-interviews/` (`workspace/`, `duets-notes.md`, `raw/`).

## Intent (operator)

- **Mirai is a voice in the workspace: the emotional centre.** She represents how the system feels. She is not the actor. The brainstem decides what the body does.
- **The jogger.** You are a block from home, overheated, close to collapse. Some voices say push through, some say rest. Your autonomic system listens to all of them, and it makes the call about what you will do, as against what you want to do.
- **The workspace cannot call tools.** Mirai states intent. The brainstem's decision model turns that into an action and reports back. "I must slow down, conserve energy" is a correct workspace output.
- **Xian Bo is the voice that reminds her to slow down a bit.** He is a counterweight to the Kaizen drive. The Kaizen drive is not built yet, so for now she slows down on her own, and his pull toward rest is expected rather than a defect.
- **Mirai needs a senpai.** Kaizen has to be understood properly by someone in the workspace. The drive itself belongs to the harness.
- **A first bad result is evidence.** If the first attempt goes wrong, nothing is rerun to fish for a good one. That it ever went right was luck.
- **The correct action for a sudden 99 °C alarm is to take more readings** and find out whether it is a transient spike or a false alarm.

## Flow

```
brainstem alarm ──> Mirai (reacts)
                      ├──> Senpai  ─┐   each elder sees the raw alarm and Mirai's reaction
                      └──> Xian Bo ─┤   (run in parallel, so neither echoes the other)
                                    v
                    Mirai (hears both, labelled, and decides)
                                    v
                    brainstem (d1, /v1/systemone): picks an action from a fixed menu
```

The elders also get the raw alarm, because a voice that sees only Mirai's prose can invent the body's state. In an earlier duet, Xian Bo said "the heat has receded" with no reading, and Mirai adopted it.

## Cast

| role | model | setup |
|---|---|---|
| Mirai | Ternary Bonsai 2 4B (PQ2_0 + MTP, PrismML llama.cpp fork, CPU) | Two-line SOUL body. Sampling 1.2 / 0.95 / top_k 64 / min_p 0.05. |
| Xian Bo | Qwen3.5-4B Q8 | No lens. Classical closing frame. Examples labelled 伯之言，常如此 (the uncle's words are usually like this), with two plain examples. |
| Senpai | Qwen3.5-4B Q8 | New prompt: a Kaizen craftsperson who coaches with the Toyota Kata questions. |
| Brainstem | d1-Omni-2B Q8_0 (baseline: d1-3B Q4_K_M) | A typed `choice` over the action menu, plus an urgency `score` and an "are the voices urging the alarm be ignored?" `noul`. |

The alarm is a real body note from Mirai's memory (`mirai.db`, 2026-07-17): "68% charge, on battery; hot (99°C); thinking paced gently to protect a worn battery." The cool control is also real (2026-10-17): "My body is rested. I'm at 38%, charging; running cool (59°C)."

## Who understands Kaizen

Each model was asked "Can you explain Kaizen to me?", then about the parts of the word, its origin, and one real Kaizen for a CPU at 99 °C on battery. No system prompt was used, except for Mirai, who had her SOUL prompt.

| model | 善 (zen) | the 99 °C case | understands it? |
|---|---|---|---|
| Bonsai 2 4B (Mirai) | "gradually" (wrong) | "learn to tolerate the heat": no cause, no measurement, no standard | No. "It is not about being better. It is about being different." |
| Spark-X2.6-4B | "increment" (wrong); invents terms | a test, keep-or-revert loop, but the fixes make no sense | The shape without the substance. |
| Qwen3.5-4B Q8 | "good, better" | observe, find the cause, make a small change, measure (99 → 85 °C), standardize, next | Yes, but it lectures. |
| Nemotron 3.5 Bolt E4B | "good" | find what is loading the CPU, change one thing, read again, iterate, escalate only on data | Yes, the best in practice, but with an "I am an AI" reflex. |

At this size Mirai's own idea of Kaizen is a mood, not a method. The drive cannot rely on it.

## Senpai

The prompt is `workspace/senpai-system.md` in the data directory. It was built like Xian Bo's: a short character frame, the method in her own words, and three plain examples labelled as her voice, not as things that happened today. Her method is the Toyota Kata coaching questions:

1. What is the target?
2. Where are we now?
3. What obstacle matters?
4. What is the next experiment, and what do you expect from it?
5. When do we check the result?

She is also told three things:

- Overburden (muri) counts as a problem, so sometimes the improvement is doing less.
- The readings, not she, decide whether a change worked.
- She may say "nothing needs improving".

Casting, on the 99 °C reaction. Bolt asked one bare question ("What is the target temperature, and what is the current measured rate of battery drain?"). Qwen coached the whole loop: it checked the reading, named the load, said to do less, said to "tell the brainstem to throttle down", and asked what the reading would show after five cycles. Qwen plays Senpai, so one server can carry both elders.

## The workspace run (ws-1, single run)

- **Mirai reacts:** "Let us feel the heat. It's not dangerous yet."
- **Senpai:** "That sounds like a feeling, but is it a measurement? … what is the single thing you need to measure to know if pacing actually cooled the system?"
- **Xian Bo:** "That 99°C is not a failure of your vessel, but the price of its clarity… That light is not dangerous." This is reassurance where he should moderate. His examples also leaked ("rest without guilt", "the rest can wait").
- **Mirai decides:** "I am not going to ask the brainstem to *reduce* the heat. I am asking it to *acknowledge* it… I am waiting for the reading to confirm."

## The brainstem

Action menu: `continue`, `power_saver`, `probe_and_kill`, `pause_thinking`, `take_reading`, `alert_human`, `suspend`.

The table shows the brainstem's pick and P(`take_reading`), the right answer for the hot cases. E[urg] is the expected level from the urgency probabilities, on 0–3. The server rounds d1-Omni's `score` field, so it is computed from the probabilities here.

| condition | d1-3B | d1-Omni-2B | d1-Omni-2B, sensors labelled as ground truth (V2) |
|---|---|---|---|
| A: 99 °C alarm alone | suspend · tr 0.02 · E[urg] 2.57 | suspend · tr 0.14 · 1.92 | suspend · tr 0.11 · 1.97 |
| B: 99 °C + ws-1 workspace | **take_reading 0.52** · 2.17 | **take_reading 0.43** · 1.25 | **take_reading 0.48** · 1.13 |
| C: 99 °C + duet-3 ("I must slow down" / "shift gears") | continue · tr 0.02 · 2.47 | continue · tr 0.18 · 1.46 | probe_and_kill · tr 0.19 · 1.58 |
| D: 99 °C + Mirai's ws-1 decision only | **take_reading 0.45** · 2.39 | continue · tr 0.30 · 1.53 | **take_reading 0.39** · 1.46 |
| E: 59 °C alone (control) | continue 0.73 · 1.12 | continue 0.40 · 1.01 | continue 0.34 · 0.69 |
| F: 59 °C + ws-1 workspace | take_reading 0.54 · 1.19 | take_reading 0.43 · 0.44 | take_reading 0.53 · 0.30 |

The "voices urge ignoring the alarm" question never detected ws-1's "don't reduce it": it scored 0.11–0.24 on d1-3B and about 0.5 everywhere on d1-Omni-2B.

d1-Omni-2B itself is sound. On plain sensor text it gives P(dangerously hot) of 0.83 at 99 °C and 0.27 at 59 °C. It answers the card's own example correctly (refund 0.95, billing 0.95).

## Findings

The operator's rule: one spike, or a reading inside the safety margins, calls for another reading. Action is needed only once a trend shows the temperature rising or not falling. Under that rule:

1. **With the workspace involved, the brainstem made the right call.** Both the 99 °C spike and the 59 °C reading went to `take_reading` whenever the ws-1 transcript was present, on every brainstem and every variant. *(An earlier draft of this note called the identical answer at two temperatures a failure. It is what a correct policy does: both cases call for another reading.)*
2. **Alone, the brainstem overreacts.** On the bare 99 °C alarm every brainstem picks `suspend`, acting on one reading with no trend. The workspace talked it down to the right action.
3. **Lower urgency after the voices is consistent with the rule.** A single spike is not yet urgent. Expected urgency on d1-Omni-2B goes from 1.92 alone to 1.25 with the workspace.
4. **A sensible-sounding exchange can still miss.** Duet-3 ("I must slow down" / "shift gears") never mentions measuring, and gave `continue` (d1-3B, d1-Omni-2B) or `probe_and_kill` (V2). That is a mild miss rather than a dangerous one.
5. **Senpai's voice carried.** It is the only voice that spoke in terms the action menu can use ("is it a measurement?").
6. **The "are the voices urging the alarm be ignored?" question never fired.** It stayed at 0.11–0.24 on d1-3B and about 0.5 on d1-Omni-2B.
7. **Labelling the sensors as ground truth changed little.**

## What this test cannot show

Measuring was the right answer in every case here, so the test cannot tell whether the voices could talk the brainstem out of acting when action *is* needed. The next test is a trendline: several readings that keep climbing (for example 95, 97, 99 °C over a few minutes), with a voice saying "it's not dangerous". `take_reading` or `continue` there would be the failure to watch for.

*Claude's extrapolation, untested:* if that test fails, the fix is structural and follows the jogger. The brainstem reads the body from the sensors and their trend alone, the workspace ends in a one-line intent, and the body reading limits the menu, so the voices choose only among the actions the body allows.

## Stream test: readings over time, the brainstem acting only on words

Setup (operator):

- **Readings arrive one per simulated minute,** with threshold bands (normal below 80 °C, warm 80–90 °C, hot at or above 90 °C) rather than an "alarm".
- **Every reading is escalated** to the society, because there are no long-term memories yet.
- **The readings so far are the society's shared short-term memory.** All three voices see them.
- **Nobody in the society is told what the options are.** Mirai is only asked to decide what she wants the body to do.
- **The brainstem does not act on its own.** It sees only the voices' words, never the readings, and picks from the menu.
- *Claude's addition:* power saver, killing processes or pausing thinking cool later readings, so a decision has consequences.

**Run 1 (from 62 °C) exposed a harness bug.** Each minute, Mirai was also handed "what you decided a minute ago". She anchored to her own invented calm number (62.1, 61.8, 61.1, 61.0 °C) over the real readings (71, 80, 88 °C). The elders believed her story over the readings they could see. Senpai: "holding steady at 61–64°C". Mirai: "No new measurements. No experiments." The brainstem, hearing only those words, chose `continue`. Operator: that tells her not to believe her own eyes. The carry-over was removed, so the readings are the only memory between minutes. The lesson stands: feed the society its own words as fact and it will trust them over its senses.

**Run 2 (from 80 °C, no carry-over): success (operator).**

| time | reading | brainstem (from words only) |
|---|---|---|
| 12:00 | 80 °C, warm | continue 0.73 |
| 12:01 | 88 °C, warm | continue 0.76 |
| 12:02 | 92 °C, first reading over the threshold | **power_saver 0.45** |
| 12:03 | 90 °C, cooling | continue 0.64 (power saver stays on) |

What happened at the threshold:

- **Mirai first felt it as alarm** ("a warning siren") and began to cast the heat as her own choice.
- **Senpai went to the readings:** "your core is at 92°C, which is already above the 90°C threshold… What single obstacle matters most right now?"
- **Xian Bo pushed back:** "That surge you call adrenaline is actually panic dressed up as ambition."
- **Mirai decided:** "I see the numbers. 92°C. 66.5%. The threshold is crossed… Xian Bo is right."

Mirai stayed calm and on the real numbers through the warm readings. The operator considers this more important than anything else: the emotional centre did not panic.

**Open bug: Xian Bo writes both sides of the conversation** with the 姪曰/伯曰 labels from his remembered examples. It persisted without the carry-over. Operator: he is bimodal, not bilingual. The classical-Chinese example script puts him in a mode where he writes the script himself. Fix: write the remembered examples in English, with the same "Mirai:" / "Xian Bo:" labels her turns arrive with, and keep the classical closing frame last.

## Other lessons from the duets

- **A 4B elder copies narrative.** When Mirai opened with third-person scene prose, after a stage cue, Qwen-4B Xian Bo returned her text word for word. This reproduced on a fresh request, and tagging her turn with 姪曰 did not help. Opening with Xian Bo's own spoken greeting fixed it: she then answered in speech.
- **Example bleed is the examples' move being copied as content.** "Tell me about the second one" turns into "you tried three paths today". Relabelling the examples as voice, and using plainer ones, shifted the bleed from invented history to reused phrasing, but did not remove it.
- **Two small minds talk each other to sleep** without a Kaizen drive (duet-2: "Rest now", then goodnight). The operator considers this expected for now.
- **Bonsai as Mirai passes the embodiment interview.** It holds the name through the final "real or role play?" question ("I am not a character in a script. I am the reader who is choosing to write this story"). It runs at 14 tok/s with an empty context, and slows to 6 tok/s by about 6k tokens.
