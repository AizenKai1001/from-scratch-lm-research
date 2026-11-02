# From-scratch language model research

Research notes, tests, results and open questions from training a 143-million-parameter language model from scratch on two consumer graphics cards, and from experiments on a model that keeps learning while it reads.

**This repository contains documentation only: no code, no data and no model weights.** Status of the numbers: 2 November 2026. The work is still running, so everything here is a snapshot, and each document says what a number was measured on.

## What the project is

- **A base model trained from scratch.** 142.9 million parameters, 12 transformer blocks, a 1,024-token context and a 32,768-token vocabulary. It is trained in chained sessions of about 25 minutes on one home graphics card, so a bad session costs minutes, not days. By the evening of 30 October it had finished 42 sessions and seen about 2.1 billion tokens of a planned 9 billion. Its planned final cooldown has not run yet, so all its numbers are for an unfinished model.
- **A "reader" built on a copy of that model.** After reading each short stretch of text it takes one small learning step, it can go back over the stretches that surprised it, and it is checked with quizzes instead of being corrected by them. The aim is a model that goes on learning from whatever it reads. The base model exists so that this reader's brain is our own and can be measured.
- **A record of how we tested things.** For the larger decisions the rules for pass and fail were written before the data existed (a few were changed later, and each change is listed), controls were run first, a second (AI) reviewer was asked to find mistakes, and the claims we had to take back are listed.

## The questions this repository answers

| Question | Where |
|---|---|
| What is the idea and the theory behind it? | [01 Overview and theory](docs/01-overview-and-theory.md) |
| How does this differ from normal AI training? | [02 Differences from standard training](docs/02-how-this-differs-from-standard-ai-training.md) |
| What exactly is the model and the recipe? | [03 Model and recipe](docs/03-the-model-and-training-recipe.md) |
| How does our model compare with other models? | [04 Comparison with other models](docs/04-comparison-with-other-models.md) |
| What has been tested, and what is proven? | The table below and documents 05 to 10 |
| What is still open? | [11 Open questions](docs/11-open-questions.md) |
| How much should you trust any of it? | [12 Method and honesty log](docs/12-method-and-honesty-log.md) |
| Can the reader be given a drive, and which rewards beyond next-word prediction are worth testing? | [13 Drive layer and reward research](docs/13-drive-layer-and-reward-research.md) |
| Can the model itself compress text, and what does its decoder cost? | [14 Compression with the model](docs/14-compression-with-the-model.md) |
| Which small variants (ternary, spiking, looped, efficiency tricks) held up, and what did Countdown puzzle training do? | [15 Proxy screens and the Countdown lift](docs/15-proxy-screens-and-the-countdown-lift.md) |
| Where is this going, and which parts of the end goal (always learning, repairing itself, hidden thinking, small enough to run anywhere) exist yet? | [16 Vision and roadmap](docs/16-vision-and-roadmap.md) |
| Which next ideas look worth testing, and what could each do for us? | [17 Candidate ideas and what they could do](docs/17-candidate-ideas-and-what-they-could-do.md) |
| Can the model learn new skills in a row without forgetting old ones (repair or rollback, and what works)? | [18 Learning new skills without forgetting](docs/18-learning-new-skills-without-forgetting.md) |
| Have the senses (eyes, ears) been built from scratch, and can the language model use them? | [19 First senses, built from scratch](docs/19-first-senses-built-from-scratch.md) |

## Results at a glance

Status words: **measured** means a logged run with its setting stated; **predicted first** means the number was written down before the measurement; **one run** means no repeat; **guess** means reasoning that was not tested.

What held up against a control or against a prediction that could have failed: the jump in weight-matrix size after the decay cut, the cloud-GPU bias caught by a control run, the reading gain with its foreign-book control, and replay against drift. Everything else rests on one run per setting, on small effects or on guesses, as the status column says.

| Result | Setting | Status | Document |
|---|---|---|---|
| Chained 25-minute sessions work: each session resumes from the last checkpoint and must pass a gate (a few failures were found and fixed along the way). Validation fell from 1.366 bits per byte at session 0 to 1.115 at session 41 (uncooled). | 143M model, 2.1 billion tokens, one run | measured | [03](docs/03-the-model-and-training-recipe.md) |
| Cutting weight decay five-fold doubled the size of the hidden weight matrices within one session, and the first two session-to-session ratios matched predictions written beforehand. The weights then kept growing past the predicted final level (2.33 times against 2.24 after four sessions, still rising); our first claim that they had settled was wrong. | Two training chains | predicted first, measured | [05](docs/05-weight-decay-and-training-dynamics.md) |
| That cut lowered the validation loss by 0.0375 bits per byte in one session. A skeptic check found most of it is an uncooled effect (a partial learning-rate cooldown, we guess); after cooling, only about 0.005 remained, which is noise-sized. | One cooled session, one seed | measured, cause is a guess | [05](docs/05-weight-decay-and-training-dynamics.md) |
| On a 4.5M-parameter proxy the best weight decay is flat between production-equivalent 0.0001 and 0.0003, far below the 0.003 the main run uses now. | Proxy, 281M tokens, two seeds | measured | [05](docs/05-weight-decay-and-training-dynamics.md), [09](docs/09-small-scale-proxy-tests-and-the-curriculum-test.md) |
| An evaluation setting on rented cloud GPUs was biased by 0.078 bits per byte; a control run caught it before any conclusion was drawn. | Proxy, same seed on two machines | measured | [09](docs/09-small-scale-proxy-tests-and-the-curriculum-test.md) |
| "Easy text first" (children's stories, then standard text) made the final model worse by 0.022 bits per byte; ending on easy text made it much worse (+0.365). | Proxy at production-equivalent decay 0.0003, two seeds per arm, pre-written rule (arm E outside it) | measured, small scale | [09](docs/09-small-scale-proxy-tests-and-the-curriculum-test.md) |
| At about 0.6 billion tokens, a broad, repeated pool of data beat a mostly educational web mix on 8 of 9 held-out sets by 0.111 nats per token, and lost by 0.068 on the educational set. The main run switched on 29 October to a 70/30 mix of the two, for robustness; a pre-written rule applied on 30 October to pooled pairs at 1.05 to 1.20 billion tokens (where the breadth gap had shrunk to 0.043) said to keep it. | One run per plan | measured, one run each | [06](docs/06-data-plans-and-the-equal-token-comparison.md) |
| Taking one small learning step after each 128 tokens of a book lowered the loss on the rest of the book by 2.4 to 4.6 percent, in every tenth of every book. Steps taken from a different book made the same book worse (-1.27 percent). | Two books of about 120,000 tokens, one run each | measured, with a control | [08](docs/08-learning-while-reading.md) |
| Mixing in a small share of real training text ("replay") cut the drift of the loss on ordinary web text from +1.39 to +0.36 percent while keeping the whole gain. | One book | measured | [08](docs/08-learning-while-reading.md) |
| The gain depends on the kind of text: 13.9 percent on code, 9.9 on science papers, 3.4 to 3.9 on fiction and math pages, 0.8 to 0.9 on ordinary web pages. | Eight held-out streams of 80,000 tokens | measured, one run each | [08](docs/08-learning-while-reading.md) |
| Going back over the most surprising 10 percent of the text made the weights keep exactly those stretches better, but did not help prediction of later text. | One book | measured | [08](docs/08-learning-while-reading.md) |
| Two quizzes to test whether the reader learned the facts of a book. The first was too easy (81 to 90 percent right before reading) for accuracy to move. The second, harder one (about 43 percent right before reading) showed a small extra gain in confidence on the part of the book that had been read (+0.4 nats, interval above zero) but no clear gain in accuracy. | Two books, about 75 questions per half | measured, inconclusive by the rule written beforehand | [08](docs/08-learning-while-reading.md) |
| A dictionary entry for a rare word placed in front of a sentence makes that word about 7 nats easier to predict (a copy effect), but learning with it was worse than learning with another word's entry, in all 50 words: the look-up as a learning action failed its pre-written rule. | 50 rare words, paired, one snapshot | measured; killed by the rule written beforehand | [08](docs/08-learning-while-reading.md) |
| A previous-token key shift makes induction heads appear by 3 million tokens; many other readings from 20 interpretability studies were narrowed or withdrawn on re-check. | Small CPU studies, mostly one seed | measured, with reviews | [07](docs/07-interpretability-findings.md) |
| Our unfinished 143M model (session 36, 1.85 billion training tokens) is level with GPT-2 small and Pythia-160M on HellaSwag and the two ARC tests, behind them on LAMBADA and by about 0.1 bits per byte on held-out web text of the kind it trained on, above Pythia-70M, and far below SmolLM-135M. It used 160 to 320 times fewer training tokens than Pythia and SmolLM. At about 2 billion tokens it is 0.30 bits per byte ahead of Pythia's own step-1000 checkpoints, which are unfinished and trained on different data with a 64-times larger batch. | One script for every model; 234 held-out documents | measured; not a ranking of recipes, because there is no same-data standard baseline | [04](docs/04-comparison-with-other-models.md) |
| Reading with replay, as one loop: stable over three bursts on validation text (zero rollbacks), and over 22 bursts (about 130,000 segments, one pass) on a private collection of web novels and song lyrics. Replay alone moved the canary more than the loop did; reading's gain showed only on the text just read. On the parts of each book never read, the gains were +4.1 to +17.4 percent (the lyrics -0.9 percent) | One 143M copy; one seed; the replay-only control for the long run was still running | measured; the long-run gains are a hint, not a result | [08](docs/08-learning-while-reading.md) |
| Four designs for an implanted drive (a chooser, a standing inner-voice prefix, drive text in the reading stream, a steering vector) were killed or inconclusive. The best one, steering, gave +0.32 nats but moved the canary by 1.4 percent, above its limit | Matched controls in every arm; rules written first | measured; killed or inconclusive by the rules | [13](docs/13-drive-layer-and-reward-research.md) |
| Verifiable-reward reinforcement learning and other reward ideas: most evidence is at 1.5 billion parameters and above. The planned tests at 143M have written pass and kill rules | Literature review in two passes | published evidence; tests open, none run | [13](docs/13-drive-layer-and-reward-research.md) |
| The model as a lossless compressor: 2.29 MB of held-out text coded to 314,132 bytes, against 692,138 (bzip2) and 696,600 (xz), with an exact decode check. The model's 286 MB decoder is not counted | 143M session-92 checkpoint, one run, precision 2^20 | measured, one run | [14](docs/14-compression-with-the-model.md) |
| Rounding the weights after training: 6 bits (108 MB) costs about 0.019 bits per byte; 4 bits and below collapse without retraining | Post-training rounding, 923 KB held-out text | measured, one run | [14](docs/14-compression-with-the-model.md) |
| Ternary weights at 24 million tokens: all three seeds lower than normal weights, mean -0.010 (the rule's bar is -0.005). At 12 million tokens they were level or slightly worse | Proxy about 21.5M parameters, three seeds | measured; the rule reads it as slightly better | [15](docs/15-proxy-screens-and-the-countdown-lift.md) |
| An extra next-token prediction head is lower on all four seeds (mean -0.047), but takes about 1.5 times as long per run. The other six efficiency tricks did not clear the written rules | 12M tokens, two kinds of hardware, seeds 1, 3, 5, 7 | measured; fails the speed condition | [15](docs/15-proxy-screens-and-the-countdown-lift.md) |
| A copy fine-tuned on programmatic Countdown puzzles solved 48.2 percent in 64 tries (base 0 of 500), but normal-text loss rose from 1.0817 to 1.1403; mixing 20 percent ordinary text kept it at 1.0773 with 41 of 100 puzzles | Copy of the base, 800 steps, held-out number sets excluded | measured; same format as training, not general reasoning | [15](docs/15-proxy-screens-and-the-countdown-lift.md) |
| Adding a new skill (100 memorised facts) to the model: unprotected training learned it (100%) but damaged old skills (text loss 3.54 to 4.06-4.27); rollback kept old skills but never learned it (0%); learn-then-heal (120 learning steps, then 80 healing steps that are mostly rehearsal) learned it (99-100%) and kept text loss within 0.05 of rollback. On 320 Countdown puzzles it matched rollback (12.8% against 12.3%, two seeds pooled) | 143M model, seeds 777, 778, 779; re-score of 640 attempts | measured; the new skill is memorisation | [18](docs/18-learning-new-skills-without-forgetting.md) |
| Many new skills in a row: with about 90% of the healing batch rehearsing earlier facts, 6 skills were kept at 88-94% recall on three seeds and 12 skills at 85% (weakest set) on two seeds; with 25% rehearsal the weakest set fell to 40%; unprotected training lost the earliest skills almost completely. Halving both learning rates cut the drift of text loss in the 12-skill chain from +0.33 to +0.13 on one seed (and on a second seed), without hurting learning; a quarter of the rate under-learned | 143M model, 1 to 3 seeds per arm, rules written first | measured; indicative | [18](docs/18-learning-new-skills-without-forgetting.md) |
| A 2.24M-parameter image model built from scratch reached 90.4% and 90.1% on CIFAR-10 test pictures (two seeds, about 10 minutes each). A small bridge let the frozen language model name the pictured class 90.5% of the time, with its text behaviour exactly unchanged | 143M language model, frozen; one bridge seed | measured; first sense, narrow test | [19](docs/19-first-senses-built-from-scratch.md) |

## What is not shown

- That the finished model will be good: the planned cooldown and the remaining 7 billion tokens are not done.
- That learning while reading is a safe, lasting improvement: it was measured over books of 120,000 tokens, with a slow drift on other text. Weeks of continuous reading are untested.
- That a lower loss means the model has learned what the text says (the first quiz could not tell).
- That anything here holds at larger model sizes. Most tests are one run on a 4.5M or 143M model.
- That learning without forgetting works for skills that need reasoning, or at larger sizes: the new skills in document 18 were memorised facts on one model size.
- Independent verification: the experiments and the writing were done with an AI assistant and checked by further AI reviews. Nothing has been peer reviewed.

## The documents

| | Document | What is in it |
|---|---|---|
| 01 | [Overview and theory](docs/01-overview-and-theory.md) | The two layers of the project, the ideas behind each test and what each idea predicts |
| 02 | [How this differs from standard AI training](docs/02-how-this-differs-from-standard-ai-training.md) | Our recipe against the published recipes of GPT-3, LLaMA, Pythia and others: optimizer, data per parameter, weight decay, session design |
| 03 | [The model and the training recipe](docs/03-the-model-and-training-recipe.md) | Architecture, optimizer, the chained-session design, every session's validation value, what failed and was fixed |
| 04 | [Comparison with other models](docs/04-comparison-with-other-models.md) | Benchmarks and bits per byte against GPT-2, Pythia and SmolLM, with published numbers and every caveat |
| 05 | [Weight decay and training dynamics](docs/05-weight-decay-and-training-dynamics.md) | The decay schedule, the jump in weight size, the loss drop and its skeptical reading, the output-head fix |
| 06 | [Data plans and the equal-token comparison](docs/06-data-plans-and-the-equal-token-comparison.md) | Three data plans, the rules written before the data, the pooled check, what was given up |
| 07 | [Interpretability findings](docs/07-interpretability-findings.md) | What 20 studies of the model's inside showed, and the claims taken back |
| 08 | [Learning while reading](docs/08-learning-while-reading.md) | The reader experiments: step size, books, safeguards, retention, revisits, eight kinds of data, the quiz |
| 09 | [Small-scale proxy tests and the curriculum test](docs/09-small-scale-proxy-tests-and-the-curriculum-test.md) | The 4.5M-parameter proxy, the hardware control, the weight-decay rounds, easy-to-hard ordering |
| 10 | [Adaptation and biology notes](docs/10-adaptation-and-biology-notes.md) | Ideas taken from brains, animals and immune systems, which were tested, and 24 corrections |
| 11 | [Open questions](docs/11-open-questions.md) | What we do not know, what would settle it and roughly what it costs |
| 12 | [Method and honesty log](docs/12-method-and-honesty-log.md) | How the experiments are run, the claims we had to correct, tooling errors, predictions against outcomes, rules adopted on 31 October |
| 13 | [Drive layer and reward research](docs/13-drive-layer-and-reward-research.md) | The four implanted-drive designs and their numbers, and the literature on rewards beyond next-word prediction with the planned tests and their rules |
| 14 | [Compression with the model](docs/14-compression-with-the-model.md) | The language model as a lossless compressor: measured sizes and speeds, the decoder-size problem, break-even estimates and limits |
| 15 | [Proxy screens and the Countdown lift](docs/15-proxy-screens-and-the-countdown-lift.md) | Ternary, spiking and looped variants, the efficiency-trick screen on two kinds of hardware, and the Countdown puzzle fine-tune with its replay check |
| 16 | [Vision and roadmap](docs/16-vision-and-roadmap.md) | The end goal (an always-learning, self-repairing, small model), what exists and what is only a plan, the hard problems, and the order of the next tests |
| 17 | [Candidate ideas and what they could do](docs/17-candidate-ideas-and-what-they-could-do.md) | Compression, learning-without-forgetting, model and safety ideas from a broad review, each with what it could give, its cost and its first test; all untested guesses unless a row points to a measured document |
| 18 | [Learning new skills without forgetting](docs/18-learning-new-skills-without-forgetting.md) | Repair, rollback and learn-then-heal on a real model: one new skill, three, six and twelve in a row, the healing batch mix, the learning-rate halving and its limits |
| 19 | [First senses, built from scratch](docs/19-first-senses-built-from-scratch.md) | A tiny image model built from scratch, the bridge that lets the frozen language model read what it sees, the ears in progress and the open design questions |

## How to read the labels

Most claims carry one of three labels. **FACT** is measured in our own logged runs, with its setting (model size, tokens, seeds, windows). **PUBLISHED** is taken from a paper or page we opened (or, where a document says so, a second-hand citation), named in the text. **GUESS** is our reasoning, which was not tested. Documents also use DERIVED or (arithmetic) for calculations on stated inputs, PLAN for intentions, and UNVERIFIED, NOT ESTABLISHED, OVERSTATED, CORRECTED and WITHDRAWN for claims that did not fully hold up. Where a document says "skeptic review", it means a separate AI pass that recomputed numbers from raw outputs and looked for mistakes.

## About this repository

- The research is run by one person with an AI assistant (Claude, made by Anthropic). The person sets the goals and makes the decisions; the assistant ran the experiments and drafted the text; reviews were also AI passes.
- The documents refer to "our notes". Those lab notes are not published. Each document is meant to stand on its own.
- No licence has been chosen yet, so by default all rights are reserved by the author. You are welcome to read and link to the documents; please ask before reusing the text.
- If you spot a mistake, an issue on this repository is welcome.
