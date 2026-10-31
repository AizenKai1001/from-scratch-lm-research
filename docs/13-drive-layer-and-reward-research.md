# Drive layer and reward research

## Summary

Two questions, both asked of the same 143-million-parameter model family, after the reading experiments in document 08.

1. **Can a standing drive to learn be built into the reader, so that it keeps improving itself?** We tried four designs (a chooser that picks what to read, a standing inner-voice prefix, drive text in the reading stream, and a steering vector). Each was compared with a matched control, and each was killed or left inconclusive under a rule written before the run. The one design with a real effect (steering) moved the control measure too much to use.
2. **Which training rewards other than next-word prediction could help a 143-million-parameter model reason?** A literature review in two passes found that most evidence is at 1.5 billion parameters and above. The planned tests have written pass and kill rules. **None of them has been run.**

Labels: **FACT** means measured in our own logged runs, with the setting given. **PUBLISHED** means taken from a paper or web page we read, with its arXiv number or URL. Where we read only the abstract or a page summary, the text says so, and the paper should be checked before any number is used as a threshold. **GUESS** means our reasoning, not tested.

## 1. The drive tests

### 1.1 Setting and rules

- **The probe (FACT).** A frozen set of target prompts (where the learning continuation is the right one) and control prompts. The score is calibrated: the log-probability of the continuation minus its own unconditional log-probability, with the learning continuation compared with an avoidance continuation. For the unchanged model the target score is -0.23 nats (95 percent interval -0.88 to +0.41), so there is no clear lean towards learning before any training. The control prompts carry an offset of about +0.65 nats, which every layer test must subtract.
- **Matched controls (FACT).** Every arm is compared with a control arm: a random direction, matched control passages, or a neutral prefix. The effect of a layer is the change in the target score minus the change in the control score.
- **Rules, written before the runs (FACT).** PASS needs an effect of at least +0.50 nats, the lower end of its bootstrap interval above zero, and a canary change (drift on a fixed set of ordinary web windows) of at most +0.5 percent. KILL when the upper end of the interval is below +0.20 nats. Anything else is INCONCLUSIVE, and the layer is switched off rather than defended.

### 1.2 The four designs

| Design | What was tried | Result (FACT) | Verdict |
|---|---|---|---|
| A. Chooser that picks what to read by learning progress | Pick the next text by how much a small update improves a held-out probe | Noise trap: the raw loss picked random-id text in 500 of 500 draws. A progress proxy built from two checkpoints picked it 87 percent of the time. In a 60-step bandit, random-id text was pulled 60 of 60 times by the raw loss, 43 of 60 by progress and 14 of 60 by uniform choice. Held-out benefit after 60 updates: progress +0.005 and -0.029 nats, uniform +0.038 and +0.010 | Killed. The progress signal cannot tell noise from learning, so the chooser stays off |
| B. Standing inner voice as a prefix | A drive passage in front of each prompt, against a matched control passage (8 pairs) | Prefix: -0.134 nats (interval -0.270 to +0.002), 4 of 8 pairs positive. Explicit first-person statements placed between whole documents: +0.108 nats (interval -0.022 to +0.239), 5 of 8 positive. The same statements inside one document: -0.279 nats (interval -0.518 to -0.045) | Prefix killed. Between-document statements inconclusive, below the bar. Inside one document killed |
| C. Drive text in the reading stream | The drive passages read as ordinary text (300 segments of 128 tokens), against matched control passages | At 100 percent dose: -0.009 nats (interval -0.119 to +0.093). Passages placed between whole documents at 30 percent dose: -0.124 nats (interval -0.212 to -0.046). The other cells ran from -0.186 to +0.056 nats. One cell (+0.230 nats at the larger step) sat on a canary damaged by 1.8 to 2.2 percent in both arms, so it is read as damage or chance | Killed |
| D. Steering vector | A reversible vector (drive passages minus control passages) added at one layer | Layer 9, dose 0.20: +0.317 nats (interval +0.121 to +0.513) on one half of the prompts and +0.310 on the other half. Layer 3 was negative. Canary change: +0.007 percent at dose 0.05, +0.226 at 0.10, +1.39 at 0.20 (above the +0.5 limit), +7.6 at 0.40 | Inconclusive and not usable. The effect grows with dose, but the damage grows faster |

Common result (FACT): no design reached the +0.50 bar with its interval above zero. The steering design came closest, and only at a dose whose canary change was above the limit.

### 1.3 What we think this means (GUESS)

- The drive passages are children's-book narrative about learning, not an explicit statement of a standing aim. A base model may follow a self-description much more readily than a story, which would fit the between-document result for explicit statements (+0.108 nats, not yet distinguishable from zero).
- Placement matters for damage, not for effect. Cutting documents to fit fixed slots raised the canary by about +0.3 percent at dose 30; placing passages between whole documents brought it to about zero (+0.00 to +0.02). The effect on the probe did not appear either way.
- A standing drive might need to be in the training data from the start, not added to a finished model. This is untested (see 1.4).

### 1.4 The untested route: an early implant

The one route not yet tested is to put a small, steady share of drive text into training while the model is still early in its run. The text available after strict filtering from public-domain books is small: about 8,700 passages and about 1.1 million tokens (FACT, from a filter run). At a share large enough to matter, the same passages would repeat many times and would probably be memorised rather than learned (GUESS from repeat counts), so the share would have to be about 0.1 percent or less in a production-sized run. The planned test uses three small models trained for about 25 minutes each: no share, a drive share, and a matched control share. **Status: OPEN. No rules file has been written.**

## 2. Reward and objective research

Two literature passes were made (October 31, 2026), using web search and page reading. The question was which training rewards or objectives beyond next-word prediction could give a 143M model more reasoning, what each one needs, and what is realistic on one consumer graphics card in a few hours. Read levels are given because several numbers come from page summaries.

### 2.1 What the evidence says

- **Verifiable rewards (a program checks the answer).** This is the most direct route to checkable reasoning, and the evidence sits at 1.5 billion parameters and above. The exceptions are partial. A 0.8 billion parameter search-agent run used reinforcement learning with an exact-match reward and reached an average exact match of 0.352 against 0.092 untrained (arXiv 2609.28765, abstract and summary; a question-answering set with a search tool, not arithmetic). A 0.5 billion model had much math in its pretraining (arXiv 2601.20312, summary). Reinforcement learning on 1 and 4 billion pretraining checkpoints raised GSM8K pass@1 from about 2 to about 18 percent at a 4-billion-token checkpoint (arXiv 2606.04272, "RL Excursions during Pre-Training", summary; no model under 1 billion was tested). TinyZero's README says its 0.5 billion base failed to learn reasoning (PUBLISHED, README read through a mirror, github.com/Jiayi-Pan/TinyZero). Students of 3 billion parameters or fewer did worse with long reasoning traces than with short ones (arXiv 2502.12143, first part of the full text). **No source we found shows a model from scratch at 143M, or below 500M, gaining reasoning from reinforcement learning with a benchmark number.** A 143M model is therefore a long shot (GUESS).
- **Latent reasoning (the model passes hidden states instead of written steps).** It works at GPT-2 scale only with written worked solutions as a teacher and a curriculum. Coconut reached 34.1 percent on GSM8k against 42.9 for written chain-of-thought, and 14.4 without its curriculum (arXiv 2412.06769, tables read). CODI reached 43.7 percent on GSM8k-Aug, about 99 percent of explicit chain-of-thought accuracy (arXiv 2502.21074, abstract and HTML). The GSM8k-Aug data may be machine-written, so its provenance must be checked before use.
- **Step rewards (scoring each intermediate step).** Process supervision beat outcome supervision on MATH, with about 800,000 human step labels (arXiv 2305.20050, abstract). Automatic step labels were tested at 7 to 70 billion parameters (arXiv 2312.08935, Math-Shepherd, first part of the full text). A countdown-style task lets a program check each step for free (our idea; its value is GUESS).
- **Multi-token prediction.** It gained for code at 13 billion parameters. The authors describe it as worse than the baseline at small sizes and better at scale, and in a synthetic test its advantage ended at about 100 million parameters (arXiv 2404.19737, first part of the full text). We expect no gain at 143M (GUESS).
- **Intrinsic rewards beyond learning progress.** We found no language-model evidence at any size. A curiosity bonus added to verifiable reinforcement learning reported about +3 points on AIME (arXiv 2509.09675, abstract; model sizes not found). A 2026 theory paper says learning-progress rewards cannot be made unbiased in general (arXiv 2606.19476, search summary only). That fits our chooser result in 1.2 (A).
- **A thought before each token, scored without a checker.** The RLP method (arXiv 2510.01265, ICLR 2026 poster; first part of the full text) rewards a short private thought by how much it improves the prediction of the real next token. It needs no checker and uses all ordinary text, but it was tested from 1.7 billion parameters upwards on existing checkpoints, not from scratch. Its fit at 143M is unknown (GUESS).
- **Self-play with a learnable-task reward.** Absolute Zero (arXiv 2505.03335, section 3.1 of the full text) tested models of 3 billion parameters and up.

### 2.2 Planned tests (OPEN: none has been run)

All arms share one budget and, unless stated, two seeds. The rules were written before any run and are fixed here. A held-out split is used only for the pass or kill decision, never for tuning.

| # | Test | Arms | PASS | KILL |
|---|---|---|---|---|
| 1 | Base behaviour probe | On 200 countdown-style prompts, count base samples that show a check or a retry | At least 10 percent of samples show one | Below 10 percent: the reinforcement test waits until a small primer, written by a program rather than by a model, is added |
| 2 | Verifiable reinforcement learning on generated countdown puzzles (four numbers, a target from 10 to 99, each number used once, a Python checker) | Floor gate first: pass@64 on 500 held-out puzzles must be at least 5 percent, or the test is "data too hard". Arms: the true reward, and the reward shuffled across samples as a control | pass@1 rises by at least 10 points; the true reward beats the shuffled one by at least 5 points on both seeds; canary rises at most 2 percent | pass@1 rises less than 3 points; or the true reward is within 3 points of the shuffled one on either seed; or the canary rises more than 5 percent |
| 3 | Reward shape and adaptive difficulty (added to test 2) | (a) exact match only; (b) exact match plus 0.1 for a valid expression that uses each given number at most once; (c) 0.1 for a valid expression with no target bonus; (d) start at three numbers and move to four when group accuracy reaches 0.5 | (b) beats (a) by at least 3 points on both seeds; (d) beats the fixed four-number arm by at least 3 points on both seeds, with no reward collapse | (b) is below (a) on either seed, or (c) matches (b); or (d) is at or below the fixed arm, or reward stays at zero for an extended stretch |
| 4 | Step rewards (only after test 2 passes its floor) | Outcome reward; outcome plus 0.1 per valid intermediate step; step bonus only (control for farming) | Step arm beats outcome-only by at least 3 points on both seeds, and the bonus-only arm does not beat it | Step arm below outcome-only on either seed, or bonus-only matches the step arm |
| 5 | Latent reasoning on human-written grade-school maths solutions | Floor gate: explicit chain-of-thought must reach at least 3 percent on the test set. Arms: explicit, answer only, latent, latent with shuffled targets. The data licence is checked first | Latent beats answer-only by at least 3 points on both seeds, and the shuffled-target arm is at least 3 points below latent | Latent at or below answer-only, or the shuffled-target arm matches latent |
| 6 | Multi-token auxiliary head | About 30 million parameters, 50 million tokens. Next token only against next token plus a 2-token head with weight 0.5 | Multi-token arm lower by at least 0.02 nats on both seeds, canary no worse | Equal or higher on either seed (the outcome we expect) |
| 7 | Thought-before-each-token proxy (RLP style) | About 30 million parameters, 10 million tokens, eight rollouts, thought length 64 to 128. Arms: plain next-token training with the same forward passes, and the RLP reward | Lower held-out loss by at least 0.02 nats on both seeds at matched forward passes, canary no worse | Equal or higher on either seed, reward near zero, or canary worse by more than 2 percent |
| 8 | Transfer-progress chooser | Chooser bandit with its score taken on text the update did not see; three seeds | Share of noise picks at most 35 percent (chance is 50), and gain at least the uniform gain on both benches | Share of noise picks above 50 percent, or gain below uniform on both benches |

The early-implant drive test in 1.4 has no rule yet and is not in this table.

## 3. What this does not show

- Most of the reward evidence comes from abstracts and page summaries. Several numbers may change on reading the full papers, so none of them should be used as a threshold before that check.
- The drive tests used one seed, between 8 and about 300 pairs, and one probe. Small effects near the bar cannot be separated from noise at that size.
- Whether any of these ideas works at 143M is not known. The evidence reviewed points to larger models, and the planned tests exist to find out cheaply.
