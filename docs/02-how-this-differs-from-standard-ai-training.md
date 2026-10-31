# How this differs from standard AI training

## Summary

We train a 143-million-parameter language model from scratch on one 8 GB graphics card, in chained sessions of about 24 minutes of training each. The recipe borrows parts from public work (Muon, value embeddings, a partial key offset, logit soft-capping and a value residual) and adds choices of our own (very small batches, a constant learning rate with weight-decay step-downs, a planned cooldown, and partly repeated data). Against the published recipes we read, the largest differences are the data per parameter (a planned 63 tokens, against about 20 in the Chinchilla rule of thumb), the batch size (32,768 tokens, against 0.5 to 4 million), the weight decay (0.003 for the weight matrices since session 27 in our notes, against 0.1 in GPT-3 and LLaMA), and the chained session design. Two of our earlier weight-decay claims were wrong or unproven, and both corrections are in section 5.

Labels used: FACT means measured in our runs, with the setting. PUBLISHED means taken from a paper or page we opened, named in the text. GUESS means our reasoning, not tested. (arithmetic) means our own calculation from labeled inputs.

## 1. The standard recipe for pretraining

The published recipes we opened mostly share a core: an Adam-family optimizer (AdamW in LLaMA and Llama 3), a short warmup, cosine decay to a fraction of the peak rate, weight decay near 0.1, gradient clipping near 1.0, large batches, and a context of 1,024 to 8,192 tokens.

- LLaMA (Touvron et al. 2023, arXiv 2302.13971; PUBLISHED): AdamW with beta1 0.9 and beta2 0.95, weight decay 0.1, clipping at 1.0, 2,000 warmup steps, cosine decay to 10 percent of the peak rate, batches of 4 million tokens.
- GPT-3 (Brown et al. 2020, arXiv 2005.14165, Appendix B; PUBLISHED): Adam with beta1 0.9, beta2 0.95 and epsilon of 10 to the minus 8; weight decay 0.1 for all models; global gradient-norm clipping at 1.0; 375 million warmup tokens; cosine decay to 10 percent over 260 billion tokens; context 2,048; batches of 0.5 to 3.2 million tokens.
- Pythia (Biderman et al. 2023, arXiv 2304.01373; PUBLISHED): context 2,048; batches of 2,097,152 tokens; Adam, not AdamW, with weight decay 0.01. It is the counterexample to "0.1 everywhere".
- Llama 3 (Dubey et al. 2024, arXiv 2407.21783; PUBLISHED): AdamW; training starts on 4,096-token sequences and moves to 8,192 after 252 million tokens.
- GPT-2 small (PUBLISHED, the Hugging Face configuration file of the openai-community GPT-2 model): context 1,024 tokens. We did not open the GPT-2 paper itself, so its batch size and optimizer are not used here.
- SmolLM-135M (PUBLISHED, Hugging Face model card): 600 billion pretraining tokens. The card we read gives no training hyperparameters.

Ours (FACT, 143M model): Muon for the weight matrices at learning rate 0.02; Adam for embeddings, the output head and vectors at learning rate 0.008 (the embedding step multiplied by 20); batches of 32,768 tokens, about 1,549 optimizer steps per 24-minute session; context 1,024 tokens; warmup only in the first session; weight decay on the Muon matrices only (0.1 at the start, see section 5). Our batch is 64 times smaller than Pythia's and 15 to 122 times smaller than the GPT-3 and LLaMA batches (arithmetic). Our context is GPT-2's value and half of GPT-3's and Pythia's.

## 2. Data per parameter

- Kaplan et al. 2020 (arXiv 2001.08361; PUBLISHED, abstract read): the abstract argues that the compute-efficient route is very large models trained on a modest amount of data and stopped well before convergence. Their fitted exponents, 0.73 for model size and 0.27 for data, appear as the Kaplan row of Chinchilla's Table 2.
- Hoffmann et al. 2022 (Chinchilla, arXiv 2203.15556; PUBLISHED): the abstract says model size and training tokens should be scaled equally. Table 2 gives exponents of 0.50 and 0.50, 0.49 and 0.51, and 0.46 and 0.54 for their three approaches. Chinchilla itself (70 billion parameters, 1.4 trillion tokens, Table 1) is 20 tokens per parameter, and the projection table gives 20 to 22 (arithmetic). We did not find the 20-tokens-per-parameter wording in the parts of the paper we could read.
- Small and overtrained models (PUBLISHED, with arithmetic): Pythia-160M saw 299,892,736,000 tokens in 143,000 steps (model card), about 1,850 per parameter with the card's total of 162.3 million parameters. SmolLM-135M: 600 billion tokens, about 4,400 per parameter. LLaMA 7B (labeled 6.7B in its Table 2): 1.0 trillion tokens, about 150 per parameter. LLaMA 65B (labeled 65.2B): 1.4 trillion tokens, about 21 per parameter. GPT-3 175B: 300 billion tokens, about 1.7 per parameter.
- Ours (FACT): 143 million parameters; 2,095 million tokens after session 41 (read from the session records). The plan is 9 billion tokens in 177 sessions, which is 63 tokens per parameter (plan figure). So far the ratio is about 15 (arithmetic). The 20 rule would give about 2.9 billion tokens for 143 million parameters (arithmetic), so the plan is about three times that. The proxy experiments use the same ratio: 4.46 million parameters on 281 million tokens, 63 per parameter (FACT).
- GUESS: our planned ratio of 63 sits between the Chinchilla point (20) and the small-model practice of Pythia and SmolLM (1,850 and 4,400), nearer the Chinchilla point. We chose a fixed token budget and did not test the compute-optimal point.

## 3. The optimizer: Muon and what we borrowed

- Muon (Keller Jordan's write-up, kellerjordan.github.io/posts/muon; PUBLISHED, copy read without a date): for each two-dimensional hidden weight matrix, the momentum update is orthogonalized by a Newton-Schulz iteration, an approximation usually run for five steps. Embeddings, the output head and vectors stay on AdamW. The write-up reports a 1.35 times speed-up on the NanoGPT speed-run and a 35 percent gain for one record. It gives no weight-decay or learning-rate scaling rule.
- Liu et al. 2025 (Muon is Scalable for LLM Training, arXiv 2502.16982; PUBLISHED; the author block lists Moonshot AI): about 2 times the compute efficiency of AdamW with compute-optimal training. Two changes let Muon reuse AdamW's settings: weight decay is added, and each matrix's update is scaled by 0.2 times the square root of its larger dimension. Their Moonlight model (about 3 billion active and 16 billion total parameters, counting embeddings) was trained on 5.7 trillion tokens with weight decay 0.1 at every stage.
- modded-nanoGPT README (Keller Jordan and contributors, github.com/KellerJordan/modded-nanogpt; PUBLISHED): lists Muon, value embeddings, logit soft-capping lowered from 30 to 15 in one record, a partial key offset, attention-window warmup with a maximum-sequence-length schedule, and skip connections with lambda scalars. It does not list the value residual as a record; it cites Zhou et al. 2024 (arXiv 2410.17897) for it.
- Gemma Team 2024 (Gemma 2, arXiv 2408.00118, Section 2; PUBLISHED): tanh-based soft-capping of attention logits at 50.0 and of final logits at 30.0.

Borrowed: Muon for the matrices with AdamW for the rest; value embeddings; the partial key offset; the value residual; the sequence-length ramp in the first session; logit soft-capping (a tanh cap of 15 on the final logits; FACT, read from the saved settings of the session-36 snapshot); the skip scalars.

Not used, as far as the notes show: the Moonlight update-scale rule. Our Muon shape option is set to none, and we did not check its code.

Ours: the batch size, the learning-rate values (Muon 0.02, Adam 0.008, an embedding learning-rate multiplier of 20, Adam beta2 0.99), the decay schedule and the session design. The notes do not trace each value to a source, and they contain no Muon-versus-AdamW test of our own.

Evidence (FACT): the key offset gave three equal-clock pairs of -0.0052, -0.0065 and -0.0070 bits per byte. A causality check at production head size found no future-token leak; GPU numerics were not checked. At 10 million tokens (older trainer) the gain was -0.0816 over four paired seeds; at 45 million, with four seeds, it was -0.0061. The effect shrank with scale (the two budgets also differ in trainer revision and seed count, so this is not a clean comparison).

## 4. Learning-rate schedules

- Cosine with warmup (GPT-3 and LLaMA, section 1; PUBLISHED).
- Warmup-stable-decay (MiniCPM technical report, Hu et al. 2024, arXiv 2404.06395; PUBLISHED): warmup, a long stable stage at the peak rate, then a short decay of about 10 percent of the tokens; a 2.5 percent decay fell short. A decay started from a stable-stage checkpoint can reach cosine's best loss.
- Hagele et al. 2024 (arXiv 2405.18392; PUBLISHED): a cooldown of 10 to 20 percent of the steps can match or beat cosine; the fraction needed shrinks as runs get longer; a square-root cooldown beats a linear one.
- Ours (FACT): session 0 warms up over 3 percent of its clock; later sessions run at a constant rate. The planned cooldown covers the last 18 sessions (10 percent of the plan) as a square-root staircase, with the final session linear to zero, to be armed at about 8.1 billion tokens. It has not run at this scale. In an older chain, one cooled session lowered validation by 0.0154 bits per byte (the stored validation values of that chain may read about 0.4 percent high, UNVERIFIED).
- Our weight-decay step-downs (FACT): 0.1 for sessions 0 to 11, 0.02 from session 12, 0.006 from session 24, 0.003 from session 27. A step to 0.0015 is on hold.
- Relation to cosine: cosine fixes the total length in advance, while warmup-stable-decay can decay from a stable-stage checkpoint at any step (MiniCPM, Section 4.5), which suits a chain whose end may be moved. We keep MiniCPM's 10 percent length and Hagele's square-root shape, but hold the rate constant for about 160 sessions and change the decay in the middle. GUESS: the gain from the cooldown at this scale is unknown, and 18 sessions may be more than needed, because Hagele et al. find that the needed length shrinks with run length.

## 5. Weight decay

- Loshchilov and Hutter 2017 (AdamW, arXiv 1711.05101; PUBLISHED): decay is applied to the weights directly rather than through the adaptive gradient. They normalize the decay by batch size and by the number of training points and epochs, and report that the best decay depends on the total number of weight updates.
- D'Angelo et al. 2023 (arXiv 2310.04415; PUBLISHED): with normalization layers, the weight norm sets an effective learning rate, so decay acts partly like a schedule. They discuss a fast-equilibrium conjecture for the norm, which they attribute to Li et al. (2020). In their GPT-2 small runs (124 million parameters, 50,000 iterations), decay raised the loss early and caught up or helped later.
- Wang and Aitchison 2024 (arXiv 2405.13698; PUBLISHED; the version read is from 2025): the decay timescale is 1 divided by (learning rate times decay), counted in iterations. They propose keeping the timescale in epochs fixed as the data grow, which makes the decay fall in proportion to 1 over the dataset size. They caution that this came from multi-epoch training and may not carry over to one-epoch pretraining.

Our observations:

- Decay memory (FACT, arithmetic): at Muon rate 0.02 and decay 0.1, the timescale (1 divided by learning rate times decay) is 500 iterations, about 16.4 million tokens at 32,768 tokens per step; the equilibrium time constant used in document 05 is half of that product's reciprocal, 250 steps. Applying the AdamW definition to Muon is a GUESS.
- Scale (FACT): at session 12 the matrix decay fell from 0.1 to 0.02, a fivefold drop, and the RMS of the 48 hidden matrices rose by a factor of about 2.0 in the first session (2.01 in the production chain; 1.96 in the comparison chain, which made the same change one session later), by 2.24 over two sessions counted together, and by 2.33 over four. A simple balance model (the squared scale settles at a level proportional to one over the decay) had predicted the first two session-to-session ratios before they were measured, and gives a final factor of the square root of five, 2.24 (arithmetic). The weights kept growing past that value, so the model's final level is slightly low (document 05).
- Reading (GUESS): in D'Angelo's picture, lower decay gives larger norms and smaller effective steps. The notes read the 0.02 change as roughly halving the effective rate. Because D'Angelo et al. report an early cost and a later benefit, the early lead of lower decay may narrow after cooldown. Untested.
- Correction (FACT): we earlier called the one-session lower-decay gain real, about 30 times the noise. That was wrong. The replicate used the same seed, so it measured only run-to-run nondeterminism. Seed-to-seed noise at 45 million parameters is 0.0046 bits per byte per run. The one-session cooled gains of -0.0052 (decay 0.02) and -0.0059 (decay 0.002), against a control at 0.1, are the same size, so the gain is not established. The veto rule we used to support the change was also inconclusive.
- Proxy (FACT): a 4.46-million-parameter proxy on 281 million tokens (10 valid rows in the results file; a verdict record counts 12) gave final cooled bits per byte of 1.5917 at production-equivalent decay 0.1, 1.4986 at 0.02, 1.4519 at 0.008, 1.4256 at 0.003, 1.4128 at 0.001 and 1.4076 at 0.0003. The fitted optimum is near 0.00054, and an earlier verdict said the optimum was not yet bracketed. Replicates on the hosted T4 runs (after an evaluation fix, section 7) show 0.0001 and 0.0003 flat: 1.4080 against 1.4082 for seed one, and 1.4041 against 1.4050 for seed two. Going below 0.0003 gained nothing at proxy scale.
- Scaling rule (FACT): the proxy decay equals the production-equivalent value times the ratio of run lengths in steps (274,658 to 8,584, about 32). That keeps the decay memory the same fraction of the run.
- Where production stands (FACT, as recorded in the notes): matrix decay 0.003 since session 27 and output-head decay 0.003 since about session 33. The head got its own decay in the newer trainer, at 0.01 from about session 30. The matrix value is about six times the proxy optimum (arithmetic). In the first trainer, embeddings and the other Adam parameters had no decay.

## 6. Training in short chained sessions

- Typical in the sources we read (PUBLISHED, from our reading of GPT-3, LLaMA and Pythia): each describes one run with a fixed token budget and schedule, not a chain of sessions.
- Repeats (PUBLISHED): Pythia's model card says just under one epoch of the Pile. GPT-3 repeated some sources (WebText2 2.9 epochs, Books1 1.9, Wikipedia 3.4, within 300 billion tokens; Table 2.2). Muennighoff et al. 2023 (arXiv 2305.16264) report that up to 4 epochs of repeated data change the loss negligibly against unique data, that the value of added compute eventually falls to zero, and that returns diminish sharply near 16 epochs.
- Ours (FACT): plan A, which uses only the chain-two pool, reaches about 3.2 epochs by 9 billion tokens. The chain-two pool is 2.81 billion tokens of web, encyclopedia, educational and fiction text. The current plan takes 36 million tokens per session from that pool and 16 million from fresh 2025 educational web text, so about 69 percent of each session is pool text (arithmetic). A paired test put the cost of repeats at +0.025 plus or minus 0.009 nats of seen-versus-unseen gap (loss on training text against unseen text of the same sources) per extra half pass, over 96 windows. An earlier cloud run used 5.2 passes over a 361-million-token corpus, beyond the 4-epoch band.
- Comparisons (FACT): the key-offset test used equal-clock pairs; the data-plan re-check used four token-matched pairs. Over those pairs the old-validation difference favored one plan by 0.0326 nats, and eight breadth sets favored the other by 0.0426 nats; the pooled rule kept the chosen plan. GUESS: equal-token ablations are normal in small-scale work, while the published recipes we read report single runs.
- Cost (FACT): sessions train for 24 minutes, under a rule of at most 30 minutes per job. The batch-43 row shows 24.42 trainer minutes; start-up and the checkpoint save add about 0.4 minutes (estimate), and the save itself takes about 13 seconds. A five-minute cooling rest follows each session (added on 30 October), so a cycle is about 30 minutes and about 17 percent of card time is rest (arithmetic). In the batch-43 logs, other jobs that took the card's lock delayed our runs by 3.6 to 5.8 minutes each.
- Continuity (FACT, one-minute smoke chain on a second card): the mean loss of the last four steps before each boundary, against the first four after, was 5.701 against 5.746 and 4.860 against 4.836. A cold start would sit near 10.40, the natural log of 32,768. A session killed at 50 seconds left both checkpoints unchanged, and the retry succeeded.
- Correction (FACT): the earlier estimate of 3.05 days for 9 billion tokens assumed no rests. With the rests, 177 sessions take about 3.7 days (arithmetic).

## 7. Evaluation: bits per byte and benchmarks

- Bits per byte (PUBLISHED, Gao et al. 2020, The Pile, arXiv 2101.00027): the paper's preferred metric is bits per UTF-8 byte, chosen for tokenizer invariance and because characters are ambiguous in Unicode.
- Why it is not comparable across tokenizers and data (GUESS, extending the paper's caveat): the byte count removes the tokenizer, so one fixed text can be compared across tokenizers. Different texts have different entropy, so a lower value can mean easier text rather than a better model. The paper makes this point for perplexity across datasets (its Section 3.3) but does not state it for bits per byte.
- Benchmarks (PUBLISHED): HellaSwag (Zellers et al. 2019, arXiv 1905.07830) asks for the most plausible continuation, with wrong endings chosen adversarially. ARC (Clark et al. 2018, arXiv 1803.05457) has grade-school science questions; its Challenge subset keeps only items that two simple algorithms both got wrong. LAMBADA (Paperno et al. 2016, arXiv 1606.06031) asks for the last word of a passage, which needs the wider passage.
- Our metrics (FACT): validation bits per byte on the first 2,000,001 tokens of a held-out file (1,952 windows, standard error 0.002), every session; a held-out chain-two set in nats (6 million tokens); benchmarks at milestones. The chain-two validation folder's metadata had been copied from another folder, so its bits-per-byte figure was wrong. We found this and use nats for that set.
- Benchmarks at identified snapshots of the production chain, and the like-for-like comparison with public models, are in document 04.
- Our bits per byte (FACT, same validation file): 1.2359 after 51 million tokens (cooled, batch-43 row); 1.1149 after session 41, about 2.1 billion tokens, uncooled.
- Proxy and production use different vocabularies (4,096 and 32,768 entries) and different validation text (3.41 against 4.56 bytes per token), so their bits-per-byte values must not be compared (FACT).
- Evaluation pitfall (FACT): on a hosted notebook with two T4 cards and an older PyTorch build than the home machine's, compiled evaluation read the same model at 1.5950 bits per byte, against 1.5167 with eager evaluation, at the same training loss (3.145 against 3.142). That 0.078 gap came from the evaluation path alone. With eager evaluation, the control matched the home GPU machine within 0.0003.

## 8. Ours versus typical

| Aspect | Typical, in the published sources we read | Ours (FACT unless marked) | Status: proven in our tests / borrowed / untested |
|---|---|---|---|
| Optimizer for weight matrices | Adam or AdamW (GPT-3: Adam; LLaMA and Llama 3: AdamW) | Muon, learning rate 0.02 | Borrowed (Jordan write-up; Liu et al. 2025); no Muon-versus-AdamW test of our own in the notes read |
| Optimizer for the rest | AdamW | Adam, learning rate 0.008; embedding multiplier 20; beta2 0.99 | Borrowed rule; the values are ours and untested as a set |
| Warmup and learning-rate shape | Warmup, then cosine to 10 percent (GPT-3, LLaMA); warmup-stable-decay (MiniCPM) | 3 percent warmup in session 0, constant rate, planned 18-session square-root cooldown | Shape borrowed (Hagele et al.; MiniCPM); cooldown untested at this scale; one cooled session in an older chain gave 0.0154 bits per byte |
| Weight decay, matrices | 0.1 (GPT-3, LLaMA, Moonlight); 0.01 (Pythia) | 0.1, then 0.02, 0.006, 0.003 | Measured at proxy scale (flat between 0.0001 and 0.0003); production effect untested |
| Weight decay, embeddings and head | Not checked in the sources we read | None in the first trainer; separate head decay 0.01, then 0.003 | Untested |
| Gradient clipping | 1.0 (GPT-3, LLaMA) | Setting is 0; meaning not checked here | Untested |
| Batch size | 0.5 to 4 million tokens (GPT-3, LLaMA); 2,097,152 (Pythia) | 32,768 tokens | Ours; untested at larger batches |
| Context length | 1,024 (GPT-2); 2,048 (GPT-3, Pythia); 8,192 (Llama 3) | 1,024, with a short sequence ramp in session 0 | Context ours; ramp borrowed from the speed-run README |
| Tokens per parameter | About 20 (Chinchilla, arithmetic); about 1,850 (Pythia-160M) and 4,400 (SmolLM-135M), arithmetic | 63 planned; about 15 so far (arithmetic) | Ours; plan not finished |
| Repeated data | About one pass (Pythia); up to 4 epochs negligible (Muennighoff et al.); GPT-3 repeated some sources 1.9 to 3.4 times | About 3.2 epochs in plan A; about 69 percent of each session from the repeated pool in the current plan | Partly tested: one paired test, +0.025 nats per extra half pass |
| Run structure | One run per model in the papers we read | About 177 chained 24-minute sessions, with validation gates and 5-minute rests | Proven in smoke tests (continuity; kill and resume); full run in progress |
| Logit soft-capping | Gemma 2: 50 on attention, 30 on final logits | Tanh cap of 15 on the final logits | Borrowed; not tested separately in the notes read |
| Partial key offset | Not in the sources read (speed-run README only) | On | Borrowed; three equal-clock pairs at -0.0052, -0.0065, -0.0070 bits per byte; at 45 million, -0.0061 over four seeds |
| Value residual | Zhou et al. 2024 (cited by the speed-run README) | On, with a 0.5 initial setting | Borrowed; no separate test in the notes read |
| Value embeddings | Speed-run README only | On | Borrowed; no separate test in the notes read |
| Muon update scale | 0.2 times the square root of the larger dimension (Liu et al. 2025) | Not used, as far as the notes show | Not used |
| Evaluation | Bits per byte (Gao et al. 2020); HellaSwag, ARC, LAMBADA | Validation bits per byte on 2 million held-out tokens; nats for chain-two; benchmarks at milestones | Proven in our runs, with noise given; not comparable across proxy and production |

## What this does not show

- It does not compare our scores with other models; document 04 covers that.
- Most production comparisons use one seed. Seed noise is 0.0046 bits per byte per run at 45 million parameters (FACT); the proxy's comparison band is 0.0008 to 0.0015 (FACT).
- No production run is cooled yet, so the final loss of the main run is unknown.
- The effect of the decay steps on the cooled model is not measured at production scale.
- The key-offset gain shrank from about 0.08 bits per byte at 10 million tokens to about 0.006 at 45 million (FACT, with the caveat above); we do not know whether it holds at 9 billion tokens.
- The speed-run gain (Muon's 1.35 times) and Moonlight's 2 times (both PUBLISHED) come from other tasks and settings.
- Each published statement comes from the parts of the paper we could read; some appendices were not read.

## Open questions

- Does a decay of 0.003, or the proxy optimum near 0.0005, improve the final cooled model? Branch tests at milestones are planned.
- Is an 18-session cooldown too long or too short?
- Does the key-offset gain hold at 9 billion tokens?
- Do our repeated passes stay inside the 4-epoch band at this scale?
- Does Muon's advantage appear at a batch of 32,768 tokens?

## Basis

Our own logged runs and notes from 26 to 30 October 2026. Those notes are not published; this document is meant to stand on its own.

## Published sources used

All opened by us: Hoffmann et al. 2022 (arXiv 2203.15556); Kaplan et al. 2020 (arXiv 2001.08361, abstract only); Brown et al. 2020 (arXiv 2005.14165); Biderman et al. 2023 (arXiv 2304.01373) and the Pythia-160M model card (Hugging Face, EleutherAI); SmolLM-135M model card (Hugging Face, HuggingFaceTB); Touvron et al. 2023 (arXiv 2302.13971); Dubey et al. 2024 (arXiv 2407.21783); GPT-2 small configuration (Hugging Face, openai-community); Keller Jordan's Muon write-up (kellerjordan.github.io/posts/muon); Liu et al. 2025 (arXiv 2502.16982); modded-nanoGPT README (github.com/KellerJordan/modded-nanogpt); Zhou et al. 2024 (arXiv 2410.17897); Gemma Team 2024 (arXiv 2408.00118); Hu et al. 2024, MiniCPM (arXiv 2404.06395); Hagele et al. 2024 (arXiv 2405.18392); Loshchilov and Hutter 2017 (arXiv 1711.05101); D'Angelo et al. 2023 (arXiv 2310.04415); Wang and Aitchison 2024 (arXiv 2405.13698); Muennighoff et al. 2023 (arXiv 2305.16264); Gao et al. 2020, The Pile (arXiv 2101.00027); Zellers et al. 2019 (arXiv 1905.07830); Clark et al. 2018 (arXiv 1803.05457); Paperno et al. 2016 (arXiv 1606.06031).
