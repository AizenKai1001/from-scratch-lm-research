# 25. Looking in two steps, and what can be cut from a small model

## What this is

Snapshot dated 6 November 2026. This document reports six findings from one day of tests. Each test was run against rules that were written and fingerprinted before it ran. Four findings use Qwen3.5-0.8B, a small open language model with a vision part. Qwen3.5-0.8B is released under Apache-2.0 (**PUBLISHED**: the model card and the LICENSE file in the release). Two findings use SmolVLM2-500M, a smaller vision model. The picture tests use one synthetic target type: a small coloured square drawn on a photograph. The vocabulary test used children's books and one dictionary passage. The quantisation and layer tests each used one 2,000-token Project Gutenberg passage. Treat every result as an indication.

Labels used in this document. **FACT**: measured by our runs; the setting and sample size are given where the notes give them. **COMPUTED**: arithmetic on measured or stated numbers. **PUBLISHED**: from a published source, named in the text and not re-checked here. **GUESS**: our reasoning, or anything beyond the tests. Where a claim could take two labels, we use the more cautious one.

## Summary of the six findings

| Finding | Test | Verdict, as the rules read it | Main number |
|---|---|---|---|
| 1. Second look behind a confidence gate | First test, uncalibrated SmolVLM2 reader | NOTHING TO GATE | One look 0.796, two looks 0.800 (test set) |
| 1. (repeat) | Same rules, calibrated SmolVLM2 reader | GATE WORKS | One look 0.885; gated 0.933 at 1.154 looks per picture |
| 2. Two squares, mask against going back | Uncalibrated SmolVLM2 reader, not repeated | MASK NOT SHOWN TO HELP | 1.685 against 1.581 squares read right per picture (+0.104; bar +0.15) |
| 3. Cheap glance, then a zoom | Glance of 90 tokens plus a 256-token crop | NOT SHOWN (time bar missed) | 0.950 at 346 tokens; 155.4 ms against 150 ms |
| 3. (resolution ladder) | Direct read at seven resolutions | No pass or fail bar for the floors; token floors reported | 40-pixel square: 198 tokens at the 90% level |
| 4. Weights in fewer bits | Six parts; groups of 32 and 128 | 8 bits PASS; 4 bits FAIL for the combined condition and the language blocks (borderline); vision projections PASS in groups of 32 | About 0.90 GB against 1.75 GB (COMPUTED) |
| 5. Smaller vocabulary | Byte spelling and sub-word spelling of cut-off tokens | Byte spelling: no trim point. Sub-word spelling: trim point 32,768 | 25.3% fewer parameters (COMPUTED); +0.96% bits per byte on the first book passage, against a 1.01 bar (COMPUTED) |
| 6. Removing whole layers | Each layer removed, then patched | KILL (depth): yes. FREE: no. PASS (repair): no. KILL (repair): no | Zero of 24 layers free; lowest text rise +2.6% |

The verdicts are the rules' own words. NOT SHOWN means the bar was not cleared. It does not mean the effect is zero.

## 1. A second look behind a confidence gate

**What we asked.** If the first look is unsure, does a second look help? Can the model's confidence decide when to look again?

**What we did (pre-registered).** The rules came first. The gate threshold was chosen on a validation set, with at most 1.30 looks per picture on average. The second look had to come from outside the first crop, and the more confident look gave the answer. The first test used SmolVLM2's uncalibrated reader. The repeat used the same rules with the calibrated reader: the eight fixed colour numbers from document 24, not refitted. Because the training photos were nearly used up, the repeat used photos from the dataset's validation and test files. An addendum recorded that change before the repeat ran.

**What we found.** Accuracy on the test sets (**FACT**):

| Run (pictures) | Reader | One look | Always two looks | Gated second look |
|---|---|---|---|---|
| First test, test set (240) | Uncalibrated | 0.796 | 0.800 | 0.800 at 1.225 looks |
| Repeat, test sets (480) | Calibrated | 0.885 | 0.919 at 2.0 looks | 0.933 at 1.154 looks |

- First test (**FACT**): **NOTHING TO GATE.** Two looks scored 0.800 against 0.796 (+0.004, one picture), far below the +0.03 bar.
- Repeat (**FACT**): **GATE WORKS.** The gate raised accuracy by +0.048 over one look on the test sets (bar +0.03), at 1.154 looks per picture (limit 1.30). It beat the rule's expected score for a random gate (0.891, from the rule's formula, not from a random run) by 0.043.
- Against always taking two looks (**FACT**): the gate scored 448 of 480 and two looks 441. The gate was right on 10 pictures where two looks were wrong, and two looks were right on 3 where the gate was wrong. With counts this small, that gap is not resolved.
- Confidence was a usable miss alarm: area under the curve 0.88 in the first test, and 0.965 (interval 0.938 to 0.985) in the repeat (**FACT**). It was misleading about colour: the first reader called cyan "blue" with confidence (**FACT**). Our explanation (**GUESS**, supported by the repeat): once the reader was calibrated, the same confidence became a useful alarm for a missed crop.
- Cost (indicative): about 101 ms per picture (**FACT**), and about 138 image tokens (**COMPUTED**: 64 for the whole view plus 64 per look, times 1.154 looks).

**Caveats.** One synthetic target: a 40-pixel coloured square on a photograph. One SmolVLM2 reader and one selector head, both trained on this generator. The repeat's gain was smaller on the validation set (+0.029) than on the test sets (+0.048). The first test's three fresh sets, all uncalibrated, scored 0.833 on the published set, 0.783 on the first test's validation set and 0.796 on its test set. The published set's lead over the mean of the other two (0.043, z about 1.4) is within noise, and differences within noise are not effects. A post-hoc look tried a summed-log rule, which scored 0.565; that rule was not pre-registered. The calibration numbers were fitted on another set and not refitted. Not yet audited by an independent pass.

## 2. Two squares, and a mask against going back

**What we asked.** With two squares in one picture, does forbidding a later look from returning to earlier places help find both? Does it hurt answers on single squares?

**What we did (pre-registered).** Three looks per picture, in three arms. Arm A takes the three best-scoring places. Arm B adds a neighbour mask: the next look may not sit within one cell of an earlier look. Arm C adds a box mask: the next look's centre must lie outside the earlier crops. The test used 480 two-square pictures and 240 one-square pictures. The rule file was fingerprinted before the scripts existed and before the first test's numbers were read. The SmolVLM2 reader was uncalibrated, and the test was not repeated.

The idea of not returning to places already seen is called inhibition of return. A review of it is Satel, Wilson and Klein (2019), "Inhibition of return", *Vision (Basel)* 3:58 (PMC6969912) (**PUBLISHED**). Our mask is an engineering choice inspired by this idea. It is not a test of the biology.

**What we found.** Verdict: **MASK NOT SHOWN TO HELP** (**FACT**).

| Arm | Squares read right per picture | Both squares read right | One-square accuracy |
|---|---|---|---|
| A, plain top three | 1.581 | 61.5% | 0.788 |
| B, neighbour mask | 1.669 | 70.0% | 0.750 |
| C, box mask | 1.685 | 71.7% | 0.754 |

- The box mask scored higher on squares read right: +0.104 over plain top three (interval 0.071 to 0.138). That interval is above zero, but the gain is below the +0.15 bar. The first set gave +0.092 and the second +0.117. The bar needed +0.10 in each set.
- One-square accuracy was lower with the mask: 0.754 against 0.788 for plain top three, and 0.792 for one look. The rule reads this as harm, because its point rule allows a drop of at most 0.02. Exact paired tests give p = 0.077 for C against A, and p = 0.049 for B against A. One is just under 0.05 and one is not, and we made no adjustment for the two comparisons. So this is weak evidence of harm, not a clear effect. The numbers are **FACT**; the reading is ours.
- An audit recomputation found that 11 of the 12 pictures lost under the box mask came from a look that did not contain the square (**FACT**).

**Caveats.** Synthetic: two coloured squares per picture, three looks, one uncalibrated SmolVLM2 reader. The thresholds came from a draft proposal, and the verdict depends on them. The most-confident answer rule is the likely cause of the one-square drop (**GUESS**); another rule was not tested. Independently audited: every headline number matched the stored rows, and some wording was corrected.

## 3. A cheap glance, then a zoom

**What we asked.** Can a low-resolution glance at the whole picture, about 90 image tokens, find a small square well enough for a 256-token crop to name its colour? And how few image tokens does a direct read of the whole picture need?

**What we did (pre-registered).** The locator is a linear probe on Qwen3.5-0.8B's own frozen features. It was trained on 1,200 generated pictures, and its layer and strength were chosen on 240 validation pictures. The test used 240 fresh photos with a 40-pixel square, the same photos with 24-pixel squares, and 240 no-square controls. The rule set bars for locating (0.90), accuracy (0.90), controls (below 0.20), tokens (a bar of 400) and median time (150 ms). The resolution ladder showed the same canvas at seven token counts, from 1,500 down to 66, for squares of 8 to 40 pixels. It used 240 pictures per cell and 120 no-square pictures per resolution.

**What we found.** Glance variants on the test set (**FACT**):

| Glance | Image tokens (glance plus crop) | Crop held the square | Colour right (95% interval) | Median time (ms) |
|---|---|---|---|---|
| 66 tokens | 322 | 0.892 | 0.917 (0.875 to 0.945) | 151.1 |
| 90 tokens (main) | 346 | 0.933 | 0.950 (0.915 to 0.971) | 155.4 |
| 390 tokens | 646 | 0.983 | 0.983 (0.958 to 0.994) | 186.2 |

- Verdict: **NOT SHOWN** (**FACT**). The locator, accuracy, control and token tests passed. The time test did not: 155.4 ms against a bar of 150 ms, on one timing run. That is a gap of 5.4 ms (**COMPUTED**).
- The full view took about 333 ms in an earlier run, timed in another session. That is about 2.1 times the glance eye's 155.4 ms (**COMPUTED**). The ratio is only indicative.
- The 66-token glance was adopted under the rule's descriptive reading, because its numbers were within 0.05 of the 90-token glance. That reading uses the test numbers, so it is descriptive only.
- For 24-pixel squares, the 390-token glance named the colour right in 0.950 of pictures, against 0.808 for the 90-token glance (**FACT**).
- The resolution ladder sets no pass or fail bar for its floors. It reports the smallest token count at which the colour was named right in at least 90% of pictures (or 80%), and at every larger count too. Its sanity and control checks passed (**FACT**).

| Square side | 90% floor | 80% floor |
|---|---|---|
| 40 px | 198 tokens | 90 tokens |
| 24 px | 390 tokens | 198 tokens |
| 16 px | 608 tokens | 390 tokens |
| 12 px | 1,500 tokens | 608 tokens |
| 8 px | none (0.81 even at 1,500) | 1,500 tokens |

- At 198 tokens the 40-pixel square was named right in 0.946 of pictures, with a median forward time of 78 ms. At 390 tokens the 24-pixel square reached 0.925, at 107 ms. These are forward passes, not end-to-end times. No-square controls stayed between 0.092 and 0.133 at every resolution (bar 0.20) (**FACT**).

**Caveats.** One synthetic target type. The probe was trained on this generator and asked one question. The image tokens come before the question in the prompt, so another question could not steer it; a question-first test is planned. Every rung is the whole canvas resized, so the floors bound plain resizing, not token-merging or pruning methods, which were not tested. At 198 tokens the 40-pixel square is about 15 pixels wide in the resized picture (**COMPUTED**). The full view's 0.992 came from an earlier, different set, so comparisons with it are indicative. Both results were audited independently, and every table value and reading matched. A confidence-gated second crop for this eye is registered and either running or queued; it has no result yet.

## 4. Storing weights in fewer bits

**What we asked.** Can the weights be stored in 8 or 4 bits per number without losing accuracy on the tests? Which parts can take the lower precision?

**What we did (pre-registered).** We rounded each part's weights to 8 bits, or to 4 bits in groups of 32 or 128, and stored them back at 16 bits. This simulates storage; speed was not measured. The 4-bit kill rule stopped a part if its text surprise rose by more than 5% on one 2,000-token Project Gutenberg passage. Picture screens were confirmed on 240 pictures, and combined conditions were tested too.

**What we found.** (**FACT**)

| Condition | Result | Reading |
|---|---|---|
| Each of six parts at 8 bits | Text rise at most +0.06%; no picture drop above the 0.03 bar | PASS for all six |
| All parts at 8 bits together | Text +0.10%; crop 216 of 240; whole canvas at 8 px 192 of 240 (drop +0.013) | PASS |
| Vision projections at 4 bits, groups of 32 | Crop 214 of 240 (drop +0.004); whole canvas at 12 px 219 against 217 | PASS on its own test |
| Language blocks at 4 bits, groups of 128 | Text +2.00%; crop 202 of 240 (drop 0.054, 0.004 over the bar) | FAIL, borderline |
| All parts at 4 bits (groups of 128), embedding table at 8 bits | Text +5.05% (0.05 points over); whole canvas at 8 px 154 of 240 (drop 0.171); at 12 px 198 (drop 0.079) | FAIL on three counts |

- No part was killed at 4 bits. The largest text rise was +2.00%, in the language blocks (**FACT**).
- All six parts at 8 bits would take about 0.90 GB, against 1.75 GB at 16 bits (**COMPUTED** from the file). The rule's plan reading is: every part at 8 bits, with the vision projections allowed at 4 bits in groups of 32 (**FACT** for the reading).

**Caveats.** The vision projections at 4 bits lost 16 of 240 pictures on the 8-pixel whole-canvas test (179 against 195; drop 0.067). They passed the 12-pixel test the rule used, and our review would not call them safe for small targets. The confirmations reuse the screen pictures, so they are not a fresh sample. The language-block failure is one picture (13 lost against 12 allowed), so it is read as borderline. Differences of about 0.03 to 0.05 are not resolved with 240 pictures. A calibrated 4-bit method is planned (**GUESS** on whether it would pass). Code-reviewed; not yet independently audited.

## 5. A smaller vocabulary

**What we asked.** Can the 248,320-token vocabulary be cut to a smaller kept set? And how should cut-off tokens be written, so that the model still predicts text well?

**What we did (pre-registered; the sub-word rule was written after a debug pass, see below).** The kept sets were the most frequent tokens of 400 public-domain children's books (15.77 million tokens), plus the always-kept base tokens (special, byte, prompt and colour tokens), at 8,192, 16,384, 32,768 and 65,536 tokens. A cut-off token was written either byte by byte, or as the longest kept pieces from the left (sub-word). The measure was bits per byte on five held-out 2,000-token children's-book passages and one dictionary passage, trimmed against original. The sub-word rule was written after a console-only debug pass of the byte version had shown the byte version to be costly; its background says so.

**What we found.** Ratio of trimmed to original bits per byte (1.000 means no change; book passages as a range) (**FACT**):

| Kept tokens | Byte spelling, books | Byte spelling, dictionary | Sub-word, books | Sub-word, dictionary |
|---|---|---|---|---|
| 65,536 | 1.000 to 1.029 | 1.189 | 1.000 to 1.002 | 1.023 |
| 32,768 | 1.028 to 1.212 | 1.398 | 1.000 to 1.010 | 1.055 |
| 16,384 | 1.316 to 1.713 | 2.398 | 1.040 to 1.096 | 1.182 |
| 8,192 | 1.677 to 2.398 | 3.390 | 1.104 to 1.188 | 1.419 |

- Byte spelling: no trim point. It missed the bar at every size tested (**FACT**).
- Sub-word spelling: trim point 32,768. The bars (1.01 at 32,768 and 65,536, 1.02 at 16,384, 1.05 at 8,192) apply to each of the five book passages (H1 to H5). The dictionary is not part of the bar. They were met at 32,768 and 65,536. At 32,768 the first book passage was 1.0096, just inside its 1.01 bar (+0.96%, **COMPUTED**); the other four were +0.3%, 0.0%, +0.8% and +0.3%. The dictionary cost 5.5% more (**FACT**).
- Only 0.1% to 0.95% of tokens in the books needed the fallback spelling (3.05% in the dictionary) (**FACT**).
- Parameters fell from 873,438,784 to 652,713,536 at 32,768 kept tokens, a cut of 25.3% (**COMPUTED** from the model file). The output step took 0.282 ms instead of 2.009 ms, one measurement each (**FACT**), which is 7.1 times faster (**COMPUTED**).
- Plan, not applied: the 32,768 sub-word variant is the planned base, and nothing was applied to any weights. With 8-bit storage its weights would be about 0.68 GB (**COMPUTED**; not measured).

**Caveats.** The calibration and held-out passages share one domain, so the kept set suits that domain. Other domains, languages, code, mathematics and instruction following were not measured. The splitter is greedy, not the standard procedure. Bits per byte measures next-token predictions, not generated text. The weights were unedited. Not yet independently audited.

## 6. Removing whole layers

**What we asked.** Can any of the 24 layers be removed for free (a text rise of at most 2%, and small picture losses)? If not, can a fitted linear patch repair the damage?

**What we did (pre-registered).** Each layer was removed (arm A) and then patched (arm B). Each version was scored on one 2,000-token Project Gutenberg passage (baseline surprise 3.0336 nats; lower is better) and on 120 native-crop pictures. The three layers with the lowest text rise (7, 11 and 12) were confirmed on 240 whole-canvas pictures at 8 and 12 pixels, with no-square controls. The patch was fitted on 256 text sequences and 200 pictures, and checked on 20 + 20 held-out ones. Some calibration pictures came from the first 220 of the 284 photos still unused, a change stated before the results.

**What we found.** (**FACT**)

| Layer (type) | Removed: text rise | Removed: 8-px picture drop | Patched: text rise | Patched: 8-px picture drop |
|---|---|---|---|---|
| 0 (linear) | +235.8% | not confirmed | +68.7% | not confirmed |
| 7 (full) | +3.75% | 0.492 | +1.8% | 0.438 |
| 11 (full) | +2.6% | 0.125 | +1.04% | 0.050 |
| 12 (linear) | +3.7% | 0.158 | +1.7% | 0.058 |

- FREE on the screen: none. The lowest text rise was +2.6%, at layer 11. FREE, confirmed: none. PASS (repair): none. The closest, layer 11, had a text rise of +1.04% (limit 1%) and a picture drop of 0.050 (limit 0.03). KILL (repair): not triggered (median recovered share 0.58). KILL (depth): yes, because zero layers were free on the screen.
- Layer 0 is essential: removing it raised the text surprise from 3.03 to 10.19 nats. Layer 23 shows text and pictures disagreeing: its text rise was +87.6%, yet 110 of 120 native-crop pictures stayed right.
- The patch recovered about 58% of the lost text quality at the median layer, but on pictures it was not enough.
- One layer is only about 2.1 to 2.5% of the parameters (**COMPUTED** from the file's shares).
- For SmolVLM2-500M, document 23 reports a different result on a different test. The two results are not compared here.

**Caveats.** Only single-layer removal was tested. The text and picture tests are short, and the 120-picture screen resolves differences of about 0.06 at best. Only three layers were confirmed, and the patch may not transfer to other tasks. The script was not in the first code review; the result is not yet independently audited.

## 7. How the program works

- **Rules and fingerprints first.** Each rule file, with its thresholds, was fingerprinted (a checksum and a time) and logged before its script existed. Thresholds were not changed after results. Exceptions are stated where they occur: the sub-word rule (written after a debug pass of the byte version), the repeat's photo source (changed by an addendum before the repeat ran), the calibration numbers taken from document 24, and the post-hoc analyses, which are labelled as such.
- **Debug passes.** Each script had a console-only debug pass on a few pictures or passages. Those numbers are not results.
- **Independent audits.** The two-square mask test, the glance eye and the resolution ladder were each checked by a separate pass that recomputed the headline numbers from the stored rows. Every headline number matched, and some wording was corrected. The second-look tests, the vocabulary tests and the layer test had no independent pass when their notes were written. The quantisation test had a code review.
- **Reviews changed readings.** A review found that the glance eye's NOT SHOWN verdict rests on one timing run, so a repeat is planned. Another found that the quantisation confirmations reuse the screen pictures, and that one 4-bit result failed by one picture. Our own reading of the results table found that the 4-bit vision projections lose pictures on small targets, which the rule did not look at.
- **Checks that changed what we claim.** A full-word spelling check of the colour reader gave the same answers as the first-token score. The tokenizer is probably not the cause (**GUESS**), but that arm could not change any answer, so it tests nothing about the tokenizer and does not exclude it. Reading the raw outputs of the first recall run exposed a scoring bug, fixed after the fact (post hoc). As a result, document 24 limits its recall claim to the easy needle case.

## 8. Not claimed

- We do not claim a general ability to see. The picture tests use one synthetic target type.
- We do not claim that the second-look gate works on natural images, with other readers, or for other targets.
- We do not claim that the mask helps with two squares (not shown). Nor do we claim that it harms single squares: the harm is the rule's reading, and the drop is not clearly more than noise.
- We do not claim that the glance eye meets the 150 ms bar. It measured 155.4 ms on one run, so NOT SHOWN stands. It does use fewer image tokens than the full view (346 against 1,500).
- We do not claim that the tokenizer has been excluded as a cause. The full-word check could not change any answer.
- We do not claim that 4-bit storage is safe. The combined 4-bit condition failed, and the vision projections lost 16 of 240 pictures on the 8-pixel test.
- We do not claim that 8-bit storage is faster. It is smaller on paper (about 0.90 GB against 1.75 GB, **COMPUTED**), and the speed of 8-bit kernels was not measured.
- We do not claim that the vocabulary cut works outside children's books. A dictionary passage cost 5.5% more at 32,768 tokens.
- We do not claim that a single layer can be removed for free, or that several can be removed together. Several layers were not tested.
- We do not claim a speed for the glance eye on optimised kernels. The figure of about 120 ms is **COMPUTED** from a speed-up measured on pictures, and was not run.
- We do not claim that the changes have been run together end to end: 8-bit weights, a trimmed vocabulary and a gate.
- We do not claim anything about always-on running, memory over long periods, or learning while running. None was tested.

## 9. What this means for a small always-on eye and brain

What the results support:

- In the calibrated SmolVLM2 repeat, a second look when unsure raised accuracy on 480 test pictures from 0.885 to 0.933, at about 1.15 looks per picture (1.154). The uncalibrated first test showed no gain (+0.004, one picture). The timings are indicative (**FACT**).
- In the calibrated SmolVLM2 repeat, two limits were measured (**FACT**). Finding the square: the crop held it in 86.3% of test pictures. Reading it: once the crop held the square, the reader named the colour right in 98.8%.
- Big targets need few tokens. A 198-token direct read named a 40-pixel square right in 0.946 of pictures. Small targets (24 pixels) needed about 390 tokens, for 0.925 (**FACT**).
- Storing the weights at 8 bits halves them on paper (about 0.90 GB against 1.75 GB, **COMPUTED**), with no drop larger than the rule allowed. Four-bit storage is not yet safe (**FACT** for the rule readings).
- A sub-word vocabulary cut to 32,768 tokens removes about a quarter of the parameters (**COMPUTED**), with up to about 1% more bits per byte on the five book passages (0.0 to 1.0 percent). On a dictionary passage the cost was 5.5% (**FACT**).
- Removing whole layers is not free in this model. No layer passed the free test, so layer removal ends here under the rule's decision link (**FACT**).

Beyond the tests (**GUESS**):

- A small always-on system might combine a cheap glance, a confidence gate and 8-bit weights. These parts were not run together end to end, and always-on running was not tested.
- A smaller brain might also use the trimmed vocabulary. Its stored size would be about 0.68 GB (**COMPUTED**, not measured).
- A confidence-gated second crop for the glance eye is registered and has not been run yet. An end-to-end test of the glance eye on 8-bit weights finished after this snapshot was audited and will be reported in the next document. The final choice of eye waits for the optimised-kernel timing and the second-crop test. A question-first test of the locator is also planned.
