# 26. An eye that survives 8-bit weights, and a second look that missed its bar

## What this is

Snapshot dated 6 November 2026. This document reports two findings from one day of tests on Qwen3.5-0.8B, the small open language model with a vision part that documents 24 and 25 use. It is released under Apache-2.0 (**PUBLISHED**: the model card and the LICENSE file in the release).

Both findings use the glance eye of document 25 (its finding 3). Document 25 had said that the 8-bit test of this eye would be reported next, and that the second-crop test had no result yet.

Each test had a written rule with fixed bars, fingerprinted (a checksum and a time) before its script ran. The picture tests use document 25's synthetic target, a small coloured square on a photograph. Paired tests are exact McNemar tests, which count the pictures where one arm is right and the other is wrong. Treat every result as an indication.

Labels used here. **FACT**: measured by our runs, with setting and sample size. **COMPUTED**: arithmetic on measured or stated numbers. **PUBLISHED**: a named published source, not re-checked here. **GUESS**: our reasoning, or anything beyond the tests. Where a claim could take two labels, we use the more cautious one.

## Summary of the two findings

| Finding | Verdict, as the rules read it | Main number |
|---|---|---|
| 1. Glance eye on 8-bit weights | EYE SURVIVES 8 BITS, at both glance sizes | 0.950 at 8 bits, as at 16 bits. At the 390-token glance, 0.996 at 8 bits against 0.983 at 16 bits, but the probe setting also changed, so that gain cannot be split between rounding and probe choice |
| 2. Second crop behind a confidence gate | NOTHING TO GATE in both runs; the 0.03 bar was missed in both | Best arm gained 14 pictures against 14.4 needed (always two; the gate 12); replication: always two 12 pictures, the gate 10 |

The verdicts are the rules' own words. A missed bar is reported as a miss, even when it is close.

## 1. The glance eye survives 8-bit weights

**What we asked.** The glance eye works in three steps. A cheap look at the whole picture (90 or 390 image tokens, the units a picture costs the model to read) finds the rough place. A small learned probe on the model's features guesses where the square is. One 256-token crop there is read for its colour. Does it still work with every weight stored in 8 bits instead of 16?

**What we did (pre-registered).** We repeated the earlier 16-bit glance test on the same 240 test pictures, with the same sizes, prompts, seeds and timing. One thing changed: every weight in six parts of the model was rounded to 8 bits and stored back as 16-bit numbers. This simulates 8-bit storage, so it measures accuracy only. The probe was refitted on the 8-bit model (trained on 1,200 pictures; settings chosen on 240 validation pictures, and listed in the table). The bars were fixed in advance. At 90 tokens: the square found in at least 0.903 of pictures, the colour right in at least 0.920, and no-square controls below 0.20. At 390 tokens: the find and the colour at least 0.953, controls below 0.20.

**What we found.** A text check came first. After rounding, the surprise on a 2,000-token book passage rose by 0.10 percent, from 3.03363 to 3.03666 nats per token (lower is better). That is inside the accepted range of 0.07 to 0.13 percent (**FACT**).

With every weight at 8 bits, the 90-token glance named the colour of a 40-pixel square right in 228 of 240 pictures (0.950), as at 16 bits. At the 390-token glance the 8-bit version scored 0.996 against 0.983, but its probe setting also changed (ridge strength 0.01 at 8 bits against 0.001 at 16 bits), so that difference cannot be split between rounding and probe choice. Every bar was met, so the eye survives 8 bits at both glance sizes (**FACT**).

| Measure (240 test pictures, 40-pixel square) | 90 tokens, 16-bit | 90 tokens, 8-bit | 390 tokens, 16-bit | 390 tokens, 8-bit |
|---|---|---|---|---|
| Probe layer, ridge strength (lambda) | 24, 0.01 | 24, 0.01 | 16, 0.001 | 16, 0.01 |
| Crop held the square | 0.933 | 0.938 | 0.983 | 1.000 |
| Colour named right | 0.950 | 0.950 | 0.983 | 0.996 |
| Colour, 95% Wilson interval | 0.915 to 0.971 | 0.915 to 0.971 | 0.958 to 0.994 | 0.977 to 0.999 |
| No-square controls: share naming the colour a square would have had (chance 0.125) | 0.133 | 0.138 | not measured | 0.100 |

All cells are **FACT**.

- The features moved only a little under rounding. For the 90-token glance, the similarity between 8-bit and 16-bit features of the same picture averaged 0.979 on training pictures, lowest 0.933. For the 390-token glance it averaged 0.966, lowest 0.926. That is why the probe was refitted (**FACT**).
- The rule's decision: store the eye's weights at 8 bits, about 0.90 GB instead of 1.75 GB (**FACT** for the decision; **COMPUTED** for the sizes, from the file sizes).
- On 24-pixel squares at 390 tokens, 8 bits scored higher (0.983 against 0.950). Two paired tests on these squares gave p = 0.012 and p = 0.021 (**COMPUTED** from the counts), among nine paired tests run with no correction. A lead to check, not a result (**FACT**).

**Caveats.**
- One synthetic target, 240 pictures per test. Differences below about 0.03 are not resolved (**FACT**).
- The rounding was simulated, so run-time speed and RAM use were not measured. The median times (155 to 189 ms per picture) came from slow reference code on a shared machine (**FACT**).
- Not tested: 4-bit storage, other glance sizes or other targets. The photos were not compared one by one with the earlier run, which stored no photo fingerprints. The code, picture selection and labels were identical. A same-picture check passed, but it shows the same random stream, not the same photos (**FACT**).
- Not yet run: the eye with 8-bit weights and document 25's trimmed 32,768-token vocabulary, about 0.68 GB (**COMPUTED**: 0.90 GB minus 220.7 million one-byte weights). It is a proposal, not in the rule file.

## 2. A second crop behind a confidence gate: nothing to gate, twice

**What we asked.** If the first crop is unsure, does a second crop help? Can the reader's own confidence decide when to take one? Document 25 asked this of SmolVLM2, a smaller vision model; here it is asked of the Qwen eye.

**What we did (pre-registered).** If the reader's confidence falls below a threshold (tau), the eye takes a second crop at the probe's best spot outside the first; the more confident look answers. Validation pictures (240) set the threshold. The bars were judged on 480 test pictures from photos that no earlier experiment had used. G0, the main bar: a gain of at least 0.03 over one crop, from always taking two or from the gate. G1: the gate keeps always-two's gain within 0.01, at most 1.30 crops per picture. G2: confidence is usable as a miss signal, meaning an AUROC (the area under the curve; 1.0 is perfect, 0.5 is chance) of at least 0.70 on at least 15 misses (the rule file sets the 15-miss minimum). G3: the gate beats a random gate at the same cost by at least 0.02. If G0 fails, G1 and G3 are reported but not interpreted.

**What we found.** On 480 test pictures, one crop named the colour right 0.942 of the time. Always taking two reached 0.971. The gate reached 0.967 and took a second crop in only 24 pictures. The main bar asked for 14.4 pictures of gain out of 480 (**COMPUTED**: 0.03 times 480). The best arm gained 14 pictures against 14.4 needed (always two; the gate gained 12). The bar was missed (**FACT**).

| Set (pictures) | One crop | Always two | Gate, tau 0.5 (crops per picture) |
|---|---|---|---|
| Validation (240) | 0.942 | 0.975 | 0.983 (1.054) |
| Test, both sets (480) | 0.942 | 0.971 | 0.967 (1.050) |
| Test-A (240) | 0.954 | 0.967 | 0.975 (1.042) |
| Test-B (240) | 0.929 | 0.975 | 0.958 (1.058) |

All cells are **FACT**.

- Missing the square was the main limit: the first crop held it in 445 of 480 test pictures (92.7%), and once it did, the colour was right 444 times out of 445 (**FACT**).
- Confidence was usable as a miss signal: AUROC 0.978 (95% interval 0.943 to 0.998) on the 35 test misses (**FACT**).
- Of those 35 misses, 8 still named the right colour (23%). The answer and label mix alone predicts about 6 (**COMPUTED** by a permutation test, which reshuffles the labels: p = 0.22 from 1,000,000 label permutations), so this is not shown to be above chance (**FACT**).
- On validation alone both arms would have cleared the bar: always two +0.033 (8 pictures) and the gate +0.042 (10 pictures), both **COMPUTED** from the stored counts. The rule judges the test sets, so that does not count (**FACT**).

**The replication.** The same rule was registered again, with the threshold fixed at 0.5 in advance and 480 new test pictures. One crop scored 0.933, always two 0.958 and the gate 0.954, at 1.05 crops per picture (**FACT**). The bar was missed again: always two gained 12 pictures and the gate 10, against 14.4 needed. G1 passed in both runs; because G0 failed, G1 is reported but not interpreted. G3 passed in the first run (margin 0.024 against 0.02) and failed in the replication (9.4 pictures against 9.6); because G0 failed, it too is reported but not interpreted. The registered reading stays NOTHING TO GATE, and the Qwen eye stays a single crop.

**Pooled, descriptive only (960 test pictures, both runs).** One crop 0.938, always two 0.965, the gate 0.960 (**FACT**). Looking twice helped in this count: the gate beat one crop on 23 pictures and lost on 1 (exact McNemar p = 0.000003, **COMPUTED**). But the gate did not beat always two: 8 wins against 12 losses (exact p = 0.50, **COMPUTED**). The registered reading stays NOTHING TO GATE, and this count does not change it.

**A lesson about setting bars (it changes no reading).** The 0.03 bar came from gate rules written for SmolVLM2, which scored 0.885 and so had 0.115 of headroom, the gap to a perfect score (**FACT**). The Qwen eye's single crop scored 0.942, leaving about 0.06 (**COMPUTED**), so 0.03 asked for about half of the available gain (**COMPUTED**). The headroom was foreseeable at registration, since the earlier glance test had measured 0.950 (**FACT**). Our advice (**GUESS**): set bars relative to headroom or cost, not as one number carried over from another model.

**Caveats.**
- A near miss on a pre-registered bar, reported as a miss. The first run's test gains (+0.025 to +0.029 on its test sets) look like the SmolVLM2 gain of document 25 (+0.048 on test), but the rule was not met (**FACT**).
- One target, one reader, one probe. Partial visibility and colour-dependent misses were seen, not controlled. White squares were missed more often by the first look: the square was held in 135 of 160 pictures (0.84), against 0.91 to 0.96 for the other colours, over 1,200 pictures (**FACT**). Of the 72 first-crop misses across both runs, 3 had part of the square inside the crop (all white, all named right). Across all 72, the right colour was named 16 times, against about 13 expected from the answer and label mix alone (permutation p = 0.17, **COMPUTED**) (**FACT**).
- Cost: in the first run one crop took 156 ms per picture and the gate 157 ms. In the replication the median was 164 ms for both arms. These are on slow reference code (**FACT**).

## 3. How the program works

- **Rules first.** Each rule file, with its bars, was fingerprinted (a checksum and a time) before its script ran. Thresholds were not changed after results. One addendum to the gate rules changed how a result was read, not the bars. An addendum to the eye rules defined the control set for the 390-token glance, because the earlier test had measured controls at 90 tokens only (**FACT**).
- **Later registrations, stated.** The replication was registered after the first verdict, with its threshold fixed at 0.5 in advance (**FACT**).
- **Scheduling and debugging.** In the eye test, the test job was queued before the extraction it depends on had finished. A check that the extraction was complete passed before any result was written. That departed from one addendum's ordering, but no number depends on it. Debug passes gave numbers that are not results (**FACT**).
- **Code review.** Our code review of these scripts finished on 6 November 2026. It rated the eye test and the second-crop test, with its replication, at 0 HIGH and 2 MEDIUM each, and every number was recomputed from the stored files. For the eye test its two points were: the same-picture check shows the same random stream, not the same photos; and the 390-token probe setting differs (0.01 against 0.001), so the 390-token gain cannot be split between rounding and probe choice. For the second-crop test, the helper modules and the exclusion of photos by raw bytes could not be checked in the review copy; this affects no number (**FACT**).
- **Audits.** An AI audit pass recomputed the headline numbers, table cells, p-values and bars of our two lab notes: 20 findings, one a wrong printed digit that changed no reading. All corrections were applied before this write-up (**FACT**). Nothing here has been peer reviewed.
- **Wording corrected after audit.** The pooled look-twice result is described as descriptive only, and the share of misses that still named the right colour (23%) is described as not shown to be above chance.

## 4. Not claimed

- We do not claim a general ability to see.
- We do not claim that the 8-bit eye is faster or smaller at run time (0.90 GB is arithmetic on file sizes), or that 4-bit storage works.
- We do not claim that the eye meets the 150 ms time bar of the original glance test (document 25). The second-crop test measured 156 ms for one crop, and the 8-bit run 158 ms in simulation.
- We do not claim that a second crop helps the Qwen eye under the pre-registered rule, or that the gate beats always looking twice. The gap was 4 pictures in 960 (**COMPUTED**), and it was not resolved.
- We do not claim that the second look works on natural pictures, with other readers or targets, or that the reader's misses are explained.
- We do not claim anything about always-on running or learning while running. Neither was tested.

## 5. What this means for a small always-on eye

What the results support (**FACT** for the readings):

- The glance eye kept its colour accuracy with every weight at 8 bits, on the one target tested. An 8-bit store would take about 0.90 GB instead of 1.75 GB (**COMPUTED**). Speed was not measured.
- A second look did not clear the bar for this eye in either run, although confidence was usable as a miss signal. Always looking twice scored higher in the pooled count, but used about 2 crops per picture against 1.05 for the gate (**COMPUTED**).

Beyond the tests (**GUESS**):

- A small always-on system might combine the 8-bit eye and a single crop (a second look only when cheap). These were not run together.
- The bigger remaining weakness looks like finding the square, not reading its colour. The first crop missed 53 of 720 validation and test pictures (**FACT**; 7.4% is **COMPUTED**), while on test the colour was right in 444 of the 445 pictures where the square was found (**FACT**). The next gains may come from the locating step.
