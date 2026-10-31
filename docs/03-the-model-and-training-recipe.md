# 03 The model and the training recipe

How to read the labels. FACT is a value measured in our own runs, with the setting in brackets (model size, tokens, seeds, windows). PUBLISHED is taken from a paper, with authors, year and arXiv identifier. GUESS is reasoning that was not tested. Plans, definitions and derived arithmetic are marked as such.

## Summary

All numbers in this summary are FACT; their settings and sources follow in the sections below. The model has about 143 million parameters: 12 transformer blocks of width 640, 5 attention heads of width 128, a context of 1,024 tokens and a 32,768-token byte-pair vocabulary. It is trained in bf16 on an 8 GB consumer GPU, with the Muon optimizer on the hidden weight matrices and AdamW on everything else. Instead of one long run, training is a chain of sessions of about 24 minutes and about 50 million tokens; each session resumes from the last checkpoint and must pass a gate before the next one starts. At the time of writing (about 8 PM on 30 October 2026) the production chain had finished session 39, about 2.0 billion tokens, with a best uncooled validation value of 1.1161 bits per byte (session 38; session 39 read 1.1199), and its planned cooldown had not run. A second, older GPU ran a comparison chain so that the effect of the data mix could be separated from the effect of weight decay.

Basis: our own logged runs and notes from 26 to 30 October 2026. Those notes are not published; this document is meant to stand on its own.

## The model

- FACT [143M model: 142,868,530 parameters, counted when the model is loaded]: 12 blocks, width 640, 5 attention heads of width 128 (5 times 128 equals 640), context 1,024 tokens.
- FACT: about 59.0 million weights sit in the 48 hidden matrices (the attention projections and the two MLP matrices of each block). About 83.9 million (embeddings, value tables, output head and scalars) sit outside them.
- FACT: vocabulary of 32,768 byte-pair tokens, used in all runs of the recipe. In single 17M-model runs, vocabularies of 4k, 8k, 16k and 49k were each worse than 32k (single runs; intervals not measured).
- FACT: value embeddings are lookup tables that add token-specific vectors into the first two and the last two blocks.
- FACT: value residual: the first block's value vectors are mixed into later blocks, with initial mixing weights of one half each. It is part of the recipe accepted at 45M tokens (table below).
- FACT: key shift: one head in each block uses partial rotary position encoding and lets each position read the key of the previous token. In the 10M test the shift carried the whole gain (table below).
- FACT: lambdas are per-block scalars: one re-injects the first-layer embedding, one carries the residual. They have no weight decay.
- FACT: output logits pass a soft cap at 15 before the loss (head-check set: 16 windows of 1,024 tokens).
- FACT: attention scale is the default 1/sqrt(128), about 0.088. A scale of 0.12 looked better at 10M tokens and was not better at 45M (table below).

## Optimizer, precision and batch

- FACT: Muon on the 48 hidden matrices, step size 0.02, decoupled weight decay (schedule in document 05). The Muon update is an orthogonalised momentum. AdamW on the other parameters, step size 0.008, second-moment decay 0.99; the embedding step size is multiplied by 20.
- FACT: batch of 32,768 tokens per step (four micro-batches of 8,192 tokens), context 1,024, no gradient clipping.
- FACT: bf16 mixed precision on the production card (an 8 GB RTX 4060); fp16 on the comparison card (an RTX 2060 SUPER).
- FACT: session 0 has learning-rate, momentum and sequence warm-ups; later sessions run at a constant learning-rate multiplier. A cooldown of 18 sessions (a staircase down to zero) was planned from about 8.1 billion tokens; it had not run at the last note read.
- FACT (10M tokens, one seed per card): a later trainer revision reproduced the earlier one to within 0.0001 bits per byte (validation 1.5572 against 1.5572 on one card, 1.5517 against 1.5516 on the other), so the revision did not change the numbers.

## Recipe evidence

All rows FACT. The paired difference is the change minus its control, in bits per byte.

| Test | Setting | Paired difference | Verdict |
|---|---|---|---|
| Value residual with sequence warm-up | 45M tokens, 4 seeds | -0.0078 (SE 0.0009) | accepted |
| Key shift | 45M tokens, 4 seeds | -0.0061 (SE 0.0005) | accepted |
| Key shift | 24-minute clock, 6 seeds, pooled | -0.0064 (SE 0.0004) | accepted; control on an earlier revision |
| Key shift without the previous-token shift | 10M tokens, 6 seeds | +0.0012 (SE 0.0014) | no effect |
| Attention scale 0.12 | 45M tokens, 4 seeds | +0.0096 (SE 0.0080) | dropped; the 10M gain did not persist |
| Output gate on attention heads | 45M tokens, 4 seeds | -0.0039 (SE 0.0017) | not significant; open |
| Logit cap 30 | 10M tokens, 6 seeds | +0.0035 (SE 0.0009) | dropped |
| Mixing in AI-written text | 45M tokens, 4 seeds | +0.0061 (SE 0.0031) | not adopted |

- The acceptance rule in our experiment record: a change counts only if it persists at 45M tokens with at least four seeds, both cards agree, and the t statistic passes its critical value. A 10M result alone never counts (FACT, the record's rules).

## Training data

- FACT [pool of 2.81 billion tokens, six groups by share of the pool]: an educational-web group (older and newer FineWeb-Edu-family text, with books and Wikipedia in a 60/20/10/10 mix), 14.2 percent; a DCLM-style web group filtered by a quality score, in two parts of 23.1 and 25.6 percent; a Wikipedia snapshot, 15.2 percent; educational pages from Wikibooks, Wikiversity and Simple Wikipedia, 11.2 percent; public-domain fiction from 1,844 Project Gutenberg books, 10.6 percent.
- FACT: all sources are human-written. An AI-written text mix was tested at 45M tokens and not adopted (table above).
- FACT [three mixes in the data-plan notes]: mix A draws 52 million tokens per session from the pool. Mix B draws about 16 million from the pool and about 36 million from a newer educational-web dump of 2025 snapshots. Mix C70 draws about 36 million from the pool and 16 million from the newer dump (70/30).
- FACT: the design note of 29 October set mix A as the default for the production chain. The production chain ran mix B for sessions 0 to 11 (decision note, 29 October).
- FACT [sessions 0 to 11, mix B]: session 11 was 73.1 percent educational-web family by tokens.
- FACT [sessions 12 onward, mix C70]: each session draws about 36 million tokens from the pool and about 16 million from the newer dump. Session 12 was 30.8 percent educational-web family. The change followed a comparison at equal tokens (document 06).
- FACT: inside a session the trainer draws windows at random with replacement; about 63 percent of a session's slice is touched at least once.
- FACT: the validation text is a held-out part of the FineWeb-Edu sample (human-written educational web pages), the same family as part of the training mix. Under mix B that family supplied most of the tokens (73.1 percent at session 11), so the validation value moves with the mix (see the validation section).

## The training form: a chain of sessions

- FACT [measured 24-minute run of the recipe on the production card, not a chain session]: 1,549 steps, 50,757,632 tokens, 24.42 minutes, 34,642 tokens per second. A chain session has a training budget of 24 minutes inside a hard limit of 30 minutes.
- FACT: each session loads the model and both optimizer states from the last good checkpoint and writes a new checkpoint only after a complete file is on disk. Two alternating slots keep the last good one. In a stop test, a session killed at 50 seconds left both slots and the chain state unchanged; the retry used the same data slice and succeeded.
- FACT [the gate]: a session counts only if the process exited cleanly, the validation value is finite, no step was skipped for a non-finite loss, the checkpoint has the expected size and record, and throughput is above half the reference. Warnings that do not stop the chain: throughput outside 5 percent of the reference, validation more than 0.02 above the best so far, free disk below 60 GB, a session cut by its time limit. Two failed sessions in a row stop the chain.
- Why a chain (the rule is FACT; the benefits are GUESS, untested): a project rule caps each job at 30 minutes; a crash loses at most one session; each session can be checked before the next starts; and settings can change at session boundaries, which is how the weight-decay steps and the data change were made. The cost is about 13 seconds of checkpoint saving per session (smoke test, FACT) plus start-up.
- PLAN (design note, 29 October): about 9 billion tokens in about 177 sessions, about 3 days at the planned rate. FACT: the chain had completed 40 sessions (0 to 39) by about 8 PM on 30 October, 1,995.6 million tokens (read from the session records; see the milestone table).

## The comparison chain on the older card

- FACT: an RTX 2060 SUPER ran a second chain in fp16 on mix A throughout, about 44 million tokens per session at about 30,000 tokens per second. Its decay was 0.1 through session 12 and 0.02 from session 13; its data did not change at that point (document 05).
- FACT: its validation values were between 1.18 and 1.20 bits per byte at sessions 22 to 29 (1.1922 at session 22, 1.1838 at 25, 1.1953 at 27, 1.1998 at 29).
- The two chains are not compared at equal tokens in this document; the equal-token comparison is in document 06.

## Monitoring added during the run

- FACT [head-size check, every session, 16 windows of 1,024 tokens]: root-mean-square of the output-head weights; the share of pre-cap logits whose z-value exceeds 10 (halt line 0.20; revert trigger 0.16 on two checks in a row); cap dependence, the loss without the cap minus the loss with it, in nats (warning 2.5, trigger 2.3).
- FACT [regime check]: per-session growth of the 48 hidden matrices (trigger 1.25), the largest skip scalar (trigger 25), growth of the residual stream, and fp16 headroom on the comparison card.
- FACT [equal-token comparison]: the two chains are scored on the same held-out windows at equal token counts; a pre-registered rule turns each difference into a label with a z-score. The rule had no agreed action for its AHEAD label, a gap the notes record.
- FACT [noise]: the adjacent-session change of the head-share indicator has a standard deviation of 0.0196 at chain level. Seed-to-seed training noise at 45M tokens is 0.0046 bits per byte per run.
- FACT [snapshots]: slim copies keep the weights, and optionally the head's optimizer state, at about half the size (739 million bytes with the head's optimizer state, against 1,479 million for a full copy; 571 million bytes for weights only; test copy). Head-check output on a slim copy matched the full copy field for field.

## Milestones

FACT for every row (read from the saved record of each session): production chain, one run per session, validation of 1,952 windows (2,000,001 tokens), uncooled (constant learning rate after the warm-up in session 0), one seed. The standard error of one validation value is about 0.002.

| Session | Matrix decay | Head decay | Tokens (million) | Validation (bits per byte) |
|---|---|---|---|---|
| 0 | 0.1 | none | 51.0 | 1.3663 |
| 1 | 0.1 | none | 101.4 | 1.3133 |
| 2 | 0.1 | none | 151.7 | 1.2916 |
| 3 | 0.1 | none | 201.8 | 1.2775 |
| 4 | 0.1 | none | 252.0 | 1.2742 |
| 5 | 0.1 | none | 302.1 | 1.2658 |
| 6 | 0.1 | none | 352.2 | 1.2675 |
| 7 | 0.1 | none | 402.3 | 1.2588 |
| 8 | 0.1 | none | 452.4 | 1.2520 |
| 9 | 0.1 | none | 502.4 | 1.2485 |
| 10 | 0.1 | none | 552.5 | 1.2478 |
| 11 | 0.1 | none | 602.6 | 1.2495 |
| 12 | 0.02 | none | 652.7 | 1.2120 |
| 13 | 0.02 | none | 702.8 | 1.2038 |
| 14 | 0.02 | none | 752.8 | 1.2026 |
| 15 | 0.02 | none | 802.9 | 1.1939 |
| 16 | 0.02 | none | 853.0 | 1.1862 |
| 17 | 0.02 | none | 903.0 | 1.1868 |
| 18 | 0.02 | none | 953.1 | 1.1829 |
| 19 | 0.02 | none | 1,002.5 | 1.1808 |
| 20 | 0.02 | none | 1,052.1 | 1.1925 |
| 21 | 0.02 | none | 1,101.8 | 1.1721 |
| 22 | 0.02 | none | 1,151.4 | 1.1685 |
| 23 | 0.02 | none | 1,201.0 | 1.1764 |
| 24 | 0.006 | none | 1,250.6 | 1.1577 |
| 25 | 0.006 | none | 1,300.2 | 1.1509 |
| 26 | 0.006 | none | 1,349.7 | 1.1410 |
| 27 | 0.003 | none | 1,399.4 | 1.1430 |
| 28 | 0.003 | none | 1,449.0 | 1.1405 |
| 29 | 0.003 | none | 1,498.6 | 1.1352 |
| 30 | 0.003 | 0.01 | 1,548.3 | 1.1307 |
| 31 | 0.003 | 0.01 | 1,598.1 | 1.1318 |
| 32 | 0.003 | 0.01 | 1,647.8 | 1.1288 |
| 33 | 0.003 | 0.003 | 1,697.4 | 1.1260 |
| 34 | 0.003 | 0.003 | 1,747.1 | 1.1299 |
| 35 | 0.003 | 0.003 | 1,796.7 | 1.1275 |
| 36 | 0.003 | 0.003 | 1,846.4 | 1.1238 |
| 37 | 0.003 | 0.003 | 1,896.2 | 1.1186 |
| 38 | 0.003 | 0.003 | 1,945.9 | 1.1161 |
| 39 | 0.003 | 0.003 | 1,995.6 | 1.1199 |

- Sessions 0 to 11 used mix B; from session 12, mix C70.
- Sessions 8 to 11 were flat, with a standard deviation of 0.002 bits per byte between sessions (FACT).
- Best value so far: session 38, 1.1161. Session 39 read 1.1199, which is 0.0038 higher and about two standard errors of one value, so the curve is noisy at this level; the trend since session 27 is still downward.
- The values are uncooled and overstate a cooled model. One cooled session from the session-11 checkpoint gave 1.1474 against 1.2495 uncooled (FACT, one seed; document 05). The cooled gap at later sessions has not been measured.

## Throughput

- FACT (production card): 34,642 tokens per second in the measured 24-minute run; between about 33,700 and 34,200 in logged chain sessions (33,855 at session 27, 33,873 at session 23, 33,895 at session 32).
- FACT (comparison card): about 30,000 to 30,300 tokens per second (30,134 at session 22; 30,291 at session 27).
- FACT: once, for about 10.5 hours, neither card had a session queued, which cost about 20 session slots. This was a scheduling gap, not a training failure.

## What failed, and what was fixed

1. Resumed runs took the decay saved in the checkpoint instead of the value set for the run. The later trainer revision re-applies the chosen settings on resume (FACT: a short test showed the checkpoint carrying the set value, and the next session returned to the chosen value).
2. A stopped session left the checkpoints unchanged and the retry reused its slice (FACT, stop test).
3. A compiled evaluation on a hosted GPU environment scored the same model 0.078 bits per byte worse than the uncompiled evaluation (FACT, 4.46M-parameter proxy, 1,500 steps). A control run caught it; the evaluation was then run uncompiled (document 05).
4. Full checkpoints of about 1.48 GB would have filled the disk above its safety floor in about 4.5 hours of snapshot saving. Slim copies solved this. The first conversion failed for the revised head layout, so sessions 30 and 31 had no slim copy until it was fixed. The comparison tool also ignored files under 1.4 GB, so its threshold had to be lowered before the next comparison (FACT).
5. Activations grew past the fp16 range. The largest MLP output reached 88,000 at session 28, against an fp16 limit of 65,504, so the production model runs in bf16 or fp32 from then on (FACT). The comparison card's fp16 margin was 5.2 at session 15, against a warning level of 6 (FACT). An earlier model overflowed fp16 on 81.1 percent of 4,096 random test tokens (FACT, one check).
6. The output head grew with no decay on it: its head-check RMS rose from 0.6343 at session 8 to 0.9472 at session 28 (FACT). Head decay from session 30 brought it into a chosen band (document 05).
7. The benchmark code could not load the revised trainer's checkpoints (FACT: a loader test failed at once). A separate loader was written for the benchmark runs. The same mistake happened again on 30 October, when the first benchmark of session 36 stopped at loading because it used the older loader; it was rerun with the newer one.
8. A metadata error: the bits-per-byte metadata of the second held-out set was copied from another file, so its bits-per-byte value was wrong. That set is now reported in nats per token (FACT).
9. A code review found that a stored validation value of an earlier chain may read about 0.4 percent high (UNVERIFIED: the arithmetic was checked, the evaluation was not). Earlier-chain numbers are not used in this document.

## The validation text and what bits per byte means

- FACT: 2,000,001 tokens, 1,952 windows of 1,024 tokens at fixed positions, the same at every session, so differences between sessions are paired. The text is a held-out part of the FineWeb-Edu sample, the same family as part of the training mix, so it is not fully independent of the mix. It is not a standard benchmark.
- FACT [uncertainty]: the standard error of one run's value is about 0.002 bits per byte (0.00208 in the measured 24-minute run, one seed).
- DEFINITION: bits per byte equals nats per token divided by the natural logarithm of 2 and by the average number of bytes per token of the text. For this text the average is 4.5639 bytes per token.
- FACT: a second held-out set (about 6 million tokens from the pool) is reported in nats per token only.
- Plainly: a bits-per-byte number depends on the text it is measured on. Our numbers can be compared only with other numbers measured on this same text with the same byte count. They are not comparable with other models' published figures unless those were measured on the same text.

## What this does not show

- The final quality of the model. The chain has not reached its planned length or its cooldown, and every number above is uncooled.
- Benchmark results at each milestone. Benchmarks were run at a few points, mainly for the comparison runs.
- That the schedule and sizes are optimal. Session length, cooldown length and data plan were chosen under time limits, and the cooldown length has not been tested at this scale.
- Performance on text outside the validation family. The validation text is one educational-web family.
- Independent verification. The runs were designed, executed and checked with an AI assistant under the owner's decisions, and the skeptic passes were also AI-run. No outside reviewer is recorded in the notes.

## Open questions

1. What is the cooled value at about 9 billion tokens, and how large is the cooled gap at later sessions? Only one measurement exists, at session 11.
2. Does the uncooled curve keep falling at its current pace after the decay steps? The next sessions will show it.
3. Is an 18-session cooldown the right length? It is untested at this scale; cooled branch tests are planned.
4. Does mix C70 remain the right mix at the next milestone? (Document 06.)
5. Will the head band hold once the cooldown is applied? (Document 05.)
