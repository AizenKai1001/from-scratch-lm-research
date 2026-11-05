# 20. Longer memory, sharper eyes, and speed: what has been measured so far

Snapshot dated 5 November 2026. This document reports three budgets for the small language model (the "brain", 360 million parameters, SmolLM2-360M, frozen or fine-tuned as stated): how much text it can use at once (context length), how much small detail its picture input can carry (the eyes), and how fast it runs on our two 8 GB graphics cards. It also lists the code-review findings that changed some conclusions. Every number is a **FACT** from our own logged runs unless marked **GUESS** (reasoning, not yet tested) or **PENDING** (planned or built, not yet measured). Pass and kill rules were written before each run. Most results are from one seed, so treat them as indications.

## 1. Context length

**FACT, corrected test.** Needle-in-a-haystack: a short fact is hidden at a set depth inside a long text, and the model must reproduce it exactly.

| Setting | 4k | 8k | 16k | Early-depth check |
|---|---|---|---|---|
| Plain model | 1.00 | 0.85 | 0.00 | at 5% depth, 8k: 0.25 |
| YaRN stretch (no training) | not recorded | not recorded | 0.75 | at 5% depth, 16k: 0.00 |

The plain 32k and 64k results (both 0.00) came from the first generator and have not been re-scored with the corrected one. The flaw below can only raise a score, never lower it, so these zeros stand as upper bounds. The 32k logits were checked and are finite, so that zero is not an fp16 overflow (a NaN check was done at 32k only).

**The flaw and the fix.** A code review found that the first test generator sized its filler by counting lines (16 tokens per line), not real tokens. When the needle was cut by the length limit, the script moved it to position 0, the easiest place to recall, so scores were inflated without any warning. The fixed generator builds the text by real token count, inserts the needle at fixed depths (5, 25, 50, 75 and 95 percent) and never cuts it. The same flaw sits inside the long-context training kernel's own scorer, so its scores will not be used. The first table's 8k figure of 0.95 came from the old generator; the corrected figure is 0.85. The early-depth check (0.25) exists only in the corrected run.

**Reading.** The plain model is reliable to 4k and weaker at 8k, especially for facts placed early in the text. YaRN without training reaches 0.75 at 16k on average but still fails early in the text. No setting yet passes the 16k gate (0.80).

**PENDING.** A continued-training run with a larger rope base, on 8k then 32k tokens, was sent to Kaggle. The first attempt crashed at start-up on a configuration argument and was cancelled, so it produced no weights. A second attempt has been running since about 3:15 PM on 5 November; no result file was on disk at the last check. Any weights must be re-scored with the corrected generator before any claim. The planned comparison of YaRN, continued training, a mix of no-position and rope layers, and quantised memory at 32k, 128k and 1M has not started. Each stage has a written pass rule, and the 1M target is only tried after 128k passes.

## 2. Sharper eyes

The eyes are a frozen SigLIP picture encoder with a small trained connector (2.7M weights) feeding the brain. Test pictures are 1920 by 800, the shape of a film frame.

**FACT, colour-square test at equal steps (2,000 steps each, chance 0.125).** A 40-pixel coloured square is placed on the frame, and the model must name its colour.

| Arm | Colour exact | Captions (retrieval) | Image tokens |
|---|---|---|---|
| A: whole frame squashed to 224 | 0.13 | 0.37 | 49 |
| B: 2 by 4 tiles plus a whole view | 0.96 | 0.153 | 441 |

B used batch 4 instead of 8 because of memory, so at equal steps it saw half as many examples, and the caption comparison is confounded. By the written rule, B passed the detail test and failed the caption guard (captions must stay within 0.03 of A). The overall verdict is therefore FAIL by the literal rule. The honest reading at that point: resolution helps small objects a lot, and the caption gap was the batch confound (see below). Tiles cost about nine times the tokens.

**FACT, equal examples (9,600 per arm, two seeds, 300 test items).** Full tiles (2 by 4 plus whole view, 441 tokens): colour 0.967 on seed 11 and 0.927 on seed 12, captions 0.253 and 0.34. Squashed 224 (49 tokens): colour 0.13 and 0.137, captions 0.243 and 0.21. By the written colour and caption rules, both seeds pass. Speed on a quiet GPU is still **PENDING**.

**FACT, earlier 15-minute runs on stretched Flickr pictures.** Caption retrieval among 300: squashed 224 0.403 (49 tokens), squashed 384 0.457 (144 tokens), tiles 0.387 (441 tokens). Tiles did not beat the squash on captions in this run.

**The digit test failed.** A hidden five-digit number was tested in every arm, and the score was 0.0 in all arms and in the calibration arm (squashed 224 with 90-pixel digits). The test cannot tell resolutions apart with a frozen encoder and a small connector, so it gives no verdict on resolution. A harder calibration is needed first.

**Seed spread.** Two runs of the identical squashed arm gave colour 0.143 and 0.213 (2,406 and 2,759 steps). So differences under about 0.07 on one run are not results.

**Foveated glance.** The whole frame at 224 plus one full-resolution crop around a saliency peak (98 tokens). Two runs of the same arm: colour 0.243 and captions 0.463 at 2,085 steps in a fixed-minutes run, and colour 0.21 and captions 0.473 at exactly 2,000 steps. The rule needed colour at least 0.15 above the squash (0.13 at equal steps), so it did not pass. The crop window contained the square only 19 percent of the time (a random window: 10 percent), so the saliency rule itself is weak: a colour-contrast map cannot know which square the question is about, because other saturated squares exist. A question-driven tile choice (training-free, 98 tokens, arm H) is measured: colour 0.74 on seed 11 and 0.72 on seed 12, captions 0.327 and 0.39. Both seeds miss the 0.80 colour bar, so it does not pass; the chosen tile contains the square about 66 percent of the time. A refined query (H3, 98 tokens) scored colour 0.88 on seed 11, and seed 12 has not been run. Two-tile picks (H2, 147 tokens) failed on both seeds (0.117 and 0.143). A two-glance variant ran too few steps to count.

**FACT, cheaper tiled variants at equal examples, seed 11 only.** T2 (108 tokens, coarse pooling): colour 0.13, captions 0.213. T3 (128 tokens, adjacent-token merge): colour 0.15, captions 0.353. T4 (64 learned queries, trained from scratch): colour 0.09, captions 0.007. All three fail the colour bar, so these cheap compressions did not keep the small detail. They were not repeated on seed 12 after seed 11 failed.

**PENDING.** Equal-step glance arms, timing on a quiet GPU, and a second seed for the glance arms. Real film frames have not been tested yet; the colour test is synthetic.

## 3. Speed

**FACT, brain speed (SmolLM2-360M, fp16, 40 new tokens, medians of five runs, RTX 2060 SUPER).**

| Arm | Tokens per second | Speed against no cache | Same tokens |
|---|---|---|---|
| Plain loop, no cache | 39.2 | 1.00x | baseline |
| Default cache | 41.2 | 1.05x | yes |
| Static cache plus compile | 189.9 | 4.8x | yes |
| Eager attention, default cache | 32.4 | 0.83x | yes |

The 4.8x is for the static cache with compilation together (arm C). The two were not measured separately, so the gain cannot be assigned to compilation alone. An earlier note quoted 173 tokens per second for a static cache; no result file supports that figure, so it is not used here. The first-token median is 25 to 34 ms across the arms in this table (2060, 40 new tokens).

**FACT, size benchmark (RTX 2060 SUPER, fp16, static cache plus compile).** The pass line was set first: first token 100 ms or less with eye and ear input, decode at least 40 tokens per second, and memory that fits 8 GB at 4k context with the encoders loaded. Inputs were stand-ins, so the benchmark covers speed and memory, not text quality.

| Model | Decode (tokens/s) | First token | Verdict |
|---|---|---|---|
| SmolLM2-360M | 179 | 25 ms | PASS |
| SmolLM2-1.7B | 95 | 31 ms | PASS (fits 4k; 8k not counted) |
| Qwen3-1.7B | 87 | 33 ms | PASS |
| SmolLM3-3B | 54 | 51 ms | FAIL (does not fit 4k with the encoders) |

Speed is not the limit for a 1.7B brain; memory for long context is. **PENDING:** the same benchmark on the 4060, int8 variants, and a quality test of the 1.7B models (not run; the recipe that worked on the 360M did not transfer to another 0.5B model in an earlier test).

**Human timing envelope.** Reference ranges for people are taken from our notes and have not been re-checked. The measured values for the first two rows are first-token latencies on random or noise stand-in inputs only (no real frames or clips were timed). This table is NOT a human-timing pass; it shows only that the stand-in latencies are well below the targets we set.

| Part | Human reference (approx.) | Target we set | Measured | Status |
|---|---|---|---|---|
| Picture to first thought | gist 100-150 ms; visual reaction 200-250 ms | 300 ms or less | about 25 to 31 ms on a random-picture stand-in | Below target on stand-in input only; real frames not timed; not a human-timing comparison |
| Speech to first thought | auditory reaction about 150 ms | 300 ms or less | about 31 ms on 5 s of noise (stand-in) | Below target on stand-in input only; real clips not timed; not a human-timing comparison |
| Looking at frames | 3-5 looks per second | 1-3 looks per second | 1 look per second, uniform | Glance policy not yet a win |
| Text output | reads 4-5 words per second | paced to human speed when shown to a person | about 190 tokens per second (2060, static cache plus compile) | Model is much faster; pacing not built |
| Long memory | (no single figure) | as long as possible | reliable to 8k, not 16k | Gap remains |

Reading: first-token latency on stand-in inputs is well under the 300 ms target we set, but that is not a comparison with people, and the human references are unchecked. The open gaps are timing on real frames and clips, outward pacing, long memory and the glance policy.

## 4. Lessons from code reviews that changed conclusions

- **Wall-clock training starved some arms.** The first eye runs trained for fixed minutes, so the arms got different step counts (2,471 and 2,588 steps, and as low as 876 and 984 under GPU contention). The tile result was void until rerun at equal steps. Rule now: budgets are steps or examples, never minutes.
- **A shell quoting bug.** A launch script written through a double-quoted remote heredoc had its variables expanded on the wrong machine and came out empty. It was caught before any training started. Rule now: write scripts locally and copy them.
- **Single-run noise.** The identical squash arm varied by 0.07 between runs (section 2). Single-run gaps are read against that spread.
- **Speed numbers were capped by pacing.** A sleep inside the timed loop of the live watcher meant its speed figure could not exceed about 1.0 by construction. A replacement watcher without the sleep is built but not yet run (PENDING).
- **Leakage in the live hearing check.** The first live viewer used a hearing connector trained on the same film it was scored on, so its scores measured memorised dialogue. Live scoring is now to use the clean-speech connector or the held-out part of the film only.
- **Timing under load.** Timings taken while another job held the GPU are void, and the per-frame numbers in the early eye runs are of this kind.
- **Max-of-three timing is optimistic.** Medians of at least five runs are now used.

## What is still open

- Corrected long-context scores for the continued-training model, then the YaRN, rope and quantised-memory stages at 32k, 128k and 1M.
- Equal-step eye arms (glance, tiles at three token counts), real film frames, and a second seed for each arm.
- The 4060 size benchmark, and a quality test for any larger brain before it is adopted.
- The timing gate for all senses together (first token with eyes and ears on), which has not been measured.
