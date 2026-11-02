# Method and honesty log

## Summary

This project is run by one person and an AI assistant on two consumer graphics cards. That makes it fast and cheap, and it also makes it easy to fool ourselves. This document describes the habits we use against that, and keeps a list of the times they caught us being wrong. The list is part of the research: a claim that survived a skeptic review is worth more than one that was never checked, and a claim that did not survive is worth knowing about.

Everything here is labelled the way the other documents are: **FACT** (measured in our own logged runs), **PUBLISHED** (taken from a paper or documentation that we opened) and **GUESS** (reasoning that was not tested).

## How experiments are run

1. **Short chained sessions.** Training is done in sessions of at most 25 minutes, each resumed from the last checkpoint and followed by a validation gate. A bad session can be stopped and retried without losing a day. (FACT: this is how the production run has been trained since 29 Oct 2026.)
2. **Rules before data.** For the larger decisions (data plan, weight decay, curriculum) the pass or fail rule, the tolerance and the possible outcomes were written down before the result existed. The rule is then applied as written, including when we dislike the answer. There were exceptions, listed here and below: a proposed reading of the data-plan rules was written while the first pairs were being measured (document 06), the rule file of the adaptation scan was rewritten after the first scan results (document 10), and the bar for the reading pilot was lowered after we found our own earlier pilot (see the table below). Example: the curriculum test (document 09) had a tolerance of 0.004 bits per byte or twice the seed-to-seed spread, whichever was larger; the result was a clear "harm" and we accepted it.
3. **Controls first.** A new environment, tool or metric gets a control run before any result from it is believed. The control caught a biased evaluation on one hardware type (see the log below).
4. **Paired comparisons.** When two models or two settings are compared, the same text windows, the same segments or the same quiz items are used for both, so the difference is not swamped by differences between the texts.
5. **A second reader.** Important results are re-read by a separate AI reviewer instructed to be sceptical, which writes its own note (confirmed, narrowed, corrected, withdrawn). We read every such note before using the result.
6. **Predictions are logged.** When we guess an outcome before a run we write the guess down. Several guesses were wrong in both directions; some of the misses are listed below (document 07 counts 2 of 14 plan predictions met, and the absolute matrix-size prediction in document 05 failed).
7. **Labels on every claim.** FACT, PUBLISHED or GUESS. A number without its setting (model size, tokens, seeds, number of windows) is treated as a defect.

## Claims we had to correct

| What we said | What was wrong | Found by | Effect |
|---|---|---|---|
| A first round of small-scale weight-decay tests (FACT: round 1, 30 Oct) showed a large difference between settings | The evaluation on one cloud GPU type was biased by about +0.078 bits per byte because of a compiled evaluation graph; training losses were close to the home machine's (3.564 against 3.550 at step 500) | A control run of the same setting on both machines | Round 1 was thrown away; the tests were rerun with eager evaluation and the control passed (1.4082 against 1.4085) |
| "Lower weight decay clearly helps; gains are about thirty times the noise" (an early weight-decay test) | The repeat run shared the random seed with the first, so it was not an independent repeat | A skeptic review | The claim was downgraded to "inconclusive"; the later, properly seeded tests found a flat optimum |
| "There is no small-model evidence for test-time learning" (first biology note) | There is published evidence at 150M to 1B parameters | A skeptic review (row B15 of that review) | The claim was removed and the published numbers were added |
| A planned pass bar of 3 percent for learning-while-reading on whole books | We had not noticed our own earlier pilot (0.67 percent after three updates) nor the published 1.3 to 2 percent | Reading our own notes before building | The bar was lowered to 1 percent; the later results (2.4 to 4.6 percent) cleared both |
| Bits per byte in the reading experiments | The conversion used the byte count of the whole book but the token count of the truncated text, so it was too small for truncated books (it was right for books that were not truncated) | A code review by a separate AI reviewer | No number we report was affected (all gains are given in nats and percent); the conversion was fixed |
| An audit of 41 summary statements from our first status reports | Of the 41: 4 wrong, 1 withdrawn, 10 too strong, 8 out of date, 12 missing a caveat, and 6 confirmed or fixed | A systematic claims table | Each statement was corrected or caveated |
| "The matrices grew by 2.24 times in two sessions, which is the equilibrium of the weight-decay balance model (the square root of five)" | The weights kept growing: 2.31 and 2.33 times after three and four sessions, about 4 percent above the prediction | A skeptic review of the weight-decay tests | The claim was withdrawn; the level is now stated as about 2.33 and still rising (GUESS), and the first-session jump and the time scale are still reported as predicted |
| An AI helper reported that our held-out evaluation text came from a different web corpus (DCLM-baseline) than the one our brief named (FineWeb-Edu) | The helper had followed the wrong data-preparation script. The files that built the validation set show it is a held-out part of the FineWeb-Edu sample; the other script builds a separate training folder | The main session checked the build scripts and the file dates, and compared a sample of documents | The comparison in document 04 carries the corrected statement, which matters because it makes overlap with SmolLM's training data likely |
| The reading loop is learning (note of 31 October, after the first stable burst) | Most of the canary gain came from the replay steps, not from reading. Reading added a gain only on the text just read, not on unread windows of the same domain (-0.4 to +0.7 points) | A replay-only control on the same segments and the same replay windows (document 08, section 12) | Narrowed to "adapts locally to what it reads"; lasting gains not shown |
| An interim status line quoting a reading gain of +0.097 nats for the first block | The line compared the mean loss over all segments of a visit with the mean over the sampled segments, which is not a paired comparison | A check of the event records against the burst summaries | The paired figure for that block is +0.034 nats. Verdicts use paired figures only |
| A misread figure for a published puzzle benchmark (a "5 percent to 29 percent" improvement for a 1B-parameter model) | It is probably 29 of 80 tasks (36 percent) on puzzles that also tuned the settings | A skeptic review | The figure was dropped from all designs |

## Errors in our own tooling (and the rule each one produced)

- A size filter in a monitoring tool silently ignored every new snapshot because the snapshot format had become smaller; a live view stayed on an old session for about half a day. Rule: when a file format changes, search every consumer for size and name filters.
- A results parser read "zero rows" from a log it could not parse and reported it as an empty result. Rule: the first time a parser meets a real output, check that it returns the rows you can see by eye, and make "zero rows from a non-empty file" a loud error.
- A script that rewrote a placeholder line also destroyed a documentation line that began with the same text. Rule: keep placeholder lines unique and syntax-check every generated script before running it.
- Restarting a live display component by pattern could terminate our own shell and left it dead; a new display line with an unprotected conversion blanked the whole panel. Rule: act on process numbers, test a copy in isolation, wrap new display code so it can only fail locally.
- An evaluation that is cheap to run in an unusual precision gave biased numbers on one chip. Rule: run the control arm on every new environment first.

## Predictions against outcomes (calibration)

| Prediction (written before) | Outcome |
|---|---|
| Test-time learning gain between 0.000 and 0.006 bits per byte, 25 percent chance of passing the pre-set bar (29 Oct pilot) | +0.008 bits per byte (0.67 percent): the pilot beat both |
| Before the pooled equal-token check of the two data plans (30 Oct) we gave: switch to the plain plan about 45 percent, keep the main plan about 30 percent, raise the fresh-data share about 25 percent | Outcome: keep the main plan, the less likely branch; the two retention ratios were 0.48 and 0.62, a little under the ranges we had predicted (0.56 to 0.69 and 0.70 to 0.89) but above the 0.4 threshold |

## Rules adopted on 31 October 2026

These are rules for new rule files. They are adopted but not yet tested on our own loops, so they are GUESS-level until they are. Existing controls (matched arms, bootstrap intervals, canary limits) remain in use.

1. **A change must beat the baseline's own noise.** A candidate counts only if it beats the baseline by more than the spread between two baseline seeds, when the budget allows two. The idea comes from the fixed-metric loop in Karpathy's autoresearch README (github.com/karpathy/autoresearch, read in raw form), which has no seeds or held-out split of its own.
2. **Two-split gate.** A change is kept only if the tuning split improves and the held-out split does not get worse. The held-out split is never read by whatever proposes changes. Source: Self-Harness, arXiv 2606.09498 (full HTML read), which accepts a candidate "only if it improves at least one split without degrading the other".
3. **Simplicity tie-break.** Inside the noise, a change that removes code or configuration is kept. Source: the same Karpathy README ("all else being equal, simpler is better").
4. **Fail closed.** Two failed sanity checks mean the run is killed and not counted as a result. Source: the fail-closed step in Routed Continual Learning, arXiv 2610.38796 (full HTML read).
5. **One line per change, written before the run:** what it should fix and what could regress, checked afterwards. Source: the predicted-fix note in the harness-evolution paper arXiv 2604.25850. That paper also shows the risk we are guarding against: its headline number was picked on the same tasks that were used for tuning, so we do not copy its headline.

## Limits of this way of working

- One lab, one person, an AI assistant that writes most of the text. Nothing here has been peer reviewed.
- Small models and short runs: 143 million parameters for the main model and 4.5 million for the proxies. Results may not transfer to larger models.
- Few seeds for the expensive runs (usually one), more for the proxies (two). Where a number rests on one run it is stated.
- Several analyses use CPU-sized samples (tens of text windows). Their uncertainty is reported where we computed it and otherwise unknown.
- Our own earlier notes were written quickly by the same assistant that ran the experiments, which is exactly why the review steps above exist; they reduce, but do not remove, the chance of errors.

## Open questions

- How many independent seeds does a production-scale result need before we should trust a 0.01 bits-per-byte difference?
- Can the separate-reviewer step be made independent enough (a different model, a different prompt style) to catch errors the first assistant is blind to?

## Changes and failures since 31 October

Snapshot dated 2 November 2026. These are the method changes and failures from documents 14 and 15.

- **Rules written before the runs, with checksums.** Each new screen has a rule file with its pass and kill conditions written before its first run, and its SHA-256 checksum recorded: the ternary and spiking screen (checksum starting b5037439), the larger 24 and 30-million-token comparison (e3ce6a9a), the three-seed ternary check (bac03090), the efficiency-trick screen (c2583dd2), the replication on cloud T4 GPUs (b9579e1d), the equal-time test of the extra prediction head (d69154ac, not finished), and the two Countdown fine-tunes (3a43f8c2 and 7d9e4d05).
- **Addenda written before results.** (1) The planned 30-million-token runs failed without a result: each was killed by a 1,000-second time limit that was too short (they need roughly 17 to 23 minutes). A dated note was added and a smaller 24-million-token set, with a 25-minute limit, was run instead. (2) For the equal-time test of the extra prediction head, a note was added after launch and before any result: on the home cards the head made a run about 1.7 times as long (11.3 against 6.7 minutes for a comparable run), more than the 1.5 times measured on the T4 GPUs, so the plain-network arm in that design gets slightly less than equal time, and a win would be reported with that handicap. (3) For the second Countdown fine-tune, the first launch ran out of GPU memory at step 0 with 96-token ordinary-text windows; it was rerun with 64-token windows, and the change was written into the rules first.
- **An unrelated overnight freeze.** The home server froze for several hours on 1 to 2 November and had to be power-cycled. It happened after the finished comparison runs and lost no results, and it is unrelated to the 30-million-token failure above.
- **An invalid arm.** The weight-averaging arm gave results identical to the base on every seed. The script never evaluated the averaged weights in that setup, so the arm is marked INVALID and is not evidence either way.
- **Single runs labelled.** A number from one run is labelled as one run in the text. The seed-1 comparison of spiking and looped layers at 24 million tokens is an indication only, and the Countdown replay result covers 100 puzzles rather than 500.

## Changes and failures: learning without forgetting and the senses

Snapshot dated 2 November 2026. These are the method points from documents 18 and 19.

- **Early repair-versus-rollback tests were uninformative, and why.** The first "new skill" was two-digit addition. The tokenizer stores every number from 0 to 100 as one token, so the model could only memorise the answers; and the first scoring sampled answers instead of reading the most likely one. Both were found by reading the code and the tokenizer, and the new skill was replaced by recall of 100 arbitrary facts. The rules were rewritten before the new runs.
- **A cloud-cost estimate was wrong by about three times.** A batch of six rented-GPU jobs cost roughly 9 USD against an estimate of 3. The lesson recorded: compute cost as seconds of GPU time times the price before launching, and put a hard time limit on every job.
- **Verdicts are applied literally, and near-misses are reported as such.** The learn-then-heal method missed its written Countdown condition by one puzzle on two seeds out of three. We reported it as mixed, did not loosen the rule, and ran a larger re-score of the saved models with a new rule written first; that re-score passed.
- **A watcher script bug.** Several times a monitoring script reported that a run had finished when it had just started, because it checked for the wrong process name. No results were affected: each time the run was checked by hand and the watcher fixed.
- **Single seeds labelled.** The 25-percent rehearsal, the weakest-set rehearsal, the extra-text-slot and quartered-learning-rate runs are single seeds. The image model and the bridge are one or two seeds. All are indications only.
