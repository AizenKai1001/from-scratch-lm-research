# Data plans and the equal-token comparison

## Summary

From the start of training on 29 October 2026, two data plans ran side by side on the 143-million-parameter model. Plan B, mostly educational web text, ran on the main run (chain 1), and plan A, a repeated pool of six corpora read in file order, ran on a comparison run (chain 2). At equal tokens plan A was better on eight of nine evaluation sets and worse on the old validation set, whose text belongs to plan B's family. Before any data for a new plan had been read, the project wrote down retention ratios with a threshold of 0.4, and on the evening of 29 October it moved the main run to a third plan, C70: about 70 percent of each session from the broad pool (in a fixed random slice order) plus about 30 percent fresh text. A check on 30 October pooled four token-matched pairs; both retention ratios came out above 0.4, so the main run kept C70, and the cost was a later headline comparison, because the comparison run was paused. Each plan has one run: the first comparison was at about 0.6 billion tokens and the pooled check at 1.05 to 1.20 billion, of a 9-billion-token plan, and these results and they say nothing yet about the final cooled models.

## Scope, labels and setting

- Basis: our own logged runs, design and decision notes and a re-check runbook from 29 and 30 October 2026. Those notes are not published; this document is meant to stand on its own.
- Labels: FACT means measured in our own runs, with the setting given. PUBLISHED means taken from a source we opened, named where used. GUESS means reasoning that was not tested. Arithmetic on measured values is marked FACT, with its inputs.
- Setting (FACT): the model has 142.87 million parameters. It was planned for 9 billion tokens in sessions of 24 minutes, each reading about 44 to 50 million tokens. The main run used the newer 8 GB card in bf16 arithmetic; the comparison run used the older card in fp16.
- Some of the CPU evaluations from about 10 PM on 29 October used weight-only copies of the saved models instead of the full files. On 16 windows, their per-window losses were identical to those from the full files (FACT, slim-copy check, 29 October). The first comparison used full files.

## The three data plans

All three plans draw on the same two stores (FACT, data build). The broad pool is a fixed file of about 2.81 billion tokens, made by interleaving six human-written corpora in blocks of one million tokens. Its mix is about 49 percent web text, 29 percent Wikipedia family, 13.5 percent fiction and books, 8 percent FineWeb-Edu family, under 1 percent each of math and code, and no science text. The fresh store is about 16.2 billion tokens of newer FineWeb-Edu-family web text, read in file order.

| | Plan A | Plan B | Plan C70 |
|---|---|---|---|
| Used by | comparison run, from the start | main run, sessions 0 to 11 | main run, from session 12 |
| Tokens per session (planned) | 52 million from the broad pool | 16 million from the broad pool, 36 million fresh | 36 million from the broad pool, 16 million fresh |
| Order of broad-pool slices | file order in the first pass, shuffled afterwards | file order | fixed random order from the first pass |
| FineWeb-Edu family share of tokens (measured) | 14.2 percent, zero after session 29 | about 73 percent (73.1 percent at session 11) | not measured at a milestone |
| Passes over the broad pool by 9 billion tokens | about 3.2 | about 1.0 | about 2.2 |
| Fresh text read by 9 billion tokens | not used | 0.39 of the file, never repeated | 0.18 of the file |

Labels: the composition and the measured shares are FACT. The pass counts and fractions are FACT arithmetic on the planned session sizes, so the 9-billion-token figures are plans, not measurements.

Plan A's first pass is in file order, which makes it a curriculum rather than a shuffled mix (FACT, timeline simulated from the mixing script). At the first milestone (session 11) it had not yet read the math, code, Simple Wikipedia or wikiversity slices, nor the last three quarters of the fiction books (FACT, same simulation).

## Why these plans

- The production design defaulted to plan A as the evidence-neutral choice, because the evidence then did not support fresh-by-default (FACT, production design note). A one-session, one-seed test had found no benefit from fresh text on unseen text matched to the repeated pool, and no gain on the benchmarks (FACT). That test was not a clean repetition test, because its fresh text came from other sources, so it could not settle the question either way (FACT, later review of that test).
- The main run started on plan B at about 1:26 PM on 29 October. The comparison run started on plan A at about 2:09 PM the same day (FACT).
- The reasons recorded for plan B were that the old validation text is educational web text, so a mixture about 70 percent of that family should score better on it, and that plan B reads each fresh token once, while plan A reads each broad-pool token about three times by 9 billion tokens (GUESS for the first reason, which is reasoning; FACT arithmetic for the second).
- Published context: Muennighoff and colleagues (2023, "Scaling Data-Constrained Language Models", arXiv 2305.16264) report in their abstract that training on repeated data for up to four epochs changes loss negligibly, and their experiments reach 900 billion training tokens (PUBLISHED; abstract opened on 30 October). Plan A's 3.2 passes sit inside that band (FACT arithmetic). Our own estimate of the repetition cost at 9 billion tokens is 0.0005 to 0.005 bits per byte, below what one run can detect (GUESS, not measured). For scale, the run-to-run spread at about 45 million tokens was 0.0046 bits per byte (FACT, registry of earlier runs).
- C70 was chosen at about 8 PM on 29 October and written into the main run's configuration at 8:12 PM (FACT). It was chosen for robustness, not for a measured win (FACT, the decision note says so). It cuts the broad-pool passes by 9 billion tokens from about 3.1 to about 2.2 (FACT arithmetic), keeps a fresh stream (FACT arithmetic), and its first pass has no source order (FACT, design). On an equal-weight scoreboard it projects to within 0.006 to 0.026 nats of plan A and 0.065 to 0.085 nats better than plan B, and it stays near the best option for any weight on the old validation between about 0.2 and 0.72 (GUESS: interpolation between two single-run endpoints).

## The first comparison (29 October, evening)

Setting (FACT): main run at 602.6 million tokens; comparison run at 577.0 million tokens; both at decay 0.1; one run each, with different starting seeds; 128 windows of 512 tokens per set, the same windows for both models; standard errors over windows. The values are nats per token. Negative means plan A is better.

| Set | What it is | A minus B | Standard error |
|---|---|---|---|
| Old validation | educational web text, plan B's family | +0.068 | 0.010 |
| Broad pool, web sample 1 | held-out slice | -0.058 | 0.012 |
| Broad pool, web sample 2 | held-out slice | -0.074 | 0.012 |
| Broad pool, Wikipedia 2026 | held-out slice | -0.132 | 0.012 |
| Broad pool, Simple Wikipedia tail | held-out slice | -0.136 | 0.014 |
| Broad pool, fiction | last books | -0.165 | 0.006 |
| Neutral, science papers | in neither pool | -0.098 | 0.010 |
| Neutral, math web pages | same corpus as the pool, disjoint text | -0.058 | 0.013 |
| Neutral, code | same corpus as the pool, disjoint text | -0.163 | 0.008 |

Labels: all rows are FACT. The mean over the eight sets that are not the old validation was -0.111 (standard error 0.004). The mean over all nine was -0.091 (standard error 0.004).

- Plan A is better on all eight sets that are not the old validation, and worse on the one set in plan B's family (FACT).
- The gap is not limited to pool membership. Science is in neither pool, and code is in the pool only in small amounts, yet their gaps (-0.098 and -0.163) are comparable to the Wikipedia gaps (-0.132 and -0.136) (FACT). GUESS: plan B's heavy share of one educational register makes it a specialist for that register, while plan A's wider exposure transfers further.
- The main run had 4.4 percent more tokens, which favours it slightly (FACT).

## Rules written before the data existed

- The rules were written at 8:07 PM on 29 October, before any data for the new plan had been read (FACT).
- Retention on the old validation, r_F, is the share of plan B's old-validation advantage over plan A that the new plan keeps. It equals minus (new plan minus A) divided by 0.068, the old-validation gap between A and B at the first comparison (FACT for the denominator).
- Retention on breadth, r_B, is the share of plan A's breadth advantage over plan B that the new plan keeps. It equals one minus (new plan minus A) divided by 0.111, the eight-set gap at the first comparison (FACT for the denominator).
- The threshold for each ratio is 0.4. If both are at least 0.4, keep C70. If r_B is below 0.4 and r_F is at least 0.4, raise the broad-pool share to 85 percent. If r_F is below 0.4, switch to plain A, because C70 would buy no headline gain. If both are below 0.4, switch to plain A. Benchmarks decide only if the composite differs by about two standard errors.
- In nats, r_F of at least 0.4 means the new plan minus A is about -0.027 or lower on the old validation, and r_B of at least 0.4 means about +0.067 or lower on the eight sets (DERIVED from the two denominators, rounded).
- Predictions written at the same time (GUESS): new plan minus A on the old validation -0.040 nats (80 percent interval -0.070 to -0.010); on the eight sets +0.04 (-0.01 to +0.09); r_F between 0.56 and 0.69; r_B between 0.70 and 0.89.
- Both denominators come from runs at decay 0.1. Both runs later moved to decay 0.02, so the ratios and the nats thresholds describe different regimes (FACT). The project kept the ratios as written and added the nats thresholds (FACT, runbook).

## Lumpy snapshots and the remedy

- Early checks between 10:05 and 11:25 PM on 29 October showed that a main-run snapshot is a noisy witness (FACT). Each broad-pool slice is a contiguous stretch in which each source sits at one position, so a snapshot mostly reflects its last one or two slices. Between main sessions 12 and 14 the main run's science loss rose by 0.186 nats and its fiction loss fell by 0.228 nats (FACT). Over the comparison run's sessions 13 to 16, its losses fell on all nine sets (FACT). The recency mechanism is GUESS; the swings are measured.
- The first pair after the switch (main session 12 against comparison session 13) gave a breadth gap of -0.053. The pairs at main sessions 14 and 15 gave +0.078 and +0.048 (FACT). The runbook attributed the first value to the slices it read (GUESS).
- Three settled early pairs (main 13 with comparison 15, 14 with 16, and 15 with 17), pooled, gave a headline retention of 0.09 (FACT). Those slices carried no block of educational web text, the family the old validation resembles, so the early headline reading was biased against the new plan (GUESS: the runbook's explanation).
- The remedy was to pool four token-matched pairs, use the spread across pairs as the error, and never act on one pair's per-set number (FACT, runbook). The pairs were: main session 20 with comparison session 23; 21 with 24; 22 with 25; 23 with 26. Their token counts differ by about one percent (DERIVED: ratios from 0.99 to 1.005).
- The single pre-registered pair (main 23 with comparison 26) would have given a headline retention of 0.44 and a breadth retention of 0.87 (FACT). That is close enough to the threshold that one pair could not settle the question.
- The runbook wrote a proposed reading of the rules at 10:55 PM, while early pairs were being measured. It differed from the literal rule in one cell only, so it was not a clean pre-registration. Both readings gave the same answer on the pooled data (FACT).

## Result of the pooled check (30 October)

Setting (FACT): four pairs, 64 windows per set for each model, CPU, the same windows for both models. The pairs cover about 1.05 to 1.20 billion tokens (DERIVED from session token counts). Values are main run minus comparison run, in nats per token.

| Quantity | Value | Error or threshold |
|---|---|---|
| Old validation, pooled | -0.0326 (main run better) | window standard error 0.0055; spread across pairs 0.0084 |
| Eight breadth sets, pooled | +0.0426 (comparison run better) | window standard error 0.0018; spread across pairs 0.0166 |
| All nine sets, pooled | +0.0343 | none stated |
| r_F | 0.479 | threshold 0.4 |
| r_B | 0.616 | threshold 0.4 |
| Eight-set gap by pair, in the order the pairs are listed above | +0.091, +0.032, +0.033, +0.015 | none |
| Old-validation gap by pair | -0.011 to -0.051 | none |
| Pre-registered pair alone (23 with 26) | old validation -0.030; eight sets +0.015; r_F 0.44; r_B 0.87 | none |

Labels: the values are FACT. The margins below are DERIVED from them.

- Both ratios are above 0.4, so the literal rule and the proposed reading both say keep C70 (FACT). The decision record of 30 October keeps the main run on C70 with no change (FACT).
- Both predicted intervals contained the result: -0.0326 is inside -0.070 to -0.010, and +0.0426 is inside -0.01 to +0.09 (DERIVED).
- The margins are small against the spread across pairs. The old-validation result is about 0.0054 nats past the r_F line, about 0.6 of the spread error. The breadth result is about 0.024 nats on the passing side of the r_B line, about 1.4 spread errors.
- The breadth gap fell from +0.091 to +0.015 across the four pairs, with one flat step between the second and third (FACT). Whether it keeps falling is not known, because the comparison run was paused.

## What was given up

- The comparison run was paused on 30 October. Its running session (session 30) finished at 5:27 PM and the run then halted (FACT). The project owner had approved the pause earlier that afternoon (FACT). The second card went to the reader experiments (document 08) (FACT).
- Given up: the later headline comparison, planned as the comparison run at session 36 (about 1.64 billion tokens) against the main run at session 32 (FACT: planned, not run).
- Given up: a matched control for the main run's later decay changes. The main run lowered its decay from session 24, while the comparison run stayed at 0.02, so the two runs no longer match after session 23 (FACT; see document 05).
- The pause can be lifted at a session boundary, and the comparison run would resume from its last committed session (FACT).

## Corrections to our earlier notes

- The early reading that C70 buys no headline gain was probably too pessimistic, as the runbook itself judged (GUESS). The pooled check gives a headline retention of 0.479, against 0.09 for the three early pairs (FACT).
- The first pair after the switch, with a breadth gap of -0.053, was not representative of the later pairs (FACT). The runbook's attribution to the slices read is GUESS.
- The comparison tool printed an automatic verdict line written for an earlier question. It would have flipped with the last slice, so it was not used (FACT).
- The proposed reading of the rules was written while early pairs were being measured, so it is not a clean pre-registration (FACT).

## What this does not show

- It does not show whether C70 beats either plan at 9 billion tokens, or after the cooldown. No cooled models were compared, and the runbook notes that a ranking can change after cooldown (GUESS).
- One run per plan and one seed each, with different starting seeds for the two runs. The two runs also used different card arithmetic (bf16 and fp16). The rule carries a lane allowance of 0.0063 nats that was not measured (FACT for the value; its adequacy is GUESS).
- Some evaluation sets are narrow: the Simple Wikipedia tail and the last books are small (FACT). Windows are drawn with replacement, and the standard errors leave out run-to-run noise, which earlier runs put near 0.015 nats (FACT for the estimate, from earlier seed spread).
- Repetition and mixture change together in these runs, so their effects are not separated (FACT).
- All pooled runs used decay 0.02 on both sides, while the retention thresholds were set at decay 0.1 (FACT).

## Open questions

1. Does the breadth gap keep falling as the comparison run reads later sources? (GUESS: probably slowly. The pooled check used no comparison-run data after session 26.)
2. How does C70 rank against plan A once both are cooled at 9 billion tokens? (Not measured.)
3. Would a plan-A run with shuffled slices, at the same decay, separate the order effect from the repetition effect? (GUESS: this is the test the design points to; it has not been run.)
4. If the comparison run is resumed, do its later sessions change the headline result? (Open. The one check so far showed no lift when the run passed the old educational block.)
