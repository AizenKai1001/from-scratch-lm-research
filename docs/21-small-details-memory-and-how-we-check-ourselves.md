# 21. Small details, memory, and how we check ourselves

*Revised 6 November 2026. Replaces the 5 November version.

Snapshot dated 6 November 2026. This document reports three things: how the eyes read a wide picture (one squashed view versus tiles, and cheaper or question-driven tile choices), whether replaying old facts keeps a small language model (the "brain", SmolLM2-360M) from forgetting them, and the checks we run on our own work before spending GPU time. Every number is a **FACT** read from our logged result files unless marked **GUESS** (reasoning, not tested), **PENDING** (planned or running, not yet measured) or **INVALID** (a run that failed its own rule). Pass and kill rules were written before each run. Most arms have one or two seeds, so treat the results as indications.

## 1. Eyes at equal examples

**Test.** A 40-pixel coloured square is placed on a 1,920 by 800 frame, and the model must name its colour (chance is 0.125). Caption retrieval is measured among 300 test captions. Every arm sees the same 9,600 training examples and is scored on the same 300 test items. Seeds are 11 and 12 unless marked.

| Arm | Image tokens | Colour, seed 11 (correct of 300) | Colour, seed 12 (correct of 300) | Captions, seed 11 / 12 |
|---|---|---|---|---|
| A: whole frame squashed to 224 (comparator; 1,200 steps at batch 8) | 49 | 0.130 (39) | 0.137 (41) | 0.243 / 0.210 |
| B: 2 by 4 tiles plus whole view | 441 | 0.967 (290) | 0.927 (278) | 0.253 / 0.340 |
| H: whole view plus the tile the question points to | 98 | 0.740 (222) | 0.720 (216) | 0.327 / 0.390 |
| H3: as H, with a reworded question | 98 | 0.880 (264) | 0.770 (231) | 0.333 / 0.360 |
| H2: whole view plus the two tiles the question points to | 147 | 0.117 (35) | 0.143 (43) | 0.277 / 0.223 |
| I: whole view plus a trained tile selector | 98 | 0.963 (289) | 0.953 (286) | 0.307 / 0.377 |
| J: whole view plus trained selector and two tiles | 147 | 0.967 (290) | running | 0.280 / not yet run |
| T2: coarse pooling of tiles | 108 | 0.130 (39) | not run | 0.213 / not run |
| T3: neighbouring tokens merged | 128 | 0.150 (45) | not run | 0.353 / not run |
| T4: 64 learned query tokens | 64 | 0.090 (27) | not run | 0.007 / not run |

**Reading (FACT unless marked).**

- The squashed view (A) misses the small square on both seeds, at chance level. Full tiles (B) read it on both seeds, but at 441 tokens, about nine times the tokens of A.
- One tile chosen by the question (H, 98 tokens) reaches 0.72 to 0.74 on both seeds, about six times chance. On seed 11 the picked tile contained the square 66 percent of the time.
- The cheaper versions of the tiles (T2, T3, T4) did not read the square on seed 11 (0.09 to 0.15). Their seed 12 runs were not made, because seed 11 failed.
- Two tiles chosen by the question (H2) fell to chance on both seeds, although it had more tokens than H. A code check on 40 test canvases found no fault. The explanation is a **GUESS**: the loss averages caption and colour tokens together, and colour tokens may carry only about 10 to 12 percent of the gradient.
- H3, a reworded version of the same question, reached 0.88 on seed 11 and 0.77 on seed 12 (mean 0.83). Its seed 12 result is 0.03 below the per-seed bar and inside the spread of about 0.07.
- A trained tile selector (I) chooses the tile from the question. On seed 11 its test accuracy was 0.93 (validation 0.955, top two 0.977); on seed 12 it was 0.92 (validation 0.95, top two 0.977). The brain then read the colour at 0.96 and 0.95 with 98 image tokens. The selector was trained for a fixed 180 seconds of wall-clock time, so its step count varied (79,146 and 83,786 steps).
- Arm J reused the seed 11 selector, reads two tiles (147 tokens), and reached 0.967 on seed 11. Its seed 12 run was still running at the time of writing.

**Pass rule and status.** The written bar for a tile arm (5 November) is: colour at least 0.80 on each seed, captions no more than 0.03 below the squashed arm on the same seed, and at most 150 image tokens.

- Full tiles (B) meet the colour and caption gates on both seeds.
- H (0.74 and 0.72) misses the 0.80 bar on both seeds.
- H3 (0.88 and 0.77) misses the bar on seed 12 by 0.03. It is not a pass.
- H2 fails on both seeds.
- I meets all three conditions on both seeds. **It is provisional, not a pass**, because of the audit in section 1 of document 20.
- J meets the conditions on seed 11; seed 12 is running.

**Audit (inflation risk).** The training canvases came from one fixed random stream for every seed, and the same 300 test canvases were used for every arm (for H, H2 and H3 they also chose the wording). The colour results may therefore be inflated. The clean rerun described in document 20 is the test that can confirm them. It is queued, not started, and it includes arms A, B and H3 only.

**Limits.**

- *Unequal updates.* Arm A ran 1,200 optimiser steps at batch 8. Arms B, H, H2, H3, I and J ran 2,400 steps at batch 4. All saw 9,600 examples, but the other arms got twice the updates. The clean rerun gives all arms the same budget.
- *Seed noise.* The same squashed arm gave colour 0.143 and 0.213 in two earlier runs with different step counts (2,406 and 2,759 steps), and captions 0.373 and 0.487. A colour difference under about 0.07 on one run is not a result, and a caption difference of about 0.1 is within the same spread.
- *Timing is void.* Every run shared its graphics card with other jobs, and the timing fields are marked invalid. Frame times were 280 ms for H3 (seed 12) and 373 ms and 1,040 ms for I (seeds 12 and 11). A quiet-card timing is **PENDING**.
- *Synthetic test only.* The square is drawn on Flickr8k captions. Real video frames have not been tested.

## 2. Does replaying old facts keep them?

**Test.** The brain learns twelve fact sets (A to L), one per stage. Each stage has 240 steps on the new set and 200 repair steps. Arm R (rehearsal, the recipe) fills 14 of the repair slots with older sets. Arm N (no rehearsal) fills them with the newest set. "Own" is recall of the set just learned. "Weakest older" is the lowest recall among older sets. "Text drift" is the change in bits per character on held-out general text compared with before training, paired and in percent; a positive number means the text got slightly worse.

| Run | Stage | Own | Weakest older | Text drift (%) |
|---|---|---|---|---|
| Seed 1, R | H (8th) | 1.00 | 0.98 | +0.80 |
| Seed 1, R | L (12th) | 1.00 | 0.95 | +1.53 |
| Seed 1, N | D (4th) | 1.00 | 0.00 | +0.12 |
| Seed 1, N | L (12th) | 1.00 | 0.00 | +1.55 |
| Seed 2, R | H (8th) | 1.00 | 0.98 | +0.32 |
| Seed 2, R | L (12th) | 0.99 | 0.94 | +0.97 |
| Seed 2, N | D (4th) | 1.00 | 0.01 | -0.40 |
| Seed 2, N | L (12th) | 1.00 | 0.00 | +1.02 |
| Seed 3, R | L (12th) | 1.00 | 0.94 | +1.52 (standard error 0.15) |

**Reading (FACT).**

- Without rehearsal, older sets were lost almost at once. On seed 1 the weakest older set was 0.04 after stage B and 0.00 to 0.01 from stage D onward. On seed 2 it was 0.04 after stage B, 0.01 at C and D, and 0.00 from E onward.
- With rehearsal, the weakest older set stayed between 0.94 and 1.00 across all twelve stages on both seeds, falling to 0.95 (seed 1) and 0.94 (seed 2) by stage L.
- The newest set was learned at 0.98 to 1.00 in every run listed.
- Mean recall over all twelve sets at the last stage was 0.98 (seed 1) and 0.97 (seed 2) with rehearsal, and 0.17 in both no-rehearsal runs.
- Text drift at the end: seed 1 +1.53 with rehearsal and +1.55 without; seed 2 +0.97 with rehearsal and +1.02 without. Drift did not differ much between arms in these runs.

**Verdict status.** The written rule reads: the effect is REHEARSAL-SUPPORTED only if arm R passes on all three seeds and arm N falls below 0.90 on the weakest older set at the last stage in at least two of three seeds. Arm N fell below 0.90 on seeds 1 and 2, so the second condition is met. Arm R passed on seeds 1, 2 and 3 (weakest older set 0.95, 0.94 and 0.94; text drift +1.53, +0.97 and +1.52 percent, inside the 2 percent bar). Both conditions are met, so the rule's verdict is **REHEARSAL-SUPPORTED**. The seed 3 no-rehearsal arm is still running and will add a third seed for that arm; it does not change the verdict unless it behaves very differently.

**Limits.**

- The facts are arbitrary three-digit codes. Recall here is memorisation of unrelated pairs. It says nothing about reasoning or general skill.
- Arm N is not a pure no-rehearsal arm. Its slots go to the newest set, so that set gets extra exposure. Any difference between arms is older-set rehearsal plus extra newest-set exposure.
- The sets are tested in the same fact format the brain was trained on. This measures retention of trained facts, not general memory.
- Drift is measured on one held-out text pool of 200 chunks. Its error bars are over chunks, not over seeds.
- The batched training path ran out of memory on the 8 GB card, so every run used the unbatched path (same maths, slower). Equivalence between the two paths was not measured.

## 3. How we check ourselves

- **Rules before runs.** Every test has a written pass rule, kill rule and verdict before the first run. Rule files are not edited after results come in. A change gets a new, dated addendum that says what changed, and it is written before any code that depends on it. For example, the clean eye rerun moved from seeds 11 and 12 to seeds 21 to 23 in an addendum written before its code was written. Document 22 describes how the order of rules and code is checked.
- **Skeptic reviews before GPU time.** Each planned GPU experiment was reviewed by two separate, read-only AI reviewers (two passes per experiment; 15 experiments, 30 passes), then fixed and checked again. In the first review wave every experiment got a "fix" verdict, and none was ready to run. In the latest pass recorded in the experiment notes (late on 5 November), 12 of the 15 had a checklist marked OK, two (K05 and K15) still needed fixes, and the status of K14 was not stated. None has started. Examples from the first wave:
  - a comparator that got twice the updates of the arms it was compared with (now addressed by the clean rerun design);
  - a progress-weight rule that would have starved the weakest sets (a set learned one stage earlier was given the floor weight);
  - a pass rule that could pass without the question helping, which led to a control with a fixed random direction in place of the question;
  - a fix that removed a slot shuffle present in the source script, which a verifier caught and restored.
- **Read the log of every remote job.** A long-context job showed RUNNING for about 12.5 hours after a crash at start-up. A second long-context run ended INVALID at 244 of 360 steps when its time limit was reached. The resume script passed a CPU-only check before any GPU time was requested.
- **Budgets are steps or examples, never minutes.** Early eye runs trained for fixed minutes, so arms got 876 to 2,759 steps depending on GPU sharing. Those arms were voided and rerun at equal steps. The later selector arms still used a fixed time (section 1), and the clean rerun does not.
- **Noise.** Single-run colour differences under about 0.07 are not treated as results. In a CPU simulation of one planned tile rule with that noise and no real effect, a false pass happened about 1.5 percent of the time with two seeds, and about 0.1 percent with a third seed (**GUESS** from the simulation, not a measured rate).
- **Scripts are written locally and copied to the GPU machine.** A launch script written inside a quoted remote command had its variables expanded on the wrong machine; it was caught before any training started.
- **Timing needs a quiet card and medians.** Timings taken while another job held the GPU are void, and timings use the median of at least five runs.
- **Data provenance.** A keyword scan of the main training text (400 million tokens) found about 46 marker strings specific to one commercial assistant and 16 generic AI phrases. A filter removes text blocks that carry a marker (about 0.014 percent of tokens), and it is applied to the replay and drift text of new runs. The filter cannot catch AI-written text that has no marker, and earlier runs used the unfiltered text. A keyword sample of about 1 percent of the blocks found nine hits, most of them false (one marker is a fragment of an ordinary word), and no model-written text on reading them. That is not proof that the text is free of AI writing.

## 4. Pending

- Arm J seed 12 (running).
- The clean eye rerun (arms A, B and H3 on seeds 21 to 23): queued behind arm J, not started. A separate clean run for arm I follows it; arm J has no queued clean run.
- Rehearsal, seed 3: the no-rehearsal arm is running. A fixed version of the test with held-out fact pairs is built and has not run.
- Quiet-card timing for the tile arms, against the 100 ms first-token target.
- Real video frames instead of the synthetic square.
- The question-driven tile choice (H) picks the wrong tile about a third of the time, so its 0.80 bar is not met.

## Corrections in this revision

- H3 at seed 12: 0.77 (the 5 November version said "not run").
- Arms I and J added, as provisional. The 5 November version had no selector arms.
- The rehearsal rule is quoted as written. The 5 November version described it as "two of three seeds show it with both arms", which left out the condition that arm R must pass on all three seeds.
- The frame-time range is replaced by the per-run values.
- The Kaggle run: its result is INVALID (244 of 360 steps), not pending.
- The provenance sentence now gives the counts and the limits of the filter.
- The "15 kits waiting" count is replaced by the latest review status (section 3).
