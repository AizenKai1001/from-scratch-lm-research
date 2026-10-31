# What we learned by looking inside the 143M model

## Summary

We ran small, CPU-only studies on saved checkpoints of our 143-million-parameter model: numbered studies 21 to 42, plus a summary page that gathers studies 00 to 30. Most studies were re-checked by a separate skeptic pass that recomputed the numbers from the raw outputs. The numbers almost always reproduced, but many readings did not, and several headline readings were corrected or withdrawn. The clearest results concern timing: copying develops between 3M and 30M tokens, and with a previous-token key shift induction heads are present by 3M tokens (the cause is not shown), although the shift's loss advantage shrinks by 30M. The weight-decay switch roughly doubled the main weight matrices within one session, while the J-lens replication did not show a workspace-like subspace (its comparison null was invalid).

## Scope, labels and setting

- Basis: our own interpretability notes 21 to 42, their skeptic reviews and a summary page, all written 28 to 30 October 2026. The notes are not published; this document is meant to stand on its own. Studies 27 and 35 have no note. No new runs were made for this document.
- Labels: FACT means measured in our own runs, with the setting given. PUBLISHED means taken from a source we checked, named where used. GUESS means reasoning that was not tested. Verdicts are quoted from the reviews: HOLDS is confirmed; HOLDS WITH CAVEAT and OVERSTATED are narrowed; WEAKENED narrows the conclusion; WRONG and CONTRADICTED are corrected, and WITHDRAWN means dropped.
- The model (FACT, read from the code and checkpoints): 12 layers, width 640, 5 heads of 128 dimensions, rotary positions, QK-norm, a 1,024-token context, a 32,768-token vocabulary, ReLU-squared MLPs with 2,560 units, value embeddings feeding layers 0, 1, 10 and 11, an untied output head with softcap 15, and two learned scalars per layer. Model A is an earlier, separately trained run of 2.29 billion tokens (1.0075 bpb on validation); model B is a 50-million-token run (1.2523 bpb). bpb is bits per byte. LAMBADA is a last-word prediction test.
- The ladder is 16 runs: the recipe (a value residual path plus a short-context-first warm-up) and a plain control, at 1, 3, 10 and 30 million tokens, with two seeds each. The key shift is a production setting in which head 0 of each layer takes half of its key from the previous token. Both production chains use the recipe with the key shift. At the time of these studies chain 1 used data plan B in bf16 on the newer card (it moved to plan C70 at session 12), and chain 2 used data plan A in fp16 on the older card (document 06).

## Skeptic outcomes by review

| Review | Studies | Result, as the review counted it |
|---|---|---|
| 00 to 10 and index | 00 to 10 | 71 claims: 43 supported, 26 weak, 2 contradicted. Index: 2 of 3 statements contradicted. Plan: 2 of 14 predictions met, 10 missed |
| 11 to 20 | 11 to 20 (no note 18) | 56 claims: 31 supported, 17 weak, 4 unsupported, 1 contradicted, 3 not checked |
| 21 to 25 | 21 to 25 | 64 claims: 41 supported, 12 weak, 4 contradicted, 7 not checked |
| 26 to 30 and autopsy | 26, 28, 30, autopsy | 69 rows: 39 hold, 16 hold with caveat, 8 weakened, 6 wrong |
| 31 to 34, plus a re-check of parts of 32, 33 and 36 | 31 to 34 | Verdicts per note; all numbers matched except three. Corrected: 3 readings in 32, 2 in 33, 3 in 36 |
| 37 to 39 | 37, 38, 39 | 45 claims: 26 confirmed, 13 overstated, 3 wrong, 3 unverified |
| 40 | 40 | 38 rows: 22 confirmed, 9 overstated, 1 wrong, 6 unverified |
| 41 | 41 | 55 rows: 38 confirmed, 9 overstated, 3 wrong, 5 unverified |
| 42 | 42 | 38 claims: 27 confirmed, 5 plausible, 4 unsupported, 2 wrong |

## 1. The key shift and its mechanism

**Study 36, key shift and circuit formation.** Question: does the key shift change when circuits form between 10M and 30M tokens?
- Finding (FACT; seed 14 pair, seed 15 as a second pair; 200 LAMBADA items): with the shift, induction heads exist at 3M tokens (best score 0.856 against 0.003; 9 heads above 0.3 against none). Copy gain is 0.68 nats against 0.14 at 3M, and 5.03 against 0.90 at 10M. Switching the shift off at inference removes the induction and most of the copy gain, and costs 0.10 to 0.42 bpb on 256-token windows from 3M to 30M. The loss gap in the run files shrinks from -0.080 at 10M to -0.010 at 30M.
- Verdict (review): numbers reproduce. The 30M prediction was corrected: the gain is -0.010, not about -0.006, and the 45M check was never run. The LAMBADA gain at 10M (+0.025, SE 0.013) is not established.
- Does not show: whether the shift or the trainer version (the baseline used an older trainer) causes the change, or anything after 30M.

**Study 37, weights of the key-shift models.** Question: how does the shift change the weights and activations of the recipe model?
- Finding (FACT; one seed per arm, 30M tokens): the stable rank of the query, key and value matrices is 79 to 94, against 161 for a random matrix of the same width. The shift moves the layer-11 skip scalar to 0.30 (0.56 without it) and the layer-10 scalar to 0.29 (0.09). No first-token sink: the largest head puts at most 1.6 times the uniform share on token 0.
- Verdict (review): one claim WRONG (see the taken-back section); the skip-scalar split replicated on two more seeds.
- Does not show: the cause, or the size of the stable-rank gap, which is a guess until there is a second seed.

**Study 39, head atlas of the key-shift model.** Question: what do the 60 heads do in the shifted model, compared with a same-recipe control?
- Finding (FACT; seed 15, 30M tokens, one seed per arm, 8 natural windows of 512 tokens): the five highest induction scores in the shifted model are all head 0 (layer 4 at 0.650, layer 10 at 0.458); the control's best is 0.405. Zeroing layer-10 head 0 costs +0.0346 bpb (SE 0.0091; positive in 8 of 8 windows), against +0.0217 for the control's best head. Mid-layer positional previous-token heads are weaker with the shift (layer 3 head 0 at 0.066, against 0.492).
- Verdict (review): OVERSTATED. The layer-4 head-1 drop (0.61 to 0.22) is a one-seed observation. Across seeds, the best previous-token score of three shifted 30M models (0.47 to 0.51) is below all five unshifted 30M models (0.65 to 0.75). The layer-4 head-0 cost is +0.0071, not +0.0039.
- Does not show: the mechanism. Reading head 0 as a copy route is a GUESS from the trainer code and the scores; nothing was traced.

## 2. Heads, induction and production-chain circuits

**Study 23, head formation on the ladder.** Question: at which budget do previous-token, induction and copy heads form, and does the recipe change the order?
- Finding (FACT; ladder runs with seeds 11 and 14; head scores on 16 windows): no previous-token head scores above 0.4 at 1M or 3M. At 10M three of four runs have one; the seed-11 control's best is 0.37. Induction heads (score above 0.3) are absent at 10M (best 0.19) and present in all four runs at 30M (0.59 to 0.83). Copy benefit at 10M is 1.84 and 1.39 nats for the recipe and 0.31 and 0.18 for the control; at 30M it is 6.2 to 7.3 nats in all four runs, against 8.2 for A and 7.9 for B.
- Verdict (review): HOLDS WITH CAVEAT. The paired re-check gives larger 10M copy benefits (2.06 and 1.62 nats); 78% to 82% are order-specific for the recipe, while the control's benefit is mostly bag-of-tokens. The 10M head thresholds are WEAK (scores sit near the 0.4 threshold, on 16 windows with no error bars).
- Does not show: formation between the four budgets, or the separate effects of the two recipe parts. Seed and card are confounded.

**Study 38, production-chain circuits over tokens.** Question: how do the circuits of the two production chains develop with tokens, compared with the ladder?
- Finding (FACT; chain 1 at 101M, 152M and 202M tokens; chain 2 at 45M and 90M; 32 random-token blocks for copy): the best induction head is layer 3, head 0 in chain 1 (0.835, 0.831, 0.819) and layer 5, head 0 in chain 2 (0.729, 0.734). Chain 1's copy gain is 3.50, 7.98 and 5.10 nats. Trainer validation loss is 1.3133, 1.2916 and 1.2775. Attention on key 0 is 0.8% to 2.2% per layer across the seven runs in the note: no sink.
- Verdict (review): mixed. Numbers reproduce. "Non-best induction heads fall" holds only from 152M to 202M and in chain 2; four rise from 101M to 152M. The layer-0 norm claim (a GUESS) is WRONG.
- Does not show: a trend. This is one run at a constant learning rate, and the review treats the swings as snapshot noise (see study 40).

**Study 40, production-chain circuits, extended series.** Question: which changes across 13 snapshots are trend and which are noise, and do the two data plans differ at equal tokens?
- Finding (FACT; 13 snapshots, 64 probe windows): chain 1's best head stays at layer 3, head 0 at all seven of its checkpoints (0.818 to 0.843; slope +0.001 ± 0.004 per 100M tokens). Chain 2's best head rises from 0.729 to 0.858 (slope +0.065 ± 0.008 per 100M; one seed). Copy gain runs from 3.5 to 8.0 nats in chain 1 and 5.3 to 9.1 in chain 2. First-token share is 0.012 to 0.014 at all 13 snapshots, against 0.0201 for uniform attention. At eight matched token counts chain 1 has lower bpb by 0.018 to 0.032 (GUESS-level, because the other chain was interpolated).
- Verdict (review): the swings are real snapshot-to-snapshot movement, not window noise. The data-plan advantage holds on the validation file used, but on four other held-out corpora plan A is lower by 0.065 to 0.21 nats per token (OVERSTATED).
- Does not show: which data plan causes the gap. The layer of the best head varies by seed (layers 5, 2 and 4 across three ladder seeds).

## 3. When structures form on the ladder

**Study 24, MLP sparsity and first-token sink.** Question: when do sparsity, dense strong units (active on over half of positions, with large typical values), dead units and first-token outliers form along the ladder?
- Finding (FACT; 16 ladder runs plus A and B; 32 validation windows): the active share of MLP units is 48.5% at 1M, 26% (recipe) and 33% (control) at 10M, and 29% and 30% at 30M (A 14%, B 35%). Dense strong units appear at 3M and reach 20 and 21 (recipe) and 17 and 19 (control) at 30M (A 19, B 23). Dead units: none in the 16 ladder runs or in B, and 16 in A. No first-token sink by 30M: the largest head puts 0.017 to 0.041 of its attention on token 0 (B 0.042, A 0.29). The recipe lowers loss at every budget by 0.005, 0.028, 0.060 and 0.017 bpb at 1M, 3M, 10M and 30M.
- Verdict (review): HOLDS WITH CAVEAT. The window length is 256 tokens in the note's setup and 1,024 in the summary page, so the 10M recipe gain (-0.060 here, -0.041 in study 28) depends on the protocol.
- Does not show: which factor makes model A's sink and dense units: its data, recipe, weight decay (0 in A, 0.1 in B) or learning rate. That is untested.

**Study 25, embeddings and lambdas along the ladder.** Question: when do token embeddings, value tables, scalars and block spectra take their final shape?
- Finding (FACT; weights only, 16 ladder runs): the input-row norm tracks training frequency (Spearman 0.68 at 1M and 0.80 at 30M, recipe). The stable rank of the attention matrices falls from 258 at 1M to 165 at 30M (random 264 at 1M). At 30M, king-queen is rank 1 in all four runs and Paris-Rome is not solved (ranks 17 to 93). At 10M the recipe lowers the scalar lam0 at layers 1 to 3 to 0.85 to 0.94 (control 0.99 to 1.07).
- Verdict (review): HOLDS WITH CAVEAT. At 30M both arms show the same U-shape in the re-injection scalar, so only the 10M suppression is recipe-specific.
- Does not show: much about meaning in the embedding table. The analogy tests are three items.

**Study 26, ladder representations timeline.** Question: at which budget do residual representations, token and position information and the logit-lens prediction form, and do the two seeds converge?
- Finding (FACT; 16 ladder runs and two references): at 1M the model beats a unigram model (2.08 against 2.44 bpb on the same windows), and it beats a bigram model overall at 3M (1.890 against 1.945). No-space letters reach bigram level at 10M (5.40 against 5.65 nats). The second-copy loss on a repeated random block falls from 12.66 nats at 10M to 6.38 at 30M. On LAMBADA the model reaches 0.133 at 30M, against 0.120 for the control and 0.125 for B.
- Verdict (review): HOLDS WITH CAVEAT. Linear CKA between seeds (a similarity score, 1 means identical) is mostly token identity: at the embedding it equals a within-token shuffle by construction; at layer 11 at 1M it is 0.423 against a shuffle null of 0.416; context-specific similarity appears from about 10M.
- Does not show: the recipe's effect apart from its data order. The note's "grammatical clauses by about 10M" is a GUESS.

**Study 28, budget ladder behaviour.** Question: what can each budget do, and when does each ability appear?
- Finding (FACT; 16 ladder runs; loss gaps on 64 windows of 256 tokens; loops on 24 prompts): the recipe lowers loss by 0.009, 0.027, 0.041 and 0.015 bpb at 1M, 3M, 10M and 30M. Greedy text loops on 24 of 24 prompts at 1M and 3M, 23 at 10M and 19 to 21 at 30M (B 16, A 8). Loss on the token after end-of-text is 6.87 nats at 1M and 6.15 at 30M.
- Verdict (review): HOLDS WITH CAVEAT. The 64-window gaps are 0.011 to 0.016 smaller at 10M than the full-evaluation gaps (-0.050 bpb in the run file, -0.053 on 64 windows of 1,024 tokens), which are the headline numbers.
- Does not show: anything between the four budgets. Loop counts rest on 24 prompts.

## 4. Copying, value embeddings and memorisation

**Study 30, value embeddings as the late copy path.** Question: do late layers fetch the identity of earlier tokens directly through the value-embedding path?
- Finding (FACT; models A and B; 32 windows of 256 tokens): removing the layer-10 value embedding costs +0.055 bpb in A and +0.059 in B. Of that cost, 73% (A) and 84% (B) falls on positions whose target occurred earlier in the window (40% of positions). Replacing both late tables with the frequency-weighted mean row keeps 83% (A) and 91% (B) of the cost of removing both. On the ladder (recipe, seed 11), removal costs at most 0.0002 bpb at 1M and 3M, 0.027 bpb (recipe) on copy positions at 10M and 0.099 (recipe) at 30M.
- Verdict (review): WEAKENED. The mean-row control also removes most of the injected norm, so the identity shares are upper bounds. Recomputed per-token nats were 3% to 4% off.
- Does not show: whether the route is copy-specific or a frequency prior. Both readings fit (GUESS). The second seed's lane was not analysed for this route.

**Study 31, a clean test of memorisation.** Question: does the loss gap between training windows and unseen windows grow with training on the same data?
- Finding (FACT; 384 windows of 1,025 tokens; one seed): in the matched design, the seen-minus-unseen change from the first pass to model A is +0.107 ± 0.153 nats (95% interval -0.19 to +0.41). The first design's +0.445 ± 0.156 is mostly source composition. About half a further pass of the data gives a seen advantage of +0.025 ± 0.009 nats, against placebo splits of about ±0.01.
- Verdict (review): HOLDS WITH CAVEAT. The earlier claim of about 0.4 nats of memorisation is NOT ESTABLISHED, because it compared two different text sets.
- Does not show: the size of first-pass memorisation. The 0.4 value lies inside the interval, so it is not excluded; training-seed noise is not in the error bars.

## 5. Weights, spectra and the weight-decay switch

**Study 21, singular directions through the vocabulary.** Question: do the top singular directions of each weight matrix, read through the vocabulary, name token classes, and does removing one change the loss?
- Finding (FACT; models A and B; 16 windows of 1,024 tokens, seed 21): the trained spectra are more concentrated than random. The participation ratio (a count of the directions that carry the energy) of the first MLP matrix is 456 (A) and 429 (B), against 512 for a random matrix of the same shape. Removing the top output direction of the last MLP layer costs A 1.80 nats per token (+0.568 bpb) and B 1.14 nats (+0.359 bpb).
- Verdict (review): WRONG. The numbers reproduce, but 94% of A's effect is a constant offset (subtracting only the mean costs 1.69 nats). The digit-class reading is CONTRADICTED: capitalised word starts lose 6.28 nats and digits 2.74. The "27 of 40 interpretable" rating has no baseline (12 random tokens meet the rule 54% of the time). The class reading is withdrawn.
- Does not show: any meaning for the top directions. One reader rated 40 directions.

**Study 41, weight-decay switch and matrix norms.** Question: did lowering the decay on the hidden weight matrices (optimised with Muon) from 0.1 to 0.02 change the production layers the way a simple decay model predicts?
- Finding (FACT; snapshots of both production chains; 48 hidden matrices): the matrices roughly doubled within one session (median ratio 1.96 and 2.01; range 1.8 to 2.4), and reached about 2.31 and 2.28 after three switched sessions. Layer-0 units that fire on under 1% of tokens rose by +214 ± 13 (chain 1) and +86 ± 7 (chain 2) at the switch, over five window sets. Layer-10 head-0 induction rose 0.161 ± 0.010 in chain 2, then stayed flat.
- Verdict (review): 55 rows: 38 confirmed, 9 overstated, 3 wrong. The equilibrium ratio is about 5.3 to 5.5, not below 5. The angle-per-step numbers were about five times too large (measured 0.0129 radians before the switch and 0.0064 after). The decay premise fails for all 48 matrices, but the ratio prediction holds as a test.
- Does not show: the cause of the rise. "Acts like a slower learning rate" is not supported as stated, because residual norms rose four to five times in the same session.

**Study 42, layer-0 rare units after the decay switch.** Question: are the layer-0 units that fire on under 1% of tokens token-identity detectors, and does the current token predict their firing better than a matched random readout? (AUC: 0.5 is chance, 1.0 is perfect.)
- Finding (FACT; 64 windows of 1,024 tokens from the held-out validation region, in two halves; chain 2 at and before the switch, chain 1 as replication): the predictions were not met. Median out-of-sample AUC of the rare units is 0.684 and 0.681 in chain 2, against 0.699 and 0.697 for matched random directions. In chain 1 it is 0.775 and 0.784, against 0.831 and 0.834. Only 15% to 42% of units beat their matched control.
- Verdict (review and erratum): CORRECTED. Of chain 2's 227 rare units, 174 are weight-decayed rows with outputs near 1e-18, and the trainer's half-precision forward gives exactly zero for 1,473 layer-0 units. The claim that the plan came before the probes is not supported.
- Does not show: any effect on loss (nothing was ablated); a dose-response (three checkpoints only); or robust counts under lower precision, which the lane check shows are fragile in chain 2.

## 6. Causal tests on units and components

**Study 32, neuron atlas v1.** Question: which MLP neurons of model A matter causally, and what does each one do?
- Finding (FACT; model A only; 60 units, the top five per layer by a first-order score; 16 windows of 512 tokens for attribution): 11 of the 60 units have zero-ablation effects of 0.002 nats per token or more, measured on the eight windows where each fires most; ten are at least two standard errors above zero. Unit 214 in layer 11 keeps 43% of its damage when replaced by its mean. Verdict counts: dense offset 6, first-token 6, monosemantic 5, class 4, unclear 39.
- Verdict (review): WEAKENED, with three corrected readings: "close to uninformative" (a rank correlation of 0.25 over 200 units, p about 0.0003); "units 214 and 697 are equal within error" (paired difference -0.026, SE 0.0065); and the position-0 caveat for first-token units (wrong for four of five).
- Does not show: what the 39 unclear units do. The matched controls matched only active share, so specificity is untested. Model B was not run.

**Study 33, steering and surgery.** Question: what changes when we steer along contrast directions, and when we scale, zero, swap or ablate components?
- Finding (FACT; models A and B; steering on 16 held-out windows of 256 tokens; surgery on 16 windows of 512): steering moved the next-token prediction in the predicted direction in 27 of 30 cells, against equal-norm random and shuffled-label controls; the strongest is digit steering in A at layer 6 (+8.0 log-odds at +2). At +2 the concept effect costs 0.2 to 6.9 nats per token for cap, digit and newline directions. Zeroing the residual mix at layers 1 to 9 costs 3.4 to 15.8 nats per token in A. Zeroing the late value-embedding injection costs 0.192 nats, and a norm-preserving shuffled-row control costs 0.588.
- Verdict (review): HOLDS WITH CAVEAT. Two sentences calling A and B linearly related are WRONG (the baselines carry scale errors of 149 to 207 times). The head 5.2 cost is 97% constant output. The sink-rescale prediction failed (0.005 and 0.001 nats against a predicted 0.05); matched controls show the sink is not a special loss channel.
- Does not show: component transplants between A and B, which did not transfer cleanly.

## 7. Representations and the workspace claim

**Study 34, J-lens replication.** Question: can the J-lens (second-hand citation: the method of arXiv 2607.15495, July 2026, as cited in our note; the paper was not re-read here) and its claimed workspace-like subspace be replicated on model A, and does such a subspace appear?
- Finding (FACT; model A; fit on 16 windows of 128 tokens; readout on 64 held-out windows): the J-lens does not improve next-token prediction (top-1 0.03 to 0.09, against 0.13 to 0.26 for the logit lens). Its top words appear 2 to 16 tokens later in the text more often: enrichment 3.66 at layer 6, against 1.23 for the logit lens. At k = 10 the J-space takes 4.1%, 10.0% and 12.0% of centred variance at layers 3, 6 and 9; a random subspace takes 1.7%. The copy-swap test was uninformative: the clean model chose the right colour on 34 of 64 prompts.
- Verdict (review): WEAKENED. The layer-6 enrichment falls to 3.02 on a refit with token-shuffled windows (paired difference +0.63, CI -0.31 to +1.88). The random-subspace comparison is invalid: a greedy fit on pure noise reaches 0.167 at k = 10, against 0.014 for a random subspace.
- Does not show: a workspace-like subspace in the 143M model. The fit used fewer windows than the cited method (the note cites 1,000 sequences).

**Study 29, mode connectivity.** Question: do the recipe and control runs (seed 11, same data order) lie on one linearly connected low-loss line, and does the 30M run lie in the 10M run's basin?
- Finding (FACT; 8 windows of 512 tokens; the barrier is the highest loss on the straight line above the line joining the two ends; 1 nat per token = 0.316 bpb): the barrier is 0.78 nats at 10M (recipe against control) and 1.38 at 30M. Recipe 10M against recipe 30M gives 0.40 nats. Two seeds on different cards give 1.40 nats. A copy with 1e-3 noise gives no barrier.
- Verdict: no skeptic review covers this note.
- Does not show: non-linear paths. Diagnostics ruled out the value-embedding sign flip, the scalar gains and norm shrinkage at the midpoint. That the cause is misaligned weight matrices is a GUESS.

## 8. Methods note

**Study 22, methods catalogue.** Question: which interpretability methods we have not used are feasible on CPU for the 143M model, and what should run next?
- Finding (FACT for timing; no model results): on two CPU threads, a 512-token forward takes 0.45 to 0.53 seconds in fp32. The dictionary-learning cost (about two CPU hours per 16k-latent model on 4M tokens) is a GUESS. Proposed next tests: circuit tracing, linear mode connectivity and a tuned lens; mode connectivity was later run as study 29.
- Verdict (review): HOLDS WITH CAVEAT. Five of seven literature claims in the note were checked against abstracts and documentation and hold (one with a protocol caveat); two were not checked. The claims are not restated here.
- Does not show: any result about the model. Circuit tracing and the tuned lens were still not run at the last summary.

## Claims we had to take back

Each item is a FACT from a re-check unless it is marked GUESS.

- Study 21: "the digit class is hit hardest" (CONTRADICTED: capitalised starts are hit hardest); "the spectra are spread out" (CONTRADICTED: they are more concentrated than random); "same-layer overlap is barely above cross-layer" (CONTRADICTED: the sign depends on pool size); the class reading (WITHDRAWN).
- Study 30: earlier attention figures of -1.29 and -1.14 nats (A) and +0.07 and +0.42 (B), which had no stated definition (WITHDRAWN by the note). The 83% and 91% identity shares were upper bounds (WEAKENED).
- Study 31 and the summary page: the earlier 0.4-nat memorisation figure (NOT ESTABLISHED: it compared two text sets).
- Study 36: the 30M prediction (about -0.006 expected, -0.010 found), and a 45M check that was never run (CORRECTED); a last-block CKA row for seed 15 that reverses the sign (CORRECTED); "the induction score does not drop" (CORRECTED: the best score rises).
- Study 37: "the shift lowers the layer-10 attention write by 0.29" (CORRECTED: across pairs it ranges from -0.02 to -1.00, a no-shift pair gives +0.79, and only the sum over layers 10 and 11 replicates).
- Study 38: "the layer-0 residual norm follows the embedding row norm" (CORRECTED: scaling the table by 1.5 leaves the block-0 residual at 180.151 before and after). "Non-best induction heads fall while the best holds" (NARROWED: one interval only).
- Study 39: "the layer-4 head-1 drop from 0.61 to 0.22" (NARROWED: one seed). "Layer-4 head-0 costs +0.0039 bpb" (CORRECTED to +0.0071).
- Study 40: "chain 1 has 0.02 to 0.03 lower bpb at matched tokens" (NARROWED: true on the one validation file, reversed on four others). A sentence on the copy blocks' provenance (CORRECTED: they are uniform random token ids, not frequent validation tokens).
- Study 41: "the equilibrium ratio is below 5" (CORRECTED: about 5.3 to 5.5); "chain 1's layer-10 rise is beyond its pre-switch steps" (CORRECTED); the angle-per-step numbers (CORRECTED: about five times too large); "acts like a slower learning rate" (NOT SAFE as stated).
- Study 42: that the rare-unit count mostly reflects rare but working units (CORRECTED by erratum: in chain 2 most of the count is dead rows). The token-predictability comparisons (UNSUPPORTED: no error bar, no matched control; do not quote).
- Study 32: "close to uninformative" (CORRECTED); "214 and 697 are equal within error" (CORRECTED); the position-0 caveat (CORRECTED for four of five units).
- Study 33: "A and B are linearly related" (CORRECTED); the note's guess about head 5.2's cost (CONTRADICTED: 97% of the cost is constant output).
- Study 34: the layer-6 enrichment as evidence for a J-space (WEAKENED); the random-subspace null (INVALID).
- Studies 00 to 20 (from their reviews and the summary page): unit 2268 called an end-of-document detector (CONTRADICTED: at most 6 of its 22 strongest positions are end-of-text, and 83% of its loss effect sits at ordinary positions). The index's embedding RMS of 0.12 against 3.6 for A (CONTRADICTED: A is 0.86, and 0.12 is the control). The plan: 2 of 14 predictions met, 10 missed, 2 never tested. "57 of 60 heads" (UNVERIFIED: could not be reproduced, yet still stated as fact in one note). The HellaSwag "below the word-overlap rule" (NARROWED: true for plain accuracy only; normalised accuracy is 32.2% against 31.4%). "Only in A" claims rest on one checkpoint per recipe (WEAK). Between-seed CKA of 0.85 to 0.93 includes token identity (CORRECTED: a within-token shuffle keeps 0.39 to 0.81 of it).

## What this does not show

- No study shows that the model relies on a named token class or feature; the one rating has no baseline.
- No study covers an output-head scale or a "share above 10" monitor; the folder has neither.
- Most arms are one seed, and within the ladder the seed and the card are confounded.
- Nothing has been ablated on the layer-0 rare units, and no circuit has been traced end to end.
- Studies 32 and 34 use model A only; model B was not run for them.

## Open questions

- Which factor creates model A's sink, dense units and large re-injection scalars: its data, its recipe, its weight decay (0 in A, 0.1 in B) or its learning rate (studies 24, 25)?
- Is the value-embedding route copy-specific or a frequency prior (study 30)?
- Does the key shift's advantage hold past 30M, and is the shift or the trainer version responsible (studies 36, 37)?
- Why is the best induction head in layer 3 in one chain and layer 5 in the other, and does that depend on seed or data (studies 38, 40)?
- Do the layer-0 rare units do anything? Only an ablation can say (study 42).
- Is there a workspace-like subspace under a valid null with matched-norm candidates (study 34)?
- Why do the recipe and control lie in different basins, and is it weight misalignment (study 29)?
- Does the memorisation effect survive other seeds (study 31)? What constant output makes up head 5.2's cost (study 33)?
- The tuned lens and circuit tracing have not been run (study 22).

## How to read these results

This is a 143-million-parameter model trained with one recipe family, and the checks use between 8 and 384 windows of text on a CPU. Many arms are a single seed, and the comparison models were trained with different settings, so differences between A and B are confounded. Where the notes give standard errors, they are shown; the skeptic passes often found effects smaller than the first reading, or no baseline at all. Treat each number as a measurement in this setting, not as a general law for language models.
