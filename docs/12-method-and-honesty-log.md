# Method and honesty log

## Summary

This project is run by one person and an AI assistant on two consumer graphics cards. That makes it fast and cheap, and it also makes it easy to fool ourselves. This document describes the habits we use against that, and keeps a list of the times they caught us being wrong. The list is part of the research: a claim that survived a skeptic review is worth more than one that was never checked, and a claim that did not survive is worth knowing about.

Everything here is labelled the way the other documents are: **FACT** (measured in our own logged runs), **PUBLISHED** (taken from a paper or documentation that we opened) and **GUESS** (reasoning that was not tested).

## How experiments are run

1. **Short chained sessions.** Training is done in sessions of at most 25 minutes, each resumed from the last checkpoint and followed by a validation gate. A bad session can be stopped and retried without losing a day. (FACT: this is how the production run has been trained since 29 Oct 2026.)
2. **Rules before data.** For the larger decisions (data plan, weight decay, curriculum) the pass or fail rule, the tolerance and the possible outcomes were written down before the result existed. The rule is then applied as written, including when we dislike the answer. Example: the curriculum test (document 09) had a tolerance of 0.004 bits per byte or twice the seed-to-seed spread, whichever was larger; the result was a clear "harm" and we accepted it.
3. **Controls first.** A new environment, tool or metric gets a control run before any result from it is believed. The control caught a biased evaluation on one hardware type (see the log below).
4. **Paired comparisons.** When two models or two settings are compared, the same text windows, the same segments or the same quiz items are used for both, so the difference is not swamped by differences between the texts.
5. **A second reader.** Important results are re-read by a separate AI reviewer instructed to be sceptical, which writes its own note (confirmed, narrowed, corrected, withdrawn). We read every such note before using the result.
6. **Predictions are logged.** When we guess an outcome before a run we write the guess down. Several guesses were wrong in both directions; the misses are listed below.
7. **Labels on every claim.** FACT, PUBLISHED or GUESS. A number without its setting (model size, tokens, seeds, number of windows) is treated as a defect.

## Claims we had to correct

| What we said | What was wrong | Found by | Effect |
|---|---|---|---|
| A first round of small-scale weight-decay tests (FACT: round 1, 30 Oct) showed a large difference between settings | The evaluation on one cloud GPU type was biased by about +0.078 bits per byte because of a compiled evaluation graph; training losses were identical to the home machine | A control run of the same setting on both machines | Round 1 was thrown away; the tests were rerun with eager evaluation and the control passed (1.4082 against 1.4085) |
| "Lower weight decay clearly helps; gains are about thirty times the noise" (an early weight-decay test) | The repeat run shared the random seed with the first, so it was not an independent repeat | A skeptic review | The claim was downgraded to "inconclusive"; the later, properly seeded tests found a flat optimum |
| "There is no small-model evidence for test-time learning" (first biology note) | There is published evidence at 150M to 1B parameters | A skeptic review (row B15 of that review) | The claim was removed and the published numbers were added |
| A planned pass bar of 3 percent for learning-while-reading on whole books | We had not noticed our own earlier pilot (0.67 percent after three updates) nor the published 1.3 to 2 percent | Reading our own notes before building | The bar was lowered to 1 percent; the later results (2.4 to 4.6 percent) cleared both |
| Bits per byte in the reading experiments | The conversion used the byte count of the whole book but the token count of the truncated text, so it was too small for truncated books (it was right for books that were not truncated) | A code review by a separate AI reviewer | No number we report was affected (all gains are given in nats and percent); the conversion was fixed |
| An audit of 41 summary statements from our first status reports | Of the 41: 4 wrong, 1 withdrawn, 10 too strong, 8 out of date, 12 missing a caveat, and 6 confirmed or fixed | A systematic claims table | Each statement was corrected or caveated |
| "The matrices grew by 2.24 times in two sessions, which is the equilibrium of the weight-decay balance model (the square root of five)" | The weights kept growing: 2.31 and 2.33 times after three and four sessions, about 4 percent above the prediction | A skeptic review of the weight-decay tests | The claim was withdrawn; the plateau is now stated as about 2.33 (GUESS), and the first-session jump and the time scale are still reported as predicted |
| An AI helper reported that our held-out evaluation text came from a different web corpus (DCLM-baseline) than the one our brief named (FineWeb-Edu) | The helper had followed the wrong data-preparation script. The files that built the validation set show it is a held-out part of the FineWeb-Edu sample; the other script builds a separate training folder | The main session checked the build scripts and the file dates, and compared a sample of documents | The comparison in document 04 carries the corrected statement, which matters because it makes overlap with SmolLM's training data likely |
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

## Limits of this way of working

- One lab, one person, an AI assistant that writes most of the text. Nothing here has been peer reviewed.
- Small models and short runs: 143 million parameters for the main model and 4.5 million for the proxies. Results may not transfer to larger models.
- Few seeds for the expensive runs (usually one), more for the proxies (two). Where a number rests on one run it is stated.
- Several analyses use CPU-sized samples (tens of text windows). Their uncertainty is reported where we computed it and otherwise unknown.
- Our own earlier notes were written quickly by the same assistant that ran the experiments, which is exactly why the review steps above exist; they reduce, but do not remove, the chance of errors.

## Open questions

- How many independent seeds does a production-scale result need before we should trust a 0.01 bits-per-byte difference?
- Can the separate-reviewer step be made independent enough (a different model, a different prompt style) to catch errors the first assistant is blind to?
