# 05 Weight decay and training dynamics

How to read the labels. FACT is a value measured in our own runs, with the setting in brackets (model size, tokens, seeds, windows). PUBLISHED is taken from a paper, with authors, year and arXiv identifier. GUESS is reasoning that was not tested. Plans and derived arithmetic are marked as such.

## Summary

All numbers in this summary are FACT, with settings and sources in the sections below. Weight decay, a pull of every weight toward zero applied at each step, is the setting that changed most in the production chain: 0.1 for sessions 0 to 11, 0.02 from session 12, 0.006 for sessions 24 to 26 and 0.003 from session 27, with a separate decay on the output head from session 30. Cutting it fivefold roughly doubled the scale of the 48 hidden weight matrices within one session, and the first two sessions matched the decay model's predictions, which had been written down before the weights were read; the weights then kept growing past the level the model called final (2.33 times their starting size after four sessions, against a predicted 2.24). The validation value fell by 0.0375 bits per byte in that session (the comparison chain fell by 0.0425 with the decay change alone). A separate skeptic pass judged the fall real but mostly uncooled: a larger weight scale lowers the effective learning rate, and a one-session cooled test left only 0.0052 bits per byte of gain after cooling. A proxy sweep found the best decay flat between two low settings, while the production chain was still at 0.003; the next step waits for the head check to stay stable.

Basis: our own logged runs and review notes from 29 and 30 October 2026. Those notes are not published; this document is meant to stand on its own.

## The schedule

| Sessions | Hidden-matrix decay | Output-head decay | Comment |
|---|---|---|---|
| 0 to 11 | 0.1 | none | Matrix scale flat: session-to-session ratio 0.997 to 1.001 |
| 12 to 23 | 0.02 | none | Switched at session 12, together with the change to mix C70 |
| 24 to 26 | 0.006 | none | First staged step; checks after every session |
| 27 to 29 | 0.003 | none | Steps halved from here |
| 30 to 32 | 0.003 | 0.01 | Head decay from session 30 (revised trainer) |
| 33 onward | 0.003 | 0.003 | Lowered after the head fell quickly; still 0.003 at session 36, the last note read |

- FACT (production chain): the schedule above. The comparison chain (second card) stayed at 0.1 through session 12 and moved to 0.02 at session 13 with no data change, which makes it the decay-only test.
- FACT: decay acts only on the 48 hidden matrices; the other parameters have none. The per-step pull is the step size times the decay: 0.002 per step at 0.1 and 0.0004 per step at 0.02 (arithmetic from the settings).
- PLAN, on hold (decision notes): 0.0015, then 0.0007 (the proxy optimum is about 0.0005). The decision notes hold the step until the head check is stable. An earlier plan (1:05 AM) used two-session steps down to 0.0006; later notes replaced it with halving steps, and this document follows the later plan.

## The measured jump

- FACT [production chain, session 12 against session 11; 48 matrices; one seed]: median ratio of matrix RMS 2.013 (interquartile 1.976 to 2.034). By type: 2.02 for the attention query-key-value matrices, 2.06 for the attention output, 1.97 for the first MLP matrix, 2.01 for the second.
- FACT [comparison chain, session 13 against session 12; decay change only]: median 1.955 (interquartile 1.918 to 1.965).
- FACT: before the switch, the same consecutive-session ratio was between 0.997 and 1.001 in both chains.
- FACT [production chain, two switched sessions, session 13 against session 11]: median ratio 2.238. The decay model's equilibrium value is the square root of five, 2.24.
- FACT [production chain, later sessions, against the snapshot before the change, mean ratio of the 48 matrices]: 2.242 after two sessions, 2.307 after three and 2.329 after four (growth of 1.010 in the last session). Comparison chain: 2.201 after two sessions and 2.282 after three, still growing (1.037 in the last session). So the weights did not stop at the square root of five. We first wrote that the two-session value of 2.24 was the equilibrium; that was wrong, and a skeptic review caught it (document 12).
- GUESS: the plateau is about 4 percent above the prediction, near 2.33 to 2.35 (squared ratio about 5.4 to 5.5, not 5), with a time constant close to the model's. The skeptic's confidence in this was about 60 percent; two more sessions would settle it.
- FACT [production chain, session 12]: the final residual-stream RMS grew about 4.5 times (measured on eight windows of 1,024 tokens), the skip scalars rose 2.1 to 7.5 times by layer, and the gradient size of the hidden matrices fell from 2.37 to 0.74.

## A prediction written before the weights were read

- FACT [note written at 8:57 PM on 29 October, before any probe or weight was read]: if updates are roughly uncorrelated with a matrix's weights, the squared norm settles at a level proportional to one over the decay. The time constant is one over twice the step size times the decay: 250 steps at 0.1 and 1,250 steps at 0.02. A fivefold cut should raise the RMS by the square root of five, 2.24, at equilibrium (the observed plateau turned out about 4 percent higher; see above).

| Test | Predicted | Pass band | Observed | Result |
|---|---|---|---|---|
| Comparison chain, first switched session | 1.909 | 1.81 to 2.01 | 1.955 | pass |
| Production chain, first switched session | 1.954 | 1.85 to 2.05 | 2.013 | pass |
| Production chain, second switched session | 1.103 | within 0.03 | 1.113 | pass |
| Comparison chain, second switched session | 1.116 | within 0.03 | 1.1255 | pass |
| Absolute matrix RMS, production chain, session 11 (query-key-value, fc, pr) | 0.005 to 0.009 | none | about 0.032 | fail |

- FACT (the table above; one seed per chain). The absolute level was about four times the noise-only estimate in RMS. The ratio prediction did not depend on that estimate, and it held.
- GUESS: a systematic (not random) update term may explain the absolute level; this was not tested.
- GUESS (two points only): a fit of the two observed ratios gave an equilibrium squared ratio near 5.4 and a time constant near 1,300 steps, close to the predicted 5 and 1,250.

## The loss drop and the skeptical reading

- FACT [validation, 2M tokens, one seed per chain]: production chain 1.2520, 1.2485, 1.2478 and 1.2495 at sessions 8 to 11, then 1.2120 at 12, 1.2038 at 13 and 1.2026 at 14. Comparison chain 1.2689 at session 11 and 12, then 1.2264 at 13 and 1.2111 at 14.
- FACT: the production step at session 12 was -0.0375 bits per byte (-0.1188 nats). The comparison step at session 13 was -0.0425 (-0.1343 nats) with the decay change alone.
- FACT [skeptic checks, CPU fp32 on saved snapshots, 108 windows of 512 tokens per set, one seed]: across nine held-out sets, the production chain gained 0.248 nats. The decay part was 0.107 (43 percent) and the mixture and start-state part 0.141 (57 percent). On the validation text, the mixture change cost about 0.005 to 0.009 bits per byte.
- FACT [leakage check]: no validation document had 20 percent or more of its 13-grams in the training streams at session 12. The drop was the same on the half of the validation file that the trainer never scored (difference 0.010 plus or minus 0.013 nats).
- FACT [timing, comparison chain, session 13 against 12]: the two loss curves were equal over the first 300 steps (difference +0.003 nats) and then split (-0.31, -0.35 and -0.18 in later blocks). The gap opened inside the session, which points to an optimizer effect rather than easier text.
- Skeptic verdict (the checks are FACT; the split is GUESS): the drop is real and caused by the decay, not by the mixture, an artefact or recency. It is an uncooled gain: the matrices doubled, so the effective learning rate roughly halved, which is a partial anneal.
- FACT [one cooled session from the session-11 checkpoint, one seed, shared by the arms]: decay 0.1 gave 1.1474, and a same-seed replicate gave 1.1475; decay 0.02 gave 1.1422 (-0.0052); decay 0.002 gave 1.1415 (-0.0059). The seed noise at 45M is 0.0046 per run, so these gains are the size of the noise and are not established.
- FACT [settling test, one seed]: a cooled session from the session-13 checkpoint at decay 0.02, 753 million tokens in total, read 1.1376 (SE 0.0021). The skeptic's ranges were set in advance: 1.12 to 1.14 means only the anneal, 1.08 to 1.10 means a real gain. The result falls in the first range.
- GUESS: the final cooled gain of the lower decay is about 0.005 to 0.01 bits per byte, the skeptic's estimate.
- FACT [rule change, disclosed in the decision notes]: during the step-down, stream growth stopped counting as a revert trigger, because the old count would have fired by construction.

## Head size and the head-decay fix

- FACT [head check, 16 windows of 1,024 tokens]: output-head RMS 0.6343 at session 8, 0.6948 at session 11, 0.7154 at session 12, 0.9355 at session 27 and 0.9472 at session 28. The head had no decay, and it kept growing.
- FACT: an earlier law, RMS equal to a constant times the step size (0.008) times the square root of the step count, with the constant 0.745, was wrong. The constant fell from about 0.77 to about 0.68 between about 100 and 450 million tokens. The notes replaced the law with a fitted curve; the mechanism was not tested (GUESS: a restoring force from the loss).
- FACT [revised trainer, insert-only changes]: an optional separate decay for the head, a head learning-rate multiplier and a z-loss option, all off by default. With the defaults, the trainer matched the previous revision on the CPU checks.
- FACT [after head decay]: head RMS 0.9472 at session 28, 0.863 at 30, 0.783 at 31, 0.7178 at 33 and 0.7200 at 36. Share above 10: 0.1552 at 28, 0.1431 at 30, 0.1438 at 31 and 0.1288 at 36. Cap dependence: 1.852 at 28, 1.8245 at 30, 1.8385 at 31 and 1.95 at 36, against a trigger of 2.3.
- FACT: the share above 10 reached 0.1600 at session 27 (one strike; the revert rule needs two checks in a row), then fell.
- Band chosen: 0.70 to 0.78 (decision notes). If the head falls under 0.70, the head decay is lowered. At session 36 it sat at 0.72, near the lower edge.
- FACT [one earlier snapshot, session 14, head-decay playbook run]: validation loss was lowest with the head scaled to 0.85 of its unmitigated size (-0.014 nats), and stayed below the unchanged value down to 0.75.
- FACT [regime after the switch]: matrix growth per session fell from 1.131 at session 27 to 1.090 at 28 and 1.038 at 31. Stream growth stopped by session 31 (1.002 per session). The largest skip scalar was 16.65 at session 27 and 30.45 at session 36; it was flagged against a threshold of 25, watched, and not acted on.

## The proxy tests

- FACT [setting]: a proxy of 4.46 million parameters (6 blocks, width 160, vocabulary 4,096), trained on 281 million tokens (63 per parameter) with a 10 percent square-root cooldown. Decay values are given in production-equivalent units, rescaled from the production values.
- FACT [sweep on the home 2060 card, 12 valid runs, run-to-run noise 0.001 to 0.002]: production-equivalent decay 0.1 gave 1.5917; 0.02 gave 1.4986; 0.008 gave 1.4519; 0.003 gave 1.4256; 0.001 gave 1.4128; 0.0003 gave 1.4076, the lowest; the fitted optimum was 0.00054.
- The 1:05 AM verdict on this sweep read: go lower to 0.0003; optimum not bracketed.
- FACT [first hosted round, invalid]: the control at 0.0003 gave 1.4951, outside the window 1.4040 to 1.4120 that the home card gave. At the same training loss, a compiled evaluation scored 1.5950 and an uncompiled one 1.5167 (training loss 3.145 against 3.142), a bias of 0.078 bits per byte from compilation in that environment. The control run caught it before any decay conclusion was drawn, and the round was discarded.
- FACT [corrected rounds, uncompiled evaluation, two seeds each]: control at 0.0003 gave 1.4082 (seed 1) and 1.4050 (seed 2); decay 0.0001 gave 1.4080 (seed 1) and 1.4041 (seed 2). The differences, 0.0001 minus 0.0003, were -0.0002 and -0.0009, mean -0.0006. The noise band was 0.0008 to 0.0015, and the seed spread of one arm was 0.0039 to 0.0041.
- FACT [verdict under the pre-registered rule]: the optimum is flat between 0.0001 and 0.0003. Going below 0.0003 buys nothing at proxy scale.
- GUESS: the proxy exaggerates the uncooled effect of the 0.1 to 0.02 step by about 2.4 times, so the production gain may be 0.01 to 0.09 bits per byte.
- FACT: the production chain stayed at 0.003, ten times the top of the flat range, while the next step waited.

## The literature on weight decay

- FACT [search as recorded in the literature note, 29 October]: no primary paper was found that compares a Muon run's constant-rate phase with its cooled endpoint. This is an absence in a search, not a proof.
- PUBLISHED: D'Angelo et al., arXiv 2310.04415 (2023). In one GPT-2-size run, decay gave higher training loss early and lower loss late, so the order of two stable-phase runs can invert later. Graded A by the project's pass.
- PUBLISHED: Kosson et al., "Rotational Equilibrium", arXiv 2305.17212 (2023). With decoupled decay, the weight norm settles where decay and updates balance, and a change in decay takes on the order of one over twice the step size times the decay to show. Graded A.
- PUBLISHED: Liu et al., "Moonlight", arXiv 2502.16982 (2025). Muon with decay 0.1 at a peak step size of 4.2 times 10 to the minus 4. Our per-step pull is about 48 times theirs (arithmetic from the settings).
- PUBLISHED: Bergsma et al., arXiv 2505.13738 (2025). A fitted power law for the best decay timescale against tokens per parameter, for AdamW with decay to zero. The project's end-of-run range (0.0015 to 0.003 at this step size) comes from three such readings (GUESS).
- PUBLISHED: Hägele et al., arXiv 2405.18392 (2024). A cooldown over 10 to 20 percent of training matches or beats cosine decay. This is the basis of the planned cooldown length.
- PUBLISHED: Loshchilov and Hutter, arXiv 1711.05101 (2017). Decoupled weight decay, the form used here.
- Caveat: the papers above were read by the project's literature pass. The values are as recorded in the project's note and were not re-read for this document.
- FACT [skeptic pass on the literature note]: of 54 checked rows, 33 were confirmed, 9 overstated, 6 wrong and 6 unverified. Wrong rows included a claim that a paper did not state its schedule (it does), a claim that a Muon paper's coupled form was not comparable (it has the same per-step form), and a claim that no Muon paper gives a tuned decay between 100 million and 1 billion parameters (one does).

## Errors the skeptic reviews found in our own claims

1. The 8:34 PM reading that the cooled gains were "real, 30 times the noise" was wrong. The replicate used the same seed, slice and settings as the control, so it measured run-to-run nondeterminism (0.0005 nats), not seed noise. The cooled gains were the same size as the noise (FACT; decision notes, item 15).
2. The veto on the share of logits above 10 was built from one same-seed pair. Under no effect it fires about 15 percent of the time per arm, so the arms were marked inconclusive, not rejected (FACT, arithmetic and check).
3. The switch to 0.02 was made at 8:19 PM, before the veto line of the same report was read. The decision notes call this a slip (FACT).
4. The 1:05 AM proxy verdict (go lower; not bracketed) was overtaken. The first hosted round was invalid, and the corrected rounds found the optimum flat between 0.0001 and 0.0003 (FACT).
5. The constant-0.745 head law was wrong; the constant fell (FACT, see the head section).
6. The stream-growth trigger was dropped during the step-down (FACT, see the cooled section).
7. The first slim copies after the head change failed; there were no copies for sessions 30 and 31 until it was fixed. This affected analysis, not training (FACT).
8. The absolute matrix-size prediction failed by about four times in RMS (FACT).
9. The literature note had 6 wrong and 9 overstated rows out of 54 (FACT).

## What this does not show

- The final cooled value at the lower decay. There is one cooled session at each setting, with one seed each.
- That the lower decay helps beyond the validation family and the nine skeptic sets. Benchmarks were not measured again after the switch (FACT, skeptic note).
- That the proxy optimum holds at production scale. The proxy has 4.46 million parameters, 6 blocks and a 4,096-token vocabulary.
- That the skip scalars are stable. They have no decay, and they were still growing at the last note read.
- That the equilibrium model explains the absolute size of the matrices. It did not.
- That the layer-level findings in the interpretability notes are robust. They rest on one seed and small probe sets.

## Open questions

1. Does a cooled run at decay 0.003 end below a cooled run at 0.02? This needs the planned cooldown or a branch test.
2. Should the step to 0.0015, and then 0.0007, go ahead once the head band holds? The notes lean to waiting; the proxy leans to lower.
3. Will the skip scalars settle, or will they need a brake of their own? They are in the monitor but have no decay.
4. Is the 0.70 to 0.78 head band the right target? The playbook's best point (0.85 of unmitigated size) lies above it, and its lower edge is 0.72.
5. How much of the switch-session gain survives the cooldown at the final decay?
