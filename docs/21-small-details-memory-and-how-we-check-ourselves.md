# 21. Small details, memory, and how we check ourselves

Snapshot dated 5 November 2026. This document reports three things: how the eyes read a wide picture (one squashed view versus tiles, and cheaper or question-driven tile choices), whether replaying old facts keeps a small language model (the "brain", SmolLM2-360M) from forgetting them, and the checks we run on our own work before spending GPU time. Every number is a **FACT** read from our logged result files unless marked **GUESS** (reasoning, not tested) or **PENDING** (planned or built, not yet measured). Pass and kill rules were written before each run. Most arms have one or two seeds, so treat the results as indications.

## 1. Eyes at equal examples

**Test.** A 40-pixel coloured square is placed on a 1920 by 800 frame, and the model must name its colour (chance is 0.125). Caption retrieval is measured among 300 test captions. Every arm sees the same 9,600 training examples. Each arm was run with two seeds (11 and 12) unless marked "not run".

| Arm | Image tokens | Colour, seed 11 | Colour, seed 12 | Captions, seed 11 / 12 |
|---|---|---|---|---|
| A: whole frame squashed to 224 | 49 | 0.130 | 0.137 | 0.243 / 0.210 |
| B: 2 by 4 tiles plus a whole view | 441 | 0.967 | 0.927 | 0.253 / 0.340 |
| H: whole view plus the tile the question points to | 98 | 0.740 | 0.720 | 0.327 / 0.390 |
| H3: as H, with a refined question wording | 98 | 0.880 | not run | 0.333 / not run |
| H2: whole view plus the two best tiles | 147 | 0.117 | 0.143 | 0.277 / 0.223 |
| T2: tiles with coarse pooling | 108 | 0.130 | not run | 0.213 / not run |
| T3: tiles with neighbouring tokens merged | 128 | 0.150 | not run | 0.353 / not run |
| T4: 64 learned query tokens | 64 | 0.090 | not run | 0.007 / not run |

**Reading (FACT).**

- The squashed view (A) misses the small square, at chance level on both seeds. Full tiles (B) read it on both seeds, but at 441 tokens, about nine times the tokens of A.
- One tile chosen by the question (H, 98 tokens) reaches 0.72 to 0.74 on both seeds, about six times chance. On seed 11 the chosen tile contained the square 66 percent of the time (centre test).
- The cheaper versions of the full tiles (T2, T3, T4) did not read the square on seed 11 (0.09 to 0.15). Only the full tiles have done so so far.
- Two tiles chosen by the question (H2) fell to chance on both seeds, although it had more tokens than H. A CPU check on 40 canvases found no code fault. The debugging note reads the drop as a GUESS: the loss averages caption and colour tokens together, and by its estimate colour tokens carry only about 10 to 12 percent of the gradient, so the colour signal may be weak.
- H3, a refined wording of the same question, reached 0.88 on seed 11 only.
- Earlier, in runs of 2,000 steps each, a glance arm (whole view plus one full-resolution crop, 98 tokens) reached colour 0.21 and captions 0.473, and a two-crop arm (147 tokens) reached 0.267 and 0.453. Neither passed the rule (colour at least 0.15 above the squashed arm).

**Pass rule and verdict.** The pre-registered bar for a tile arm was colour at least 0.80 on each seed, with captions no more than 0.03 below the squashed arm. Full tiles (B) meet both gates on both seeds. H (0.74 and 0.72) misses the 0.80 bar on both seeds. H3 has one seed. H2 fails on both seeds.

**Limits.**

- **Unequal updates.** The squashed comparator (A) ran 1,200 optimiser steps at batch 8, while B, H, H2 and H3 ran 2,400 steps at batch 4. All saw 9,600 examples, but the new arms got twice the updates. A comparator with the same number of steps is **PENDING**.
- **Seed noise is larger than the colour gaps we care about.** The same squashed arm gave colour 0.143 and 0.213 in two earlier runs (2,406 and 2,759 steps), and captions 0.373 and 0.487. A colour difference under about 0.07 on one run is not a result, and a caption difference of about 0.1 is within the same spread.
- **Timing is void.** All runs shared the GPU with other jobs, and the frame times (287 to 294 ms) reflect that sharing. A quiet-GPU timing run is **PENDING**.
- **Synthetic test only.** The square is drawn on Flickr captions. Real film frames have not been tested.

## 2. Does replaying old facts keep them?

**Test.** The brain learns twelve fact sets (A to L), one per stage. Each stage has 240 steps on the new set and 200 repair steps. Arm R (rehearsal) fills the repair steps with 14 slots of older sets. Arm N (no rehearsal) does not. "Own" is recall of the set just learned. "Weakest older" is the lowest recall among older sets. "Text drift" is the change in bits per character on held-out general text compared with before training, paired and in percent; a positive number means the text got slightly worse.

| Run | Stage | Own | Weakest older | Text drift |
|---|---|---|---|---|
| Seed 1, R | H | 1.00 | 0.98 | +0.80 |
| Seed 1, R | L (12th) | 1.00 | 0.95 | +1.53 |
| Seed 1, N | D | 1.00 | 0.00 | +0.12 |
| Seed 1, N | L (12th) | 1.00 | 0.00 | +1.55 |
| Seed 2, R | H | 1.00 | 0.98 | +0.32 |
| Seed 2, R | L (12th) | 0.99 | 0.94 | +0.97 |
| Seed 2, N | not counted | | | |

**Reading (FACT).**

- Without rehearsal, older sets were lost at once. On seed 1 the weakest older set was 0.04 after stage B, and 0.00 to 0.01 from stage D to the end.
- With rehearsal, the weakest older set stayed between 0.94 and 1.00 across the twelve stages on both seeds, falling to 0.95 (seed 1) and 0.94 (seed 2) by stage L.
- The newest set was learned at 0.98 to 1.00 in every run listed.
- Text drift did not differ much between arms on seed 1 (+1.53 with rehearsal, +1.55 without). Rehearsal did not add visible drift in that run. Seed 2 with rehearsal ended at +0.97.

**Limits.**

- Only seed 1 has both arms complete. The seed 2 no-rehearsal file holds only its settings and initial scores and has no stage results, so it is not counted. Seed 3 has not been run.
- Our pre-registered rule calls the effect "rehearsal-supported" only when two of three seeds show it with both arms. That condition is **not yet met**, so this document does not claim it.
- The sets are tested in the same fact format that the brain was trained on. This measures retention of trained facts, not general memory.
- Drift is measured on one held-out text pool.

## 3. How we check ourselves

- **Rules before runs.** Every test has a written pass rule, kill rule and verdict before the first run (for example, the tile bar above). Rule files are not edited after results come in. A change gets a new, dated addendum that states what changed.
- **Skeptic reviews before GPU time.** An independent reviewer reads the code, the rules and the queue before any run. In our latest set of 15 planned GPU kits, every kit received a FIX verdict and none was ready to run at first. The reviews found blocking problems, for example:
  - a comparator that got twice the updates of the arms it was compared with;
  - a progress-weight rule that would have starved the weakest sets (a set learned one stage earlier was given the floor weight);
  - a pass rule that could pass without the question helping, which led to a control with a fixed random direction in place of the question;
  - a fix that removed a slot shuffle present in the source script, which a verifier caught and restored.
  
  A later check of our own fixes is part of the same loop: the reviewer and verifier are different agents from the one that wrote the fix.
- **Budgets are steps or examples, never minutes.** Early eye runs trained for fixed minutes, so arms got 876 to 2,759 steps depending on GPU sharing. Those arms were voided and rerun at equal steps.
- **A Kaggle run that never ran.** A long-context run was pushed to Kaggle and reported as RUNNING for about 12.5 hours. Its log showed a crash after 60 seconds: a `TypeError` about a `rope_theta` argument that the newer transformers library no longer accepts in that form. The lesson is to read the kernel log within minutes of a push and never to trust a RUNNING status. The next version passed a CPU-only smoke test (the model loads with the corrected position settings and the script now exits on any error), then crashed at its first training step with an out-of-memory error: on the 16 GB T4 the grouped-query attention fell back to a path that builds the full attention matrix. That was fixed, a short GPU memory test passed at 8k, 16k and 32k tokens (peak 3.8, 4.9 and 7.4 GB without optimizer states), and the full run was pushed again. Its result is **PENDING**.
- **Noise.** Single-run colour differences under about 0.07 are not treated as results. In a CPU simulation of one planned tile rule with that noise and no real effect, a false pass happened about 1.5 percent of the time with two seeds, and about 0.1 percent with a third seed (**GUESS** from the simulation, not a measured rate).
- **Scripts are written locally and copied to the GPU machine.** A launch script written inside a quoted remote command had its variables expanded on the wrong machine; it was caught before any training started.
- **Timing needs a quiet card and medians.** Timings taken while another job held the GPU are void, and timings use the median of at least five runs.

## 4. Pending

- Seed 12 for H3, and seed 12 for T2, T3 and T4 (the T arms were not repeated after seed 11 failed; a pass needs both seeds).
- A squashed comparator at the same number of steps as the tile arms.
- Quiet-GPU timing for the tile arms, against the 100 ms first-token target.
- Real film frames instead of the synthetic square.
- The question-driven tile choice picks the wrong tile about a third of the time. Its 0.80 bar is not yet met.
- Rehearsal: a second-seed no-rehearsal run (the current file is incomplete) and a third seed for both arms.
- Data provenance: a keyword scan of the main training text (400 million tokens) found a few dozen web-page captions written by AI models. A filter now removes the blocks that carry such markers (0.014 percent of tokens, with a safety margin) from the replay and drift text of new runs. Text written by AI without a marker cannot be removed this way, and the earlier runs used the unfiltered text.
- Fifteen planned kits are waiting on fixes and on a shared GPU lock. Their memory and cost numbers are estimates until the first measured run.
