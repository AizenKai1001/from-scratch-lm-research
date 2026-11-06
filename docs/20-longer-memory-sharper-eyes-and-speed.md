# 20. Longer memory, sharper eyes, and speed: what has been measured so far

*Revised 6 November 2026. Replaces the 5 November version.

Snapshot dated 6 November 2026. This document reports three budgets for the small language model (the "brain", SmolLM2-360M, frozen or fine-tuned as stated): how much text it can use at once (context length), how much small detail its picture input can carry (the eyes), and how fast it runs on our two 8 GB graphics cards. Every number is a **FACT** read from our own logged result files unless marked **GUESS** (reasoning, not yet tested), **PENDING** (planned, built or running, not yet measured) or **INVALID** (a run that failed its own written rule). Pass and kill rules were written before each run. Most results come from one or two seeds, so treat them as indications.

## 1. Context length

**FACT, corrected test.** Needle-in-a-haystack: a short fact is hidden at a set depth inside a long text, and the model must reproduce it exactly. Each setting uses five depths (5, 25, 50, 75 and 95 percent of the text), with four items per depth.

| Setting | 4k | 8k | 16k | Early-depth check (5 percent depth) |
|---|---|---|---|---|
| Plain model (original weights) | 1.00 | 0.85 | 0.00 | 8k: 0.25; 16k: 0.00 |
| YaRN stretch (no training) | not run | not run | 0.75 | 16k: 0.00 |

No 32k or 64k score exists in our result files. Earlier notes listed zeros for those lengths; no file supports them, so they are withdrawn.

**The flaw and the fix.** A code review found that the first test generator sized its filler by counting lines (16 tokens per line), not real tokens. When the needle was cut by the length limit, the script moved it to position 0, the easiest place to recall, so scores were inflated without any warning. The corrected generator builds the text by real token count, inserts the needle at the fixed depths and never cuts it. The first table's 8k figure of 0.95 came from the old generator; the corrected figure is 0.85. The long-context training run has its own built-in check with the same flaw. It printed 0.875 at 8k for the original weights, and that number is not used.

**Reading.** The plain model is reliable at 4k. At 8k it scores 0.85 overall, but only 0.25 for facts placed in the first 5 percent of the text, so we do not claim reliability at 8k. At 16k the plain model scores 0.00, and YaRN stretching without training reaches 0.75 on average while still missing facts near the start (0.00 at 5 percent depth). No setting passes the 16k gate of 0.80.

**Trust note.** The needle script loads model weights without a strict check that every key was found, so a wrong checkpoint name would pass silently. The table above is for the original weights only. A key-audit script exists but has no result yet; it must pass before any trained checkpoint is scored.

**INVALID: continued training.** A continued-training run with a larger position base (rope base 1,000,000) was meant to lift the model to 16k and 32k tokens: 200 steps at 8k, then 100 at 16k, then 60 at 32k, under a limit of 180 minutes of training time. It stopped at 244 of 360 steps (200 of 200 at 8k, 44 of 100 at 16k, 0 of 60 at 32k) when the limit was reached after 181.5 minutes. No steps were skipped. The weights were saved but not scored, so the run gives no retrieval number, and it is **INVALID** by its own rule (schedule incomplete). The time limit was set from a step-time estimate that had not been measured. A first attempt had crashed after 60 seconds on a configuration argument, and its job then showed RUNNING for about 12.5 hours. A RUNNING status is not a result.

**PENDING: resume.** A resume (56 steps at 16k, then 60 at 32k) is written. Its script passed a CPU-only check: the model loaded with the new position setting, and attention and output matched the reference. No training was run in that check. The GPU job was refused twice because the weekly Kaggle GPU quota (30 hours) was used up. It is expected to take about 7 hours once the quota resets, and no retrieval number will be claimed until it is scored with the corrected generator and the key audit.

## 2. Sharper eyes

The eyes are a frozen SigLIP picture encoder with a small trained connector (about 2.7 million weights) that feeds the brain. Test pictures are 1,920 by 800 pixels, a widescreen shape. The test is synthetic: a 40-pixel coloured square (one of eight colours) is placed at a random position on a stretched Flickr8k photograph, and the model must name its colour. Chance is 0.125. Caption retrieval among 300 captions checks that the picture still carries the scene.

**FACT, equal steps (2,000 steps per arm; earlier runs; chance 0.125).**

| Arm | Image tokens | Colour exact | Captions |
|---|---|---|---|
| A: whole frame squashed to 224 | 49 | 0.13 | 0.37 |
| B: 2 by 4 tiles plus a whole view | 441 | 0.96 | 0.15 (batch 4, so half the examples) |
| F: whole frame plus one full-resolution crop | 98 | 0.21 | 0.47 |
| G: whole frame plus two full-resolution crops | 147 | 0.27 | 0.45 |

B passed the colour test but failed the caption guard. Its caption score is confounded, because it saw half as many examples (batch 4). The glance arms F and G did not reach the rule's line of the squashed arm plus 0.15 (0.28). Each glance arm has one seed at equal steps; a second seed is **PENDING**.

**FACT, equal examples (9,600 examples per arm, 300 test items; seeds 11 and 12 unless noted).**

| Arm | Image tokens | Colour, seed 11 | Colour, seed 12 | Captions, seed 11 / 12 |
|---|---|---|---|---|
| A: squashed 224 (comparator; 1,200 steps at batch 8) | 49 | 0.13 | 0.14 | 0.24 / 0.21 |
| B: 2 by 4 tiles plus whole view | 441 | 0.97 | 0.93 | 0.25 / 0.34 |
| H: whole view plus the tile the question points to | 98 | 0.74 | 0.72 | 0.33 / 0.39 |
| H3: as H, with a reworded question | 98 | 0.88 | 0.77 | 0.33 / 0.36 |
| H2: whole view plus the two tiles the question points to | 147 | 0.12 | 0.14 | 0.28 / 0.22 |
| I: whole view plus a trained tile selector | 98 | 0.96 | 0.95 | 0.31 / 0.38 |
| J: whole view plus trained selector and two tiles | 147 | 0.97 | running | 0.28 / not yet run |
| T2: coarse pooling of tiles (seed 11 only) | 108 | 0.13 | not run | 0.21 |
| T3: neighbouring tokens merged (seed 11 only) | 128 | 0.15 | not run | 0.35 |
| T4: 64 learned query tokens (seed 11 only) | 64 | 0.09 | not run | 0.01 |

- The squashed view (A) misses the small square on both seeds. Full tiles (B) read it on both seeds, but use about nine times the image tokens of A.
- The question-picked tile (H, 98 tokens, no training of the picker) reaches 0.72 to 0.74, about six times chance. On seed 11 the picked tile contained the square 66 percent of the time.
- The cheaper tile versions (T2, T3, T4) did not read the square on seed 11, so their seed 12 runs were not made.
- H3 (reworded question) reached 0.88 on seed 11 and 0.77 on seed 12, a mean of 0.83. Seed 12 is 0.03 below the per-seed bar of 0.80, inside the run-to-run spread, so H3 is not a pass.
- Two picked tiles (H2) fell to chance on both seeds, although it had more tokens than H. A code check found no fault. The cause is a **GUESS**: the caption and colour answers share one loss, and colour tokens may carry only a small share of the training signal.
- A trained selector (I) picks the tile from the question. On seed 11 its test accuracy was 0.93 and on seed 12 0.92; the correct tile was among its top two choices 98 percent of the time. The brain then read the colour at 0.96 and 0.95 with 98 image tokens. The selector was trained for a fixed 180 seconds of wall-clock time, so its step count (79,000 to 84,000) depended on how busy the card was.
- Arm J reused the seed 11 selector and reads two tiles (147 tokens). It reached 0.97 on seed 11. Its seed 12 run was still running at the time of writing.

**Pass rule and status.** The written rule for a tile arm (5 November) requires, on each seed, colour at least 0.80, captions within 0.03 of the squashed arm on the same seed, and at most 150 image tokens. Full tiles (B) meet the colour and caption gates on both seeds. H (0.74 and 0.72) misses the 0.80 bar on both seeds. H3 misses on seed 12. H2 fails on both seeds. Arm I meets all three conditions on both seeds, and J meets them on seed 11. **Arms I and J are provisional, not a pass.** The audit below applies to them, and their two seeds share one training-canvas stream.

**Audit of the eye tests (inflation risk).** Two problems were found in the scripts that produced arms A to J. First, the training canvases came from one fixed random stream for every seed, so seeds 11 and 12 share their canvases and the seed spread is understated. Second, the same 300 test canvases were used for every arm, and for H, H2 and H3 they also chose the question wording and the gate. Results from these arms may therefore be inflated. Until a clean run confirms them, the colour numbers above are reported, not established.

**Clean rerun: queued, not started.** A written rule (5 November, amended before any code for it was written) fixes both problems. Each seed gets its own canvas streams, validation and test splits are disjoint, the wording choice and the gate use validation only, and the test set is scored once. It runs arms A, B and H3 on seeds 21, 22 and 23, each at batch 4 with 9,600 examples and 2,400 steps (nine jobs), so all arms get the same budget. It sits in the GPU queue behind arm J's seed 12 run and is expected to take about 7 hours. A separate clean run for arm I follows it in the queue (seeds 21 to 23, selector trained on the training split only and gated on validation only, test scored once); its go-flag was set on 6 November after five CPU smoke checks passed. Arm J is not in a queued clean run.

**Earlier 15-minute runs (wall-clock budgets, not used for verdicts).** On stretched Flickr pictures, caption retrieval among 300 was 0.40 for the squashed 224 view (49 tokens), 0.46 for the squashed 384 view (144 tokens) and 0.39 for tiles (441 tokens). Tiles did not beat the squash on captions in these runs.

**The digit test failed.** A hidden five-digit number was tested in the same earlier runs. The score was 0.0 in every arm, including a calibration arm with 90-pixel digits. The test cannot tell resolutions apart with a frozen encoder and a small connector, so it gives no verdict on resolution.

**Seed spread.** The identical squashed arm gave colour 0.14 and 0.21 in two runs with different step counts (2,406 and 2,759 steps). Colour differences under about 0.07 in one run are not results.

**Timing.** Every eye run shared its graphics card with other jobs, and the timing fields are marked invalid. Frame times such as 280 ms (H3, seed 12) or 1,040 ms (I, seed 11) are not speed claims. A quiet-card timing is **PENDING**.

**Real frames.** Real video frames have not been tested. The square is drawn on Flickr8k photographs.

## 3. Speed

**FACT, brain speed (SmolLM2-360M, fp16, 40 new tokens, medians of five runs, RTX 2060 SUPER).**

| Arm | Tokens per second | Speed against no cache | Same tokens as the plain loop |
|---|---|---|---|
| Plain loop, no cache | 39.2 | 1.00x | baseline |
| Default cache | 41.2 | 1.05x | yes |
| Static cache plus compile | 189.9 | 4.8x | yes |
| Eager attention, default cache | 32.4 | 0.83x | yes |

The 4.8x is for the static cache and compilation together. The two were not measured separately, so the gain cannot be assigned to compilation alone. An earlier note quoted 173 tokens per second for a static cache. No result file supports that figure, so it is withdrawn. First-token medians across these arms are 25 to 34 ms.

**FACT, size benchmark (RTX 2060 SUPER, fp16, static cache plus compile).** The pass line was set first: first token 100 ms or less with eye and ear input, decode at least 40 tokens per second, and memory that fits 8 GB at 4k context with the encoders loaded. Inputs were stand-ins, so the benchmark covers speed and memory, not text quality.

| Model | Decode (tokens/s) | First token | Verdict |
|---|---|---|---|
| SmolLM2-360M | 179 | 25 ms | PASS |
| SmolLM2-1.7B | 95 | 31 ms | PASS (fits 4k; the 8k fit flag is false) |
| Qwen3-1.7B | 87 | 33 ms | PASS (the 8k fit flag is false) |
| SmolLM3-3B | 54 | 51 ms | FAIL (does not fit 4k with the encoders) |

Speed is not the limit for a 1.7B brain; memory for long context is. **PENDING:** the same benchmark on the 4060 (no result file yet), a quality test of the 1.7B models (not run), and an int8 variant (a held-out check failed; see document 22).

**Human timing envelope.** The human references are our reading of the literature. They were only partly re-checked, and where a check corrected a value, the corrected value is shown. This table is **NOT** a human-timing pass.

| Part | Human reference (approx.) | Target we set | Measured here | Status |
|---|---|---|---|---|
| Picture to first thought | Quick scene judgments about 150 ms (a narrow go/no-go task); visual reaction 200 to 250 ms (**GUESS** for general use) | 300 ms or less | First-token latency 25 to 31 ms on random-picture stand-in input | Below target on stand-in input only; real frames not timed |
| Speech to first thought | Auditory reaction about 150 ms (**GUESS**, not confirmed) | 300 ms or less | About 31 ms on 5 seconds of noise (stand-in; **PENDING**, lab note, file not re-read) | Stand-in only; not a human-timing comparison |
| Looking at frames | 3 to 5 looks per second (**GUESS**, not confirmed) | 1 to 3 looks per second | 1 look per second, evenly spaced | Glance policy not yet a win (section 2) |
| Text output | Reading about 240 to 260 words per minute, about 4 words per second (corrected from our earlier 4 to 5) | Paced to a person's reading speed when shown to a person | About 190 tokens per second (RTX 2060 SUPER, static cache plus compile) | Model much faster than reading; pacing not built |
| Long memory | No single figure | As long as possible | Reliable at 4k; 0.85 at 8k with weak early-depth recall; 0.00 for the plain model at 16k | Gap remains |

The first-token figures are well under the 300 ms target, but they are not a comparison with people. The open gaps are timing on real frames and clips, outward pacing, long memory and the glance policy.

## 4. Lessons from reviews that changed conclusions

- **Wall-clock budgets starved some arms.** The first eye runs trained for fixed minutes, so arms got different step counts (2,471 and 2,588 steps, and as low as 876 and 984 under GPU contention). Those tile results were void until rerun at equal steps. The continued-training run was also stopped by a time limit, at 244 of 360 steps. Rule now: budgets are steps or examples, never minutes.
- **A selector trained for a fixed time.** The arm I and J selectors trained for 180 seconds of wall-clock time, so their step counts varied with card load. Any rerun needs a fixed step count.
- **A shell quoting bug.** A launch script written through a double-quoted remote heredoc had its variables expanded on the wrong machine, and came out empty. It was caught before any training started. Rule now: write scripts locally and copy them.
- **Single-run noise.** The identical squashed arm varied by 0.07 between runs (section 2). Single-run gaps are read against that spread.
- **Speed numbers were capped by pacing.** A sleep inside the timed loop of the live watcher meant its speed figure could not exceed about 1.0 by construction. A replacement watcher without the sleep is built but not yet run (**PENDING**).
- **Leakage in the live hearing check.** The first live viewer used a hearing connector trained on the same film it was scored on, so its scores measured memorised dialogue. Live scoring now uses the clean-speech connector or the held-out part of the film only.
- **Test rows reused across choices.** The eye tests used the same 300 test canvases to choose wording and gates (section 2 audit). The clean rerun separates them.
- **Timing under load.** Timings taken while another job held the GPU are void, and the per-frame numbers in the early eye runs are of this kind.
- **Max-of-three timing is optimistic.** Medians of at least five runs are now used.
- **A RUNNING status is not a result.** A continued-training job showed RUNNING for about 12.5 hours after a crash at start-up. The log is read within minutes of every push now.

## What is still open

- The continued-training resume, once the Kaggle quota resets; then a key-audited needle test. The 32k, 128k and 1M stages have not started. Each has a written pass rule, and the 1M target is tried only after 128k passes.
- Arm J's seed 12 run (running), the clean rerun of arms A, B and H3 (queued, not started), and a clean run for arm I (queued, not started; arm J is not queued).
- Quiet-card timing for the eye arms and the speed benchmark, a second seed for the glance arms, and a benchmark on the 4060.
- Real video frames instead of the synthetic square.
- Most human reference values in section 3 have not been checked against their sources.

## Corrections in this revision

- H3 at seed 12 is 0.77 (the 5 November version said "not run").
- The continued-training run is INVALID (the 5 November version said pending).
- "Reliable to 8k" is replaced by the measured figures: reliable at 4k; 0.85 at 8k with weak early-depth recall.
- The glance arms have equal-step runs (F 0.21, G 0.27). The sentence that the two-glance run "ran too few steps to count" is removed; the 876-step run is void.
- The 32k and 64k zeros are withdrawn: no result file.
- Arms I and J are added, as provisional.
- The reading-speed reference is corrected to about 4 words per second; the other human references are marked **GUESS** or are unchanged from the earlier check.
