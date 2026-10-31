# Learning while reading: the reader experiments

## Summary

We took a copy of our 143M-parameter model and let it keep learning while it reads: after scoring each short stretch of text it takes one small gradient step on that stretch. On whole books this lowered the loss on the text still to come by **2.4 to 4.6 percent**, in every tenth of every book, and the gain grew the further the model read. A control that took its steps from a different book made the same book **worse**, which is consistent with the gain coming from the text being read (one control, on two books). The price is a slow drift of the loss on ordinary web text; mixing in a small share of real training text (replay) removed most of it (+1.39 percent drift down to +0.36 percent). Revisiting the most surprising ten percent of segments made the weights keep those segments better (7.9 against 6.5 percent) but did not help prediction of later text. About 60 percent of what is learned about a book survives reading one or two more books. Results on other kinds of data (web pages, Wikipedia, fiction, science, math, code) are in section 8. A first quiz-based measurement of what the model knows after reading is in section 9: it was too easy to show any change in accuracy (its margins hint at a small book-specific gain), so we do not yet know whether the lower loss means knowledge of the book. A second, harder quiz found a small extra gain in confidence on the part of the book that had been read but no clear gain in accuracy, which is inconclusive by our rule. A test of dictionary cards in front of a sentence (section 10) failed its pre-written rule. Section 12 reports a longer run: the loop was stable over 22 bursts on a private collection of web novels and song lyrics, with large gains on parts of each book that were never read; the replay-only control for that run was still running when this was written, so those gains are a hint, not a result.

All numbers below are **FACT** from our own logged runs unless marked otherwise. Setting for every run: the same 143M model, one run per setting (no repeated seeds: the procedure is deterministic given the text up to small numerical noise, about 0.0005 nats in our trainer; the differences we interpret are large against the block-to-block spread we report, but several small differences, such as 0.2 points between safeguards, rest on single runs).

## 1. Setup

- **Model and precision.** A snapshot of the main training run (session 34 for sections 2 to 6, about 1.7 billion training tokens), evaluated in 32-bit floating point because the model's inner activations exceed the range of 16-bit numbers.
- **Reading loop.** The text is cut into segments of 128 tokens. Each segment is **scored first**, with the weights that have seen only the earlier text and up to 896 tokens of context in front of it; then the model takes one gradient step on that segment's own loss. A static baseline scores exactly the same segments with the unchanged weights. The gain is the baseline loss minus the reading loss, averaged over the scored tokens (the first segment, scored before any update, is left out).
- **The step.** For every weight tensor the update is a fixed fraction (the "relative step", written eta) of the tensor's own typical size, in the direction of the normalised gradient. It is applied to every tensor except the input-embedding table and the value-embedding tables (about 63 million parameters); the other 80 million (the hidden matrices, the output head and the scalars) are updated.
- **Drift monitor ("canary").** Sixteen fixed windows of ordinary web text (never trained on), scored with the current weights every 100 segments.
- **Speed.** About 0.17 seconds per update on a consumer 8 GB graphics card.
- **Text.** Public-domain English books (long and coherent; a search found no exact 12-token phrase of the three main books in our training data, which is not proof that they were never in it) for sections 2 to 7; held-out text for section 8 (validation shards of our data mixes, science papers that are in neither training pool, and code and math pages disjoint from the training text).
- **Code review.** A separate AI reviewer checked the reading loop for leaks and wrong numbers (section 11).

## 2. Finding the step size (tuning book, first 30,000 tokens)

| relative step | gain on the rest of the text | blocks improved (of 10) | change of the canary at the end |
|---|---|---|---|
| 3e-6 | 0.54 % | 10 | -0.12 % |
| 1e-5 | 1.20 % | 10 | -0.05 % |
| 3e-5 | 1.86 % | 10 | +0.51 % |
| 1e-4 | 1.54 % | 10 | +2.97 % |

The best step is about 3e-5, but it already moves the canary; at 1e-4 the gain falls and the canary drifts by 3 percent. An earlier pilot of ours with three updates per window found a much narrower usable range (three times the best step already lost), so the usable window is far wider when there are many small steps.

## 3. Whole books (two books of about 120,000 tokens each)

| book | relative step | gain | canary drift at the end |
|---|---|---|---|
| Coral Island | 1e-5 | 2.35 % | +0.18 % |
| Black Arrow | 1e-5 | 3.50 % | +0.50 % |
| Coral Island | 2e-5 | 2.88 % | +0.78 % |
| Black Arrow | 2e-5 | 4.25 % | +1.27 % |
| Coral Island | 3e-5 | 3.09 % | +1.39 % |
| Black Arrow | 3e-5 | 4.60 % | +1.95 % |

Every tenth of every book improved. The gain **grows with reading**: for the small step on Coral Island the ten block gains rise from 0.022 to 0.114 nats per token (first to last tenth); on Black Arrow from 0.051 to 0.191. The older, harder text (Black Arrow) gains more. (PUBLISHED, for scale: 1.3 to 2.0 percent for 150M to 1B models, Rannen-Triki and colleagues 2024, arXiv 2403.01518; our pilot with three updates gave 0.67 percent.)

**Control.** Scoring Coral Island while taking the updates from the same-numbered segments of a different book (step 3e-5): **-1.27 percent**, no block improved, canary +2.1 percent. The same step with the book's own text gave +3.09 percent. A reviewer had warned that a same-genre foreign book might still help; at book length it did not.

## 4. Safeguards against drift (Coral Island, step 3e-5)

| safeguard | gain | canary drift |
|---|---|---|
| none | 3.09 % | +1.39 % |
| pull the weights toward their starting values by 3e-4 per step | 3.04 % | +1.13 % |
| pull-back 1e-3 per step | 2.91 % | +0.73 % |
| **replay: a real 1,024-token training window takes a step after every 8th segment** | **3.11 %** | **+0.36 %** |
| (for comparison) a smaller step, 2e-5, nothing else | 2.88 % | +0.78 % |

Replay is the best safeguard we found: it keeps the whole gain of the large step and cuts the drift to a quarter, for about 12 percent more compute. Pull-back buys about what a smaller step buys. (GUESS on why: replay keeps the weights consistent with the old data distribution instead of merely anchoring them near a point.)

## 5. Carrying the weights from book to book, and how much is kept

Without a safeguard, step 3e-5, two books in a row: the second book gains 4.81 percent (4.60 alone, so carried weights helped a little) but the canary drift reaches +2.5 percent after two books.

Three books in a row at the small step 1e-5, no safeguard (Coral Island, Black Arrow, Hans Brinker):

| book | gain while reading | canary at the end | right after reading it | after the later books | share of the gain still kept |
|---|---|---|---|---|---|
| Coral Island | 2.35 % | +0.19 % | 3.2618 (baseline 3.4002) | 3.3193 (two later books) | 58 % |
| Black Arrow | 3.83 % | +0.66 % | 3.4825 (baseline 3.6950) | 3.5582 (one later book) | 64 % |
| Hans Brinker | 2.77 % | +0.65 % | 3.5253 (baseline 3.6926) | last book | 100 % |

At the small step the drift on ordinary text levels off (+0.66 then +0.65 percent), but the weights forget roughly 36 to 42 percent of what they learned about a book once one or two other books follow.

## 6. Revisiting surprising spans (Coral Island, step 3e-5 with replay)

A revisit is one more gradient step on a segment already read. Ten percent of the segments are flagged: "hard" means the segment's surprise is above the 90th percentile of the previous 100 segments (using only text already read); "random" flags the same share by chance. After the book, the hardest tenth of every page (judged by the unchanged model) and a random tenth are scored again with the final weights to see what the weights keep.

| policy | gain on later text | canary drift | hard spans: kept gain | random spans: kept gain |
|---|---|---|---|---|
| none | 3.11 % | +0.36 % | 6.53 % | 5.60 % |
| revisit hard segments at once (back to back) | 3.12 % | +0.56 % | 7.86 % | 5.64 % |
| revisit hard segments 50 segments later (spaced) | 3.14 % | +0.53 % | 7.91 % | 5.71 % |
| revisit random segments 50 segments later | 3.13 % | +0.48 % | 6.73 % | 5.75 % |

Revisiting does **not** change how well the model predicts the text that follows. Revisiting the **surprising** segments makes the weights keep exactly those segments better (+1.4 points; one book, one run); revisiting the same number of random segments does little. Spaced and back-to-back revisits are indistinguishable at a 50-segment gap (we have not tested the 1,000-step gap that the memory literature would suggest). So at this scale a revisit works as a memory strengthener, not as extra understanding. Cost: about 10 percent more steps and 0.1 to 0.2 points of extra drift.

## 7. What the failed and finished side tests say

- **Pull-back and a smaller step are interchangeable; replay is not.** (section 4)
- **Easy-text-first curricula did not help** in the separate small-scale curriculum test (document 09), which is why the reader keeps reading real text on top of the finished model rather than starting from children's books.

## 8. Other kinds of data (held-out shards of our data mixes)

The books above were chosen because they are long and clean. The reader is meant to read any data we have, so we repeated the measurement on eight kinds of text. Each stream is 80,000 tokens of **held-out validation text that the model never trained on**, read in file order (documents separated by end-of-text markers, so contexts cross document borders as in training); the weights are reset between streams; step 3e-5 **with replay** after every 8th segment; the same canary (static 3.6334).

| kind of data | baseline loss (nats per token) | gain on the rest of the stream | blocks improved (of 10) | gain in the first and last tenth (nats) | canary change at the end | random tenth of segments re-scored with the final weights |
|---|---|---|---|---|---|---|
| code (GitHub) | 1.510 | **13.91 %** | 10 | 0.099 and 0.154 | +0.99 % | 19.00 % lower |
| science papers | 2.100 | **9.85 %** | 10 | 0.145 and 0.172 | +0.43 % | 13.95 % lower |
| fiction (held-out books) | 2.924 | 3.91 % | 10 | 0.035 and 0.125 | +0.48 % | 7.16 % lower |
| mathematics (web pages) | 2.821 | 3.43 % | 10 | 0.049 and 0.129 | +0.28 % | 6.19 % lower |
| education pages | 2.972 | 2.97 % | 10 | 0.075 and 0.095 | +0.49 % | 6.52 % lower |
| Wikipedia | 2.832 | 1.59 % | 10 | 0.020 and 0.130 | +0.06 % | 4.53 % lower |
| web pages (FineWeb-Edu) | 3.485 | 0.89 % | 10 | 0.038 and 0.035 | -0.27 % | 3.75 % lower |
| web pages (DCLM) | 3.708 | 0.76 % | 10 | 0.012 and 0.022 | -0.08 % | 3.20 % lower |

Reading (numbers FACT, interpretation GUESS):
1. **Learning while reading helps on every kind of data we tried**, in every tenth of every stream, but by very different amounts: 13.9 percent on code and 9.9 percent on science papers, down to under 1 percent on ordinary web pages.
2. The gain is largest where the text **repeats its own names, terms and patterns** (a code file, a paper, a novel) and smallest where every page is a new topic (a stream of unrelated web pages). On the FineWeb-Edu web stream the gain does not grow along the stream (first and last tenth about equal), on the DCLM web stream it rises slightly (0.012 to 0.022 nats), and on the others it grows clearly.
3. With replay, the loss on the ordinary-web-text canary stayed between -0.3 and +1.0 percent in all eight runs, even after 80,000 tokens of code (+0.99 percent, the largest).
4. Caveats: one run per stream, 80,000 tokens each, no foreign-text control for these streams, and the code and science streams have much lower baseline loss, so a given absolute gain is a larger percentage.

## 9. Quiz measurement of what the model knows after reading

A lower loss on the text (sections 2 to 8) does not by itself say that the model has learned anything about the book. To look for that, we built a multiple-choice quiz for two books and asked it after reading.

**The quiz (FACT).** Two public-domain children's novels, *The Secret Garden* and *Treasure Island*. For each, a fixed program (no model involved, same output on every run) cut 300 passages of two consecutive sentences, blanked one word of 5 to 10 letters that occurs at least twice in the book, and offered four options: the answer and three other words from the same book of similar length and frequency. Half of the passages come from the first 60 percent of the book (the "read" half) and half from the last 40 percent (the "unread" half, until the model has read it). The model answers by putting each option into the blank and picking the one that gives the whole passage the highest probability; chance is 25 percent. The **margin** is the log-probability of the true option minus the average of the other three (nats; larger means more confident). Passages close to the 60 percent cut are dropped, leaving 139 to 143 per half.

**The procedure (FACT).** The session-36 snapshot (about 1.85 billion training tokens, fp32). Quiz the unchanged model ("base"); read the first 60 percent of the book with learning on (relative step 3e-5, replay after every 8th segment, as in section 4); quiz again; read the remaining 40 percent; quiz a third time. One run per book. A third run on *The Secret Garden* added spaced revisits of the hardest spans (47 extra steps in the first part, 32 in the second).

| book and stage | read half: accuracy | margin | unread half: accuracy | margin | canary (static value 3.606) |
|---|---|---|---|---|---|
| Secret Garden, base | 89.5 % (±2.6) | +10.22 | 85.1 % (±3.0) | +10.48 | 3.606 |
| after the first 60 % | 90.2 % | +10.14 | 85.8 % | +10.07 | 3.654 (+1.3 %) |
| after the whole book | 89.5 % | +9.99 | 87.9 % | +10.28 | 3.640 (+0.9 %) |
| Treasure Island, base | 85.3 % (±3.0) | +9.02 | 81.3 % (±3.3) | +8.28 | 3.606 |
| after the first 60 % | 86.0 % | +8.99 | 83.5 % | +7.81 | 3.626 (+0.6 %) |
| after the whole book | 85.3 % | +8.79 | 87.1 % | +8.14 | 3.633 (+0.7 %) |
| Secret Garden with spaced revisits, after the first 60 % | 90.2 % | +10.15 | 86.5 % | +10.01 | 3.662 (+1.5 %) |
| the same, after the whole book | 89.5 % | +10.02 | 88.7 % | +10.28 | 3.646 (+1.1 %) |

The figures in brackets are one standard error of the accuracy. The canary here uses a different set of web windows from the one in sections 3 to 8 (its unchanged value is 3.606, not 3.633), so its drift is not directly comparable with those sections.

Items whose answer changed (FACT, same items before and after; wrong-to-right against right-to-wrong):

| | read half, after first 60 % | read half, after whole book | unread half, after first 60 % | unread half, after whole book |
|---|---|---|---|---|
| Secret Garden | 3 against 2 | 2 against 2 | 5 against 4 | 6 against 2 |
| Treasure Island | 2 against 1 | 1 against 1 | 5 against 2 | 8 against 0 |

(All changes are counted against the unchanged model. Each half has about 140 items.)

Reading (numbers FACT, interpretation GUESS):
1. **The quiz is easy for the unchanged model:** 81 to 90 percent right at the start, against 25 percent by chance. That leaves little room, and one cell has a standard error of 2.5 to 3.3 points.
2. **Reading the first 60 percent did not raise accuracy on that part beyond noise** (+0.7 points in both books, one item net), and it did not raise the margin either (it fell by 0.03 to 0.08 nats; on the unread half it fell by 0.40 and 0.47 nats, see point 4).
3. **The unread half rose once it had been read:** +2.8 points (Secret Garden) and +5.8 points (Treasure Island) from the start to the end of the book. In Treasure Island 8 items went from wrong to right and none the other way, which would be unlikely by chance alone (a sign test gives about 0.008). But the same half had already gained a little (+0.7 and +2.2 points) after the model had read only the other 60 percent, and the read half never gained. So the late rise may be a general adaptation to the language of the book rather than memory of particular passages (GUESS).
4. **The margins did not rise;** they fell by 0.03 to 0.5 nats (of about 10) in every cell, compared with the unchanged model. GUESS: the three wrong options are words from the same book, so reading makes the book's words more likely in general, including the wrong options. **But the fall was smaller on the half that had been read:** after the first 60 percent the read halves fell by 0.08 and 0.03 nats and the unread halves by 0.40 and 0.47, a difference of +0.32 and +0.44 nats in favour of the read half. That is the pattern expected if reading teaches something about the passages read. We had set no rule for it on this quiz, so it counts as a hint, and the second quiz below tests it under a rule.
5. **Spaced revisits of the hardest spans changed nothing visible** in this quiz (read half 89.5 percent at the end in both runs; unread half 88.7 against 87.9 percent, one item).
6. **Cost:** the canary rose by 0.6 to 1.5 percent after the first part and by 0.7 to 1.1 percent at the end of the book, two to three times the +0.36 percent of section 4. The books and the canary windows differ, so this is not a clean comparison.

**What this does and does not show.** The loss gains of section 3 (2 to 5 percent) do not appear as a visible change in accuracy on this quiz, and only as a small hint in the margins (point 4). That does not show that the model learned nothing about the book: the quiz is too easy to start with, the wrong options come from the same book (so learning the book raises them too), many items can probably be solved from grammar alone (GUESS, not measured), and each cell has only about 140 items. It does show that this form of the quiz is not sensitive enough to serve as the "thermometer" of document 01 for this kind of reading. A harder version is needed: wrong options that fit the sentence, questions about names and events that cannot be guessed from grammar, and more items. A second, harder quiz was then built and run (next subsection).

**A harder quiz (second test).** For each book, 160 four-option questions whose three wrong options are words of the same book that the unchanged model itself ranks highly for the blank (ranks 2 to 25), so they fit the sentence. The unchanged model answers 43 percent (The Secret Garden) and 42 percent (Treasure Island) of such questions right by likelihood, chance being 25 percent (calibration on 60 questions per book). The procedure is the one above (session-36 snapshot, step 3e-5 with replay after every 8th segment; quiz before reading, after the first 60 percent and after the whole book). After the questions near the 60 percent cut were dropped, 76 read-half and 75 unread-half questions remained in one book and 75 and 74 in the other. The rule was written before the run: the primary statistic D is the margin gain of the read half minus the margin gain of the unread half after reading the first 60 percent, pooled over both books (a 95 percent bootstrap interval over questions); PASS needs the interval above zero, D at least +0.30 nats and at least +5 points of accuracy; KILL if the upper end of the interval is below +0.15 nats. (A first analysis stopped with a void verdict because my own threshold of 150 scored questions per book ignored the dropped ones, 151 and 149 remained. The threshold was changed to 140 in a new rule file before any statistic had been computed, and the same data were re-analysed.)

| | Read half | Unread half | Difference |
|---|---|---|---|
| Margin gain after the first 60 percent, Secret Garden (nats) | +0.67 | +0.44 | +0.23 |
| Margin gain after the first 60 percent, Treasure Island (nats) | +0.70 | +0.14 | +0.57 |
| **Pooled difference D (nats), 95% interval** | | | **+0.40 (+0.16 to +0.64)** |
| Accuracy difference, pooled (points), 95% interval | | | +2.6 (-4.8 to +9.9) |
| Difference after the whole book (nats), 95% interval | | | -0.01 (-0.25 to +0.24) |

Accuracy before reading was 46 and 37 percent on the read halves and 49 and 45 percent on the unread halves of the two books. **Verdict by the rule written beforehand: inconclusive.** The margin part passes (interval above zero, D above +0.30) but the accuracy part does not (+2.6 points against the +5 required).

Reading (numbers FACT, interpretation GUESS):
1. On the harder quiz, reading the first 60 percent raised the model's confidence in the true answer more on the part it had read than on the part it had not, in both books (+0.23 and +0.57 nats). That is the pattern expected if reading teaches facts of the book. The easy quiz above shows the same sign and a similar size in its margins (+0.32 and +0.44 nats); both quizzes score the weights of the same reading procedure, so they are two samples of questions about one reading, not two independent readings.
2. The effect is small next to margins of about 3 nats and did not show up clearly in accuracy: with about 75 questions per half, the accuracy interval is plus or minus 7 points.
3. After the whole book the difference disappears (-0.01), as it should once both halves have been read.
4. Not decisive: one run per book, two books, and a bank in which some questions still have a second option that fits (the helper judged 13 of 20 sampled questions to have one, GUESS). Settling it needs more questions, more books and a repeat with a different reading order.

## 10. Dictionary cards in front of a sentence (FACT)

**Question.** Does the dictionary entry (a "card") of a rare word, put in front of a sentence that uses the word, help the model predict that word? And do three learning steps on card plus sentence teach the word better than three steps with the card of a different word? **Setting.** Fifty rare words, each with a one-line dictionary entry, four training sentences and four held-out sentences from other books; the session-36 snapshot, 32-bit arithmetic, three relative steps of 3e-5 for the learning arms (loss on the sentence tokens only). The measure is the summed log-probability of the target word's own tokens in each held-out sentence (higher is better). The control for each word is the card of a different word of about the same training frequency; everything is paired over the words (bootstrap over the 50 words). The pass and kill rules were written before any number: for each of the two contrasts, a 5 percent lower loss on the target word, an interval above zero and at least 60 percent of the words improved; a contrast whose interval stays below +0.05 nats is killed.

| Arm | What happens | Mean target log-probability (nats) |
|---|---|---|
| S | no card, no learning | -16.45 |
| K | the word's own card in front, no learning | -8.96 |
| KC | another word's card in front, no learning | -16.02 |
| L | three learning steps on own card plus one training sentence, then the held-out sentence without a card | -14.26 |
| LC | the same with the other word's card | -12.83 |

| Contrast | Mean difference (nats) | 95% interval | Words improved | Verdict |
|---|---|---|---|---|
| In context: K minus KC | +7.06 | +6.30 to +7.83 | 50 of 50 | pass |
| After learning: L minus LC | -1.43 | -1.72 to -1.17 | 0 of 50 | **killed** |

By the rule written beforehand the verdict is **KILL**: the learning contrast failed. All correctness checks passed (the unchanged-model score equals an independent forward pass within 1e-6 nats, the weights are restored exactly after every word, nothing after the target word changes what comes before it).

Reading (numbers FACT, interpretation GUESS):
1. **In context the card helps a great deal (about 7 nats), but the card contains the word, so this is mostly a copy effect.** It is cheap and expected and it does not show learning.
2. **Learning steps on the sentence raise the word by 2.2 to 3.6 nats whatever the card** (L minus S, LC minus S): that is exposure to the word in the training sentence.
3. **The right card made the learning worse than another word's card, in all 50 words.** GUESS: with its own card in front, the word is already predictable in the training sentence, so the update has little to learn about the word; with another word's card it is a surprise and the update teaches it. The whole-sentence loss agrees (the own-card arm is worse by 0.05 nats per token in all 50 words).
4. So, as designed (card as context, loss on the sentence only), the look-up does not help learning and is switched off. A design in which the model also learns the card text, with a control that gives the word the same number of exposures, is a different test and needs its own rules.

**Limits.** One snapshot, one training sentence per word, three steps at one step size, first-sense dictionary glosses (15 of the 50 cards describe the base word of an inflected form), frequency matched by training count only. The first two launches on the graphics card stopped in a correctness check that only works on the CPU; they produced no result and were not read.

## 11. Review of the method

A separate reviewer re-read the reading loop and ran checks on a CPU: (a) **no leak**: changing every token after a segment changes neither that segment's score nor any earlier one (bit-identical), with a positive control showing the test can fail; (b) the update rule is identical to our earlier pilot's; (c) the baseline always uses the unchanged weights, also when weights are carried from book to book; (d) a CPU run reproduced the graphics-card run (gain 0.0059639 nats in both, static and dynamic losses within 1.1e-5); (e) no exact 12-token phrase of the three books was found in our training data (27 searches; a retyped copy would not match, so this is not proof of absence). One conversion (bits per byte) was wrong for truncated books and was fixed; none of the numbers on this page used it.

## 12. A longer run: stability, a replay control, and a hint

All numbers in this section are **FACT** from our logs unless marked. The loop is the one described in sections 1 to 4: a step after each 128-token segment, a replay window after every eighth segment, and a rollback to the last good weights if the canary (the drift monitor on ordinary web windows) rises by more than 2 percent. Its pass and stop rules were written before the first run.

**Stability on validation text.** Three bursts (about an hour, 18,303 segments, 306 visits of 100 segments) passed the stability rule: zero rollbacks and no alarm. The canary at the end of each burst was -0.80, -0.86 and -0.55 percent (limit +1.0). The paired gain over the frozen base was +0.122 nats per visit (interval +0.110 to +0.134; 293 of 306 visits positive).

**Replay-only control on validation text.** For the first 6,104 segments, the same segments and the same replay windows were run with the reading updates switched off. Replay alone moved the canary more than the loop did (-1.07 against -0.80 percent). Reading added a gain on the next text of the same stream: about +7.6 points on science, +6.2 on code, +2.1 on math and +0.3 on the mixed web pool. That gain did not show on held-out windows of the same domain that were never read: the change there was between -0.4 and +0.7 points. So at this scale, reading helps with the text just read, in the way a cache does, and does not yet give a lasting gain across the domain. The stability claim stands, and the earlier wording that "the loop is learning" is narrowed accordingly (document 12).

**A longer run on a private collection of web novels and song lyrics.** The loop read one full pass of five books and one set of song lyrics, each read once: 22 bursts and 130,065 segments. It ran with zero rollbacks and no alarm. The final canary change was +0.74 percent and never went above +1.15 percent. This shows stability over one pass, with one seed. It does not show stability over weeks.

On the parts of each book that were never read, the live model was compared with the frozen base:

| Text | Gain on never-read parts |
|---|---|
| Five books (one each) | +17.4, +10.1, +7.9, +5.7 and +4.1 percent |
| Song lyrics | -0.9 percent |

**Status: a hint, not a result.** The gains on unread parts are large, and unlike the validation run they are not local to the text just read. But the replay-only control for this run (same stream, replay steps only, no reading updates) had been started and had not finished when this was written. Replay alone could explain part or all of the gain, each book was read only once, and there is one seed. The song lyrics got slightly worse.

The rule for the control was written before it ran. Reading counts as learning the book if the gain on unread parts minus the control's gain is at least +2.0 points on at least 4 of the 5 books. Replay explains the result if that difference is below +1.0 points on at least 3 of the 5 books. Anything else is inconclusive.

## What this does not show

- That the reader is a better language model: the gains are measured on the text being read, the drift on other text is only partly controlled, and one run per setting was done.
- That lower loss means better understanding. Section 9's first quiz neither supports nor rules this out.
- Anything about much longer reading (days), larger models, or a different training recipe.
- That revisiting helps when the later task is answering questions instead of predicting text (the quiz of section 9 was too easy to show either way).

## Open questions

- Which mix of replay share, step size and pull-back keeps a model stable over weeks of continuous reading?
- Does a longer gap between revisits, or revisiting whole passages, improve what is kept?
- Does the gain on unread parts of a book survive the replay-only control on the longer run (section 12, control still running)?
- Does the gain survive on data the model has partly seen in training, and on code and mathematics?
- Can the model itself decide what to revisit better than the surprise rule?
