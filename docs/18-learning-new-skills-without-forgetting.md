# 18. Learning new skills without forgetting: learn, then heal

Snapshot dated 2 November 2026. This document reports a series of small tests of one question from the end goal in document 16: can a model keep learning new things while it is in use, without losing what it already knew? Every number is **FACT** (measured in our own logged runs, with its setting stated) unless it is marked GUESS. Every test had its pass and kill conditions written down before it ran, and every verdict below applies those conditions as written. All results come from one model size (143 million parameters) and from one to three seeds, so they are indications, not proof.

## The setting

- **The model.** The 143-million-parameter model from document 03, after a fine-tune on Countdown number puzzles (document 15). It starts with three skills that we can measure: solving Countdown puzzles (the share of puzzles with a correct answer among eight tries), predicting ordinary web text (loss, lower is better), and predicting a held-out book (loss).
- **The "new skill".** Recalling a set of 100 arbitrary facts: a short arithmetic-looking prompt ("a+b=") must be answered with a fixed, randomly chosen three-digit code. This is memorisation, not reasoning. Scoring is near-greedy and exact.
- **Why not real arithmetic.** Our first attempts used two-digit addition as the new skill. They told us nothing: the tokenizer stores every number from 0 to 100 as a single token, so the model cannot learn addition digit by digit and can only memorise the answers, and the sampled scoring was too noisy. We therefore switched to arbitrary facts, which a model of this size can learn cleanly, and which make forgetting easy to see. This is a limit of the test, not a discovery about arithmetic.
- **Rules.** Each test has a rules file with the pass and kill conditions written first. Where two methods are compared, they start from the same saved model and see the same batches of data.

## 1. One new skill: four ways to add it

The four methods, each starting from the same model:

- **Unprotected.** Train on the new facts only.
- **Rollback.** Train on the new facts, and restore the best earlier copy whenever an old skill drops by more than 5 percent.
- **Repair as first built.** When an old skill drifts, mix replay of old material into training and lower the learning rate until the skill recovers.
- **Learn, then heal.** Train on the new facts for 120 steps, then spend 80 steps at half the learning rate on batches that mix ordinary text, book text, Countdown training examples and rehearsal of the new facts.

Results (final numbers after 200 steps; Countdown is the share of 64 held-out puzzles solved in eight tries):

| Seed | Method | New facts recalled | Text loss | Book loss | Countdown |
|---|---|---|---|---|---|
| 778 | Unprotected | 100% | 4.27 | 3.67 | 4.7% |
| 778 | Rollback | 0% | 3.54 | 3.20 | 18.8% |
| 778 | Repair as first built | 2% | 3.75 | 3.15 | 7.8% |
| 778 | Learn, then heal | 99% | 3.59 | 3.06 | 12.5% |
| 779 | Unprotected | 100% | 4.06 | 3.52 | 3.1% |
| 779 | Rollback | 0% | 3.54 | 3.21 | 7.8% |
| 779 | Learn, then heal | 99% | 3.59 | 3.06 | 10.9% |

Seed 777 gave the same pattern for learn-then-heal (recall 100%, text 3.59, book 3.06, Countdown 10.9% against 17.2% for rollback). The starting values were text 3.54 and book 3.21.

Reading:

- **Rollback keeps the old skills but never learns the new one.** Its recall stays at zero, because every time the new skill starts to take hold it disturbs an old one and the copy is restored.
- **Unprotected training learns the new skill and damages the old ones.**
- **Repair as first built barely learns** (2 to 16 percent recall), so it is not better than rollback by our rule.
- **Learn, then heal** learns the new skill almost completely, keeps text loss within 0.05 of rollback, and lowers book loss below its starting value, because the healing batches contain book text.
- **The Countdown column is noisy.** On the 64-puzzle test, learn-then-heal was below rollback by one puzzle more than the written rule allowed on two of three seeds, and above it on the third. A single puzzle is within sampling noise. We therefore re-scored the saved final models of seeds 778 and 779 on 320 puzzles each (pooled 640 attempts): rollback solved 79 (12.3 percent), learn-then-heal 82 (12.8 percent). By the rule written for that re-score, learn-then-heal passes, and the earlier misses were sampling noise. The re-score did not cover seed 777, whose rollback model was not saved.

## 2. Several new skills in a row

The harder question is what happens when new skills keep arriving. The next tests teach 100 new facts per stage, one stage after another, each stage being 200 steps (learn, then heal). The healing batch is 16 examples, and what goes into it turned out to matter a great deal. The first column of each table below gives its make-up as text / book / Countdown / fact examples. "Recall" below is the recall of the weakest earlier set at the end of the chain, so it measures how much of the early learning survives.

Three skills in a row:

| Healing batch | Seed | Recall of sets A, B, C at the end | Text loss at the end |
|---|---|---|---|
| 4 text / 4 book / 4 Countdown / 4 fact examples (facts chosen at random) | 779 | 43%, 70%, 100% | 3.64 |
| 3 text / 3 book / 2 Countdown / 8 fact examples (facts balanced over the sets) | 779 | 82%, 94%, 100% | 3.63 |
| same | 778 | 78%, 93%, 99% | 3.62 |
| Unprotected (no healing) | 779 | 5%, 31%, 100% | 5.07 |

Six skills in a row (seed 779 unless stated):

| Healing batch (text / book / Countdown / facts) | Weakest set at the end | Average recall | Text loss | Book loss |
|---|---|---|---|---|
| 3 / 3 / 2 / 8 balanced | 63% | 82% | 3.73 | 3.03 |
| 3 / 3 / 2 / 8, facts chosen by current weakness | 69% | 82% | 3.71 | 3.03 |
| 4 / 4 / 4 / 4 balanced (25% rehearsal) | 40% | 63% | 3.74 | 3.03 |
| 1 / 0 / 1 / 14 balanced (seed 779) | 92% | 96% | 3.67 | 3.31 |
| 1 / 0 / 1 / 14 balanced (seed 778) | 94% | 96% | 3.70 | 3.30 |
| 1 / 0 / 1 / 14 balanced (seed 780) | 93% | not recorded | 3.69 | 3.32 |
| 1 / 1 / 1 / 13 balanced (seed 779) | 88% | not recorded | 3.65 | 3.02 |

Reading:

- **The share of the batch spent on rehearsing earlier facts is the main lever.** At 25 percent the earlier skills decay quickly; at half they mostly survive; at about 90 percent (14 of 16 examples) six skills are kept at 88 to 94 percent on three seeds. Choosing which set to rehearse by current weakness gave no clear gain over balanced rehearsal.
- **Ordinary text, book text and Countdown need very little.** One example of each per batch kept them within a small distance of their starting values. Without a book example the book loss drifts about 0.1 above its starting value (3.21); one book example per batch removed the drift (3.02) at the cost of a few points of recall.
- **Unprotected training forgets the earlier skills almost completely** and wrecks the general skills (text loss 5.07 after three skills against 3.54 at the start).

## 3. Twelve skills in a row

Using the batch of one text, one Countdown and fourteen fact examples:

| Learning rate (learn / heal) | Seed | Weakest set | Text loss | Book loss | Countdown at the end |
|---|---|---|---|---|---|
| 2e-4 / 1e-4 | 779 | 76% | 3.80 | 3.44 | 10.2% |
| 2e-4 / 1e-4 | 778 | 85% | 3.87 | 3.45 | 11.7% |
| 1e-4 / 5e-5 (halved) | 779 | 85% | 3.65 | 3.30 | 9.4% |
| 1e-4 / 5e-5 (halved) | 778 | 85% | 3.67 | 3.29 | 11.7% |
| 5e-5 / 2.5e-5 (quartered) | 779 | first set only 78% recalled; stopped | 3.58 | 3.23 | not measured |

The starting values were text 3.54 and book 3.20 to 3.21, and Countdown 10.2 percent (seed 779) and 14.1 percent (seed 778).

Reading:

- **Halving both learning rates cut the drift of the general skills by more than half** (text from +0.33 to +0.13, book from +0.24 to +0.09, on seed 778), with every new set still learned to 96 to 100 percent and the weakest earlier set equal or higher (85 percent on both seeds). Our guess is that the drift comes from the learning phases themselves, because a test that gave the healing batch two text examples instead of one made text loss worse (3.91) and cost recall, rather than better.
- **There is a floor.** At a quarter of the original rate the first set was recalled only 78 percent (our rule asked for 90 percent), so the model under-learns. A lower rate with more learning steps was not tried.
- **Text loss still creeps up with every skill** (about 0.01 per stage at the halved rate), so a long enough chain will eventually cross any fixed limit. The 24-skill run below shows this.

## 4. Twenty-four skills in a row

One seed (779), halved learning rates, healing batch of one text, one Countdown and fourteen fact examples. When more than fourteen sets had been learned, the fourteen rehearsed facts rotated over the learned sets so that each set came up regularly.

| After skill | Weakest earlier set | Text loss | Book loss |
|---|---|---|---|
| (start) | n/a | 3.543 | 3.211 |
| 6th | 96% | 3.61 | 3.26 |
| 12th | 81% | 3.66 | 3.31 |
| 18th | 85% | 3.79 | 3.41 |
| 24th | 80% | 3.81 | 3.46 |

- **Verdict by the rule written first: pass.** All 24 sets are recalled at 80 percent or more after the last stage (weakest 80 percent, average about 88 percent), Countdown is 11.7 percent against a start of 10.2 percent, and every new set was learned to at least 97 percent in its own stage.
- **Text and book loss are close to their limits.** Text rose by 0.27 against an allowed 0.30, and book by 0.24 against an allowed 0.25. A longer chain would cross them, as the 0.01-per-stage creep predicted. So 24 skills is about where this recipe stops fitting those limits, not evidence that it scales without bound.
- **Limits.** One seed, memorised facts only, and each stage used 100 facts.

## 5. Letting unimportant things fade, and a slower learning rate

**FACT, one seed each, 12 skills in a row.** Two follow-up tests, each with its pass rule written first.

- **Slower learning (a quarter of the original rate, twice the learning steps per skill).** Every set was learned to 99 percent or more. Drift of the general skills was the smallest yet (text loss +0.07, book loss +0.045, against +0.11 and +0.095 at the halved rate), at the cost of slightly more forgetting (weakest set 81 percent against 85). Verdict: pass, but not "improved" by the stricter rule, which it narrowly missed.
- **Rehearsing only the important sets.** Only four of the twelve sets were ever rehearsed after they were learned. Those four stayed at 98 to 100 percent. The other eight faded to 40 percent on average (the two oldest to 2 percent, the newest still at 93 and 100 percent), so the model forgets what it does not use and keeps what matters, a little like a person. Verdict: pass. Drift of the general skills was the same as with full rehearsal (text +0.07, book +0.04), so what limits drift is the learning rate, not how much is rehearsed.
- **Limit.** The "important" sets were chosen by us in advance. How a real system decides what matters is untested.

## 6. A skill that needs reasoning, not memory

**FACT, one seed, one skill.** The biggest open question was whether learn-then-heal works when the new skill cannot be memorised. The new skill was sorting four numbers from 0 to 99 (for example "sort 52 7 90 13" should give "7 13 52 90"), scored by exact match on 200 fresh puzzles never seen in training. The model started at 0 percent. Both arms started from the same saved model: learn-then-heal (600 learning steps, then 100 healing steps with a batch of one text example, one Countdown example and fourteen sorting examples), and unprotected (700 steps on sorting only).

| Arm | Sorting solved | Text loss | Book loss | Countdown |
|---|---|---|---|---|
| Start | 0% | 3.54 | 3.21 | 10.2% |
| Learn-then-heal | 88.5% | 3.60 | 3.25 | 8.6% |
| Unprotected | 85.0% | 3.85 | 3.41 | 2.3% |

Verdict by the rule written first (sorting at least 60 percent, text and book loss up by no more than 0.15, Countdown within 6 points of its start): **pass** for learn-then-heal. During learning Countdown fell to 1.6 percent and recovered during the healing phase, which is what the healing phase is for. Limits: one seed, one simple skill, and the healing batch here rehearses fresh sorting puzzles, not a stored set. Whether it holds for harder skills, or for several reasoning skills in a row, is untested.

**Three reasoning skills in a row (FACT, one seed).** The same recipe was then run on three skills one after another: sort four numbers, reverse four numbers, and find the largest of four. Each stage was 600 learning steps and 100 healing steps, and the healing batch rehearsed the skills learned so far with fresh puzzles.

| After stage | Sorting | Reversing | Largest | Text loss | Book loss | Countdown |
|---|---|---|---|---|---|---|
| Start | 0% | 0% | 0% | 3.54 | 3.21 | 10.2% |
| Sort learned | 83% | 4% | 0% | 3.60 | 3.25 | 11.7% |
| Reverse learned | 77% | 100% | 0% | 3.62 | 3.27 | 10.2% |
| Largest learned | 68% | 100% | 97% | 3.62 | 3.26 | 10.9% |

Verdict by the rule written first (all three skills at 60 percent or more, text and book loss up by no more than 0.20, Countdown within 6 points of its start): **pass**. The sorting skill, the hardest of the three, slowly faded (83, 77, 68 percent) and is the closest to the line, so a longer chain of reasoning skills would need more healing or more rehearsal of the hard ones. This is untested beyond three skills.

**Longer healing (FACT, one seed).** Doubling the healing phase to 200 steps per stage raised the final sorting score from 68 to 75.5 percent (reversing 100, largest 98), with text loss up only 0.05, book loss up 0.035 and Countdown at 14.1 percent. Verdict: pass, but it missed the stricter "improved" line (sorting at 80 percent or more) by 4.5 points. Sorting still faded a little at every stage (82.5, 76.5, 75.5), so the hard skill is the one that needs the most care.

## 7. Repeating the key results on new seeds (cloud GPUs)

**FACT.** To find out which findings are real and which were luck, five of the tests above were repeated on new random seeds on rented cloud GPUs, with the pass lines unchanged.

| Test (seed) | Weakest skill at the end | Text loss | Book loss | Verdict by the written rule |
|---|---|---|---|---|
| 12 fact skills, quartered rates (new seed) | 87% (home seed: 81%) | 3.616 (home: 3.613) | 3.238 | pass |
| 12 fact skills, halved rates (third seed) | 85% (home seeds: 85%, 85%) | 3.698 (home: 3.653, 3.673) | 3.325 | pass |
| 3 reasoning skills, 100 healing steps (new seed) | sort 79% (home seed: 68%) | 3.609 | 3.243 | pass |
| 3 reasoning skills, 200 healing steps (new seed) | sort 87.5% | 3.587 | 3.230 | **improved** |
| 3 reasoning skills, 200 healing steps (second new seed) | sort 76% | 3.597 | 3.248 | pass |

Reversing stayed at 100 percent and finding the largest at 97 percent or better in every reasoning run. Reading across the seeds:

- **The fact-skill results repeat.** Weakest-skill recall of 81 to 87 percent and text loss around 3.61 to 3.70 on four seeds in total. The slower learning rate again gave the least drift.
- **Longer healing helps the hardest reasoning skill, and it repeats.** On every matched pair, 200 healing steps beat 100 (sorting 68 to 75.5 percent on the home seed, 79 to 87.5 percent on the first cloud seed). The size of the gain varies a lot from seed to seed (the sorting score at 200 steps was 75.5, 87.5 and 76 percent), so single-seed numbers for the hard skill should be read as roughly plus or minus 6 points.
- Cost of this batch on rented hardware: under 9 US dollars for five runs of one to one and a half hours each.

## What this does and does not show

- It is evidence, at small scale, that a model can take on new material and keep what it had if the new learning is followed by a short healing phase that is mostly rehearsal of what was learned before. Rolling back, in our test, protects old skills only by refusing to learn.
- **Most of the new skills are memorised facts.** Facts are easy to rehearse exactly. Section 6 shows the recipe working for one reasoning skill (sorting), but several reasoning skills in a row, harder skills, and material that cannot be replayed verbatim are untested and are the biggest open questions.
- **Rehearsal needs old material to be kept.** An always-learning system would need a small store of earlier examples, and a way to decide what to keep as the number of skills grows.
- One model size and one to three seeds. The 64-puzzle Countdown test is too noisy to decide small differences; the 128-puzzle and 320-puzzle versions are better but still limited.
- The learning rates and batch mixes were tuned within these tests, one change at a time, each against a rule written before the run. They are not proven to be best.
- On the starting numbers: the rented cloud GPU runs redid the first training phase on different hardware, and reproduced the starting text and book losses to two decimals.
