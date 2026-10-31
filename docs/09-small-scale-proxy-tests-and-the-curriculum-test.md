# Small-scale proxy tests and the curriculum test

## Summary

Before and alongside the production run, the project tested choices on a small proxy model of 4.46 million parameters, 281.28 million tokens and 8,584 steps, matched to the production run at 63 tokens per parameter and on its weight-decay scale. A check on free cloud GPUs caught a bias in one evaluation path that the home card did not show, and after a fix the cloud runs matched the home runs. The weight-decay rounds found that a much lower decay than the 0.02 the main run used from session 12 gave a better final loss at this scale, and that between 0.0001 and 0.0003 the difference was within noise. In the curriculum test, plain standard text beat arms B and D by 0.0222 and 0.0486 bits per byte on two seeds, and arm E, which ends on easy text, was far worse; easy text placed first beat the same text shuffled in by 0.0264, but it used up standard-text tokens. This is a small-scale result on a model about 3 percent the size of the production model, and it may not transfer to larger models.

## Scope, labels and setting

- Basis: our own design notes, run records and dataset notes from 29 and 30 October 2026. Those notes are not published; this document is meant to stand on its own.
- Labels: FACT means measured in our own runs, with the setting given. PUBLISHED means taken from a source we opened, named where used. GUESS means reasoning that was not tested. Arithmetic on measured values is marked FACT, with its inputs.
- Home card: the older GPU in the home GPU machine, in fp16 arithmetic. Cloud GPUs: free cloud notebook GPUs, one proxy run per GPU, with a different PyTorch version (2.11 against 2.14 at home).

## Why proxies

- A production session is a 24-minute slot, and the production run is planned for about 9 billion tokens, roughly three days of slots (FACT, production design and run timing).
- One proxy run took 14.2 to 15.2 minutes on the home card and about 21 to 23 minutes on the cloud GPUs (FACT). The eight curriculum runs finished in under two hours of wall-clock time on two cloud GPUs (FACT, round timestamps), and the cloud runs did not take the production card.
- Several seeds per setting were needed because of seed noise. A registry estimate at about 45 million tokens put the run-to-run spread at 0.0046 bits per byte, from three to four runs per setting (FACT, rough). One run cannot separate effects of that size (GUESS: reasoning).

## Matching the proxy to the production run

| | Production | Proxy |
|---|---|---|
| Parameters | 142.87 million | 4.46 million (about 3 percent) |
| Width, layers, heads | 640, 12, 5 | 160, 6, 5 |
| Vocabulary | 32,768 | 4,096 (the first token ids of the same tokenizer) |
| Tokens | about 9 billion | 281.28 million (8,584 steps of 32,768 tokens) |
| Tokens per parameter | 63.0 | 63.0 |
| Share of parameters in the block matrices the decay acts on | 41.3 percent | 41.3 percent |
| Batch and context | 32,768 tokens; 1,024 | same |
| Learning rates | 0.02 and 0.008; embeddings times 20 | same, untuned at this size |
| Warm-up, stable phase, cooldown | 46 steps; constant rate; cooldown over the last 18 of 177 sessions | 46 steps; constant to 90 percent of steps; cooldown over the last 10 percent |

Labels: the counts and settings are FACT (from the design and the trainer). The tokens per parameter are FACT arithmetic.

- The vocabulary was cut to the first 4,096 token ids of the production tokenizer. Checked against the full tokenizer library on 2,500 documents, the cut had zero mismatches (FACT).
- Decay units: the proxy decay is the production-equivalent decay multiplied by 274,658 and divided by 8,584, which is about 32 (FACT arithmetic; 274,658 is 9 billion tokens divided by 32,768 tokens per step). The aim is to keep the total decay over the run the same in both, so the proxy's decay per step is about 32 times larger (GUESS: the matching principle; whether it carries over to production scale is not tested).
- Known differences (FACT): the proxy has about 32 times fewer steps; its validation text has 3.41 bytes per token against 4.56 for the production validation text; width, depth and vocabulary differ.

## The hardware control that caught a biased evaluation

- Before the cloud rounds, a control was set. The same proxy setting run on a cloud GPU had to land within 1.4040 to 1.4120 bits per byte, a window taken from the home card's three seeds (FACT, set before the run).
- The first round failed: the control landed at 1.4951 (FACT). The cloud environment used a different PyTorch version (FACT).
- A diagnostic on 1,500 steps with the same seed showed training losses close to the home run (3.564 against 3.550 at step 500). Validation loss from the compiled evaluation path was 0.078 bits per byte higher than in eager mode (1.5950 against 1.5167) at the same training loss (FACT).
- The fix was to evaluate in eager mode, with the training code unchanged. The control then landed at 1.4082, against 1.4085 at home with the same seed, inside the window (FACT).
- The first round's results were declared invalid for any decay conclusion (FACT). The lesson recorded: run a cross-environment control before any experiment (FACT, the lesson as written in the notes).

## Weight-decay rounds

Final cooled validation loss on the home card, in bits per byte, at each production-equivalent decay (FACT; proxy setting; seeds as shown):

| Decay (production scale) | Loss | Seeds |
|---|---|---|
| 0.1 (the value before session 12) | 1.5917 | 1 |
| 0.02 (the main run from session 12) | 1.4986 | 3 (1.4992, 1.4977, 1.4988) |
| 0.008 | 1.4519 | 1 |
| 0.003 | 1.4256 | 1 |
| 0.001 | 1.4128 | 1 |
| 0.0003 | 1.4076 | 3 (1.4085, 1.4051, 1.4093) |

- The control and the lowest rung had three seeds; the other rungs had one (FACT).
- A second stage confirmed the lowest rung against the 0.02 control on two fresh seeds: the control was worse by 0.0926 and 0.0895 bits per byte, mean 0.0910 (FACT).
- The pre-registered rule said: go lower, to 0.0003, with the optimum not bracketed (FACT). A quadratic fit placed the minimum near 0.00054; this fit is descriptive only (FACT).
- The cloud replicate, after the fix, found 0.0001 and 0.0003 flat. Seed 1 gave 1.4080 against 1.4082; seed 2 gave 1.4041 against 1.4050. The paired differences were -0.0002 and -0.0009, mean -0.0006, inside a noise band of 0.0008 to 0.0015 (FACT). There was no reason to go below 0.0003 (FACT).
- The proxy sets a landing value only. The production decay was lowered in stages afterwards, with checks at each stage (see document 05) (FACT).
- Count note: the results file held 10 valid rows when it was read on 30 October at 2:24 AM, while the verdict record of 1:03 AM says 12 valid runs. The two counts are not reconciled (FACT).

## The curriculum test

Question: does the order of the text, or adding easy text, change the final loss on standard text at this scale? (FACT, design note.)

The design was written on 30 October at 2:24 AM, before any run (FACT):

- Standard text is read in random document order. The standard trainer draws random windows from the whole file, so file order would have been ignored; the screen used a copy that reads in sequence, and it was smoke-tested on the cloud GPUs before the rounds (FACT).
- Each arm reads 281.28 million tokens, 8,584 steps (FACT).
- The primary metric is the final cooled loss on the standard validation set. The tolerance is the larger of 0.004 and twice the spread of the two seeds of arm A (FACT).
- Outcome 1, curriculum helps: ordering beats shuffling and the curriculum beats plain text, each by at least the tolerance. Outcome 2, only extra easy text helps: easy text beats plain text but ordering does not. Outcome 3, no effect or harm: anything else. HARM is marked when plain text beats an easy arm by at least the tolerance. A third seed is run only if a two-seed mean sits between half the tolerance and the full tolerance, or if the two seeds disagree in sign on a gain whose mean is at least the tolerance (FACT).

| Arm | Order of text | Tokens | Easy share |
|---|---|---|---|
| A | standard text in random document order | 281.28 million | none |
| B | easy text first: TinyStories 56.26 million, children's books 56.26 million, OneStopEnglish articles at three reading levels 0.67 million; then standard text, 168.1 million | 281.28 million | 40.2 percent, first |
| D | the same easy and standard units shuffled into one random order | 281.28 million | 40.2 percent, mixed |
| E | standard text, 168.1 million, first; then the easy units last | 281.28 million | 40.2 percent, last |

Labels: FACT (design and data build). Three points about the arms:

- Arm C, standard text sorted by reading level, was designed but not run.
- Arm E was added after the design note was written. The rule is defined on arms A, B and D (arm C was never run), so E is reported descriptively and sits outside the decision (FACT).
- The screen was first planned for the home card, which would have meant pausing the comparison run from document 06. It ran on free cloud GPUs instead (FACT).

Results, bits per byte on the standard validation set, final cooled, two seeds per arm (FACT). The second-text column is held-out text of the broad pool, the production mixture.

| Arm | Seed 1 | Seed 2 | Mean | Second text, mean |
|---|---|---|---|---|
| A | 1.4079 | 1.4073 | 1.4076 | 1.4581 |
| B | 1.4308 | 1.4288 | 1.4298 | 1.4791 |
| D | 1.4568 | 1.4556 | 1.4562 | 1.4991 |
| E | 1.7729 | 1.7721 | 1.7725 | 1.9215 |

The seed spread of arm A is 0.0006, so the tolerance is 0.004 (FACT arithmetic).

Gains, positive when the first named arm is better, mean of two seeds (FACT):

| Comparison | Gain | Seeds |
|---|---|---|
| B over D (order) | +0.0264 | both positive (+0.0260, +0.0268) |
| B over A (curriculum against plain) | -0.0222 | both negative (-0.0229, -0.0215) |
| D over A (extra easy text) | -0.0486 | both negative (-0.0489, -0.0483) |
| A over B (plain beats B) | +0.0222, above the tolerance, so HARM is marked | |

Verdict: OUTCOME 3, no effect or harm, with HARM marked. The order condition held: easy text first beat the same easy text shuffled, on both seeds. The curriculum condition and the extra-easy-text condition did not hold: plain text beat both easy-text arms by more than the tolerance. No third seed was needed (FACT).

Other observations (FACT):

- Easy text took 40 percent of the budget, so arms B, D and E saw 168.1 million standard tokens against 281.28 million for arm A. This is the cost in standard tokens.
- Easy text carries fewer bytes per token. The measured figures were 3.18 for the TinyStories part and 2.83 for the children's books, against 3.41 for standard text. At equal tokens, the easy arms therefore saw less text, not only a different order.
- Curves, seed 1: at half the run arm B was at 1.5483 against 1.4762 for A; at 90 percent it was at 1.4754 against 1.4508. Arm E was at 1.4797 at half the run, close to A, then rose to 1.8197 at 90 percent and ended at 1.7729.
- Reading: ending on easy text wiped out most of what the standard half had taught. The mechanism, that the last phase overwrites what was learned, is GUESS.

Corrections to our earlier notes:

- A round record described arm B as standard text for the last 40 percent. The data build and the design give easy text first (113.2 million tokens, 40.2 percent) and standard text last (168.1 million tokens, 59.8 percent), as stated above (FACT).
- The design note suggested decay 0.02 but left the choice open, and the round records did not say which value was used. The runs used the production-equivalent decay 0.0003, the lowest rung of the weight-decay round, for every arm (FACT, the run settings). Arm A's results (1.4073 and 1.4079) agree with that: they sit next to the 0.0003 runs of the weight-decay round (1.4041 to 1.4093) and far from the 0.02 runs (1.4986).

Published context (PUBLISHED, low confidence): published curriculum results are mixed and mostly small. The design note recorded search summaries of several papers, graded as low confidence (grade C), and we did not read them in full. One paper we opened is Elgaar (2026), "Curriculum Learning for LLM Pretraining: An Analysis of Learning Dynamics" (arXiv 2601.21698). Its abstract, read through a page-summary tool, reports that the stability gap between curricula and random ordering narrows as models get larger. The design note gave two authors and a 10 to 60 billion token range; the abstract we opened gives one first author and a 300 billion token budget, so only its qualitative claim is used here.

## Children's-literature data and licences

Context (FACT, research rules): a survey on 30 October looked for open or public-domain children's and child-level text, under a download cap of 50 GB. Machine-written text was allowed only when it was openly licensed and labelled with the model that wrote it. Sets whose generating model was not named were excluded.

Used in the curriculum screen (FACT):

- TinyStories, GPT-4 clean version: about 2.73 million short stories, machine-written by GPT-4 and labelled as such. Measured at 552.5 million tokens in the 32,768-token tokenizer, with a median reading grade of 3.1. It made up 20 percent of the tokens of arms B and D.
- Public-domain children's books from the Gutenberg collection: 6,925 books selected, 1.76 GB of unique text, taken from a public mirror because the site's robot policy says it is intended for human users (PUBLISHED: site policy and book pages). Public domain in the United States, book by book (PUBLISHED). It made up 20 percent of the tokens of arms B and D.
- OneStopEnglish: adult English-as-second-language articles at three levels, CC BY-SA 4.0 (PUBLISHED: repository). It made up 0.2 percent of the tokens of arm B.

Downloaded, not used in the screen (FACT): two of 43 shards of the Cosmopedia stories (Apache-2.0 on the card; generated by a named Mixtral model; median reading grade 11.9 on the part we tokenised, so not child-level). The other shards were not downloaded. The TinyStories validation file is kept for monitoring only.

| Source | Licence as recorded | Status |
|---|---|---|
| TinyStories GPT-4 (clean and validation) | CDLA-Sharing-1.0 (PUBLISHED: dataset card and hub metadata) | clean version used; validation for monitoring only |
| Gutenberg children's books | public domain in the United States, per book (PUBLISHED: book pages) | used |
| OneStopEnglish | CC BY-SA 4.0 (PUBLISHED: repository) | used, 0.2 percent of tokens |
| Cosmopedia stories | Apache-2.0 (PUBLISHED: dataset card and hub metadata) | two shards downloaded, not used |
| KidLM children's web text | CC BY-NC-SA 4.0, research use only; 67.97 million tokens (PUBLISHED: dataset card) | not used; licence decision pending |
| Children's Book Test | GNU Free Documentation License 1.3 (PUBLISHED: dataset README) | not used; licence decision pending |
| SimpleStories | MIT on the card; the card names no generating model (PUBLISHED: dataset card) | not used |
| TalkBank child-language corpora | the TalkBank rules bar including the data in models (PUBLISHED: TalkBank rules page) | skipped; also part of one 2026 benchmark corpus, which was skipped |
| PG-19 | Apache-2.0 on the repository; books before 1919; the page advises against general language-model training (PUBLISHED: repository and page) | not used |
| Internet Archive children's items | per item, from the archive's search listing; terms not read in full (PUBLISHED: archive search listing) | not downloaded |

Limits (FACT and GUESS): the Gutenberg books are older English and public domain in the United States only (FACT). The easy text is narrow in register, and the effect of that narrowness on results is GUESS.

## What this does not show

- Scale. The proxy has 4.46 million parameters and 281.28 million tokens, against a production plan of about 143 million parameters and 9 billion tokens. The published paper above reports that the stability gap between curricula and random ordering narrows as models get larger, so the screen's effects may not carry to the production model (PUBLISHED for the narrowing; GUESS for the transfer).
- Seeds. There were two seeds per arm. Arm A's two seeds differed by 0.0006, but two seeds is a thin basis for any claim (FACT, design).
- Confounds. The easy text is partly machine-written, narrower in register and shorter in bytes per token, so the arms differ in content and in text length, not only in order (FACT).
- Settings. One schedule and one decay value (production-equivalent 0.0003) were used for all arms, so the answer at another decay value is unknown (FACT).
- Arm C, the readability-sorted order, was not run (FACT).
- The decay matching assumes that the total decay over the run is what matters. The 32-fold difference in steps is the largest caveat for the decay results, and whether the best decay carries over to production is not tested (GUESS).
- The children's books are public domain in the United States only, and the easy-text arms saw fewer bytes at equal tokens (FACT).

## Open questions

1. Does ordering still beat shuffling when the model and the data are larger? (Not tested.)
2. Does a short easy phase, for example 5 to 10 percent of the run, change the answer? (Not tested.)
3. Does the answer change at another decay value? (Not tested; all arms used the production-equivalent 0.0003.)
4. The project chose to use the children's books in the reading experiments (document 08) rather than in pretraining. Does that choice hold once those experiments are read? (Open.)
