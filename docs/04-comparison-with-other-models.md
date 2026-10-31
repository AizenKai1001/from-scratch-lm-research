# 04 How our model compares with other models

How to read the labels. FACT is measured in our own runs, with the setting stated. PUBLISHED is taken from a paper or page we opened, named in the text and listed at the end. GUESS is reasoning that was not tested. Arithmetic on stated inputs is marked as such.

## Summary

We compared the 143-million-parameter model with public models of similar size in three ways: the same four zero-shot benchmarks run through one script for every model, bits per byte on the same held-out web text, and the numbers other authors have published. The snapshot used is session 36 (1.85 billion training tokens, no cooldown yet), with sessions 11 and 20 for the trend.

- **Benchmarks (FACT).** At session 36 our model is level with GPT-2 small and Pythia-160M on HellaSwag and the two ARC tests, a little higher on ARC-Easy, clearly lower on LAMBADA, and well above Pythia-70M. It is far below SmolLM-135M. HellaSwag, ARC-Easy and LAMBADA rose steadily from session 11 to 20 to 36; the ARC-Challenge scores stayed within noise.
- **Bits per byte on held-out text of the kind we train on (FACT).** Our model needs 1.080. GPT-2 small needs 0.986, Pythia-160M 0.982 and SmolLM-135M 0.846; Pythia-70M needs 1.113. So we are 0.095 behind GPT-2 small and 0.033 ahead of Pythia-70M.
- **Training data.** We got there with 1.85 billion training tokens. GPT-2 small saw an estimated 9 billion or more, Pythia 300 billion and SmolLM-135M 600 billion.
- **What this does not show.** It does not rank recipes. The models differ in data by up to 300 times, our model is unfinished, our evaluation text comes from the same family as our training data, and we have not trained a baseline with the standard recipe on our own data, so we cannot say whether our recipe is better or worse than the standard one. The comparison is a position report, not a verdict.

## 1. The models

| Model | Parameters (counted) | Training tokens | Tokens per parameter (arithmetic) | Training data |
|---|---|---|---|---|
| Ours, session 11 | 142.9M | 0.60 billion | 4 | FineWeb-Edu sample and a broad pool of six human-written corpora (documents 03 and 06) |
| Ours, session 20 | 142.9M | 1.05 billion | 7 | same |
| Ours, session 36 | 142.9M | 1.85 billion | 13 | same |
| Our earlier run A (document 07) | 142.9M | 2.29 billion: 1.88 billion in a rented-GPU run, then 0.41 billion on our educational web file, ending with a cooldown session | 16 | different, see text |
| GPT-2 small | 124.4M | not stated by the authors; an open replication of the same kind of web corpus has about 9 billion (estimate) | about 70 (estimate) | WebText: pages linked from Reddit, Wikipedia removed (PUBLISHED) |
| Pythia-70M | 70.4M | 299.9 billion | 4,260 | The Pile (PUBLISHED) |
| Pythia-160M | 162.3M | 299.9 billion | 1,850 | The Pile (PUBLISHED) |
| SmolLM-135M | 134.5M | 600 billion | 4,460 | FineWeb-Edu (deduplicated, 220 billion tokens), synthetic textbooks and stories (28 billion), Python code (4 billion) (PUBLISHED) |

The three SmolLM components add up to 252 billion unique tokens, so 600 billion training tokens implies the data were seen about 2.4 times (arithmetic on the numbers in source [8]; the page does not say so). For scale, the Chinchilla paper's example model has 20 tokens per parameter [9], and the Cerebras-GPT authors state that about 20 gives the most efficient use of compute [3]. Our planned 9 billion tokens would be 63 per parameter (document 02).

## 2. Four zero-shot benchmarks, one script (FACT)

**Setting.** The four tests are HellaSwag (all 10,042 validation items), ARC-Easy (2,376 test items), ARC-Challenge (1,172 test items) and LAMBADA (5,153 test items, last-word accuracy and perplexity); the tests are described in source [10]. Each multiple-choice item is scored zero-shot: the model gets no examples, and its answer is the choice with the highest summed log-probability (accuracy) or the highest log-probability per byte of the answer (length-normalised accuracy, "norm"). Chance is 25 percent on HellaSwag and about 25 percent on ARC. One script and one set of preprocessing steps were used for every model, run in 32-bit arithmetic with a 1,024-token context; the public models were loaded through the standard Hugging Face interface. One run per model. A standard error is about 0.45 points on HellaSwag, 1.0 on ARC-Easy, 1.2 on ARC-Challenge and 0.6 on LAMBADA accuracy, so two models need to differ by about 1.3, 2.9, 3.3 and 1.8 points to be more than two standard errors apart (independent-sample rule of thumb; the questions are shared, so paired differences are somewhat tighter).

| Model | HellaSwag acc / norm | ARC-Easy acc / norm | ARC-Challenge acc / norm | LAMBADA acc | LAMBADA perplexity |
|---|---|---|---|---|---|
| **Ours, session 11 (0.60B tokens)** | 27.2 / 28.8 | 41.8 / 39.4 | 19.4 / 24.0 | 16.8 | 535.7 |
| **Ours, session 20 (1.05B tokens)** | 27.5 / 28.9 | 43.9 / 41.9 | 18.9 / 23.3 | 23.3 | 134.8 |
| **Ours, session 36 (1.85B tokens)** | 28.6 / 30.5 | 46.4 / 43.1 | 19.7 / 24.5 | 26.8 | 79.7 |
| Our earlier run A (2.29B tokens, cooled) | 29.2 / 32.2 | 49.7 / 44.2 | 21.8 / 25.1 | 29.7 | 61.0 |
| Pythia-70M (300B tokens) | 26.7 / 27.5 | 37.5 / 35.2 | 17.5 / 21.3 | 22.7 | 131.0 |
| GPT-2 small | 28.9 / 31.2 | 43.8 / 39.9 | 19.0 / 22.4 | 32.6 | 40.1 |
| Pythia-160M (300B tokens) | 28.4 / 30.2 | 43.6 / 40.1 | 19.5 / 23.5 | 35.4 | 37.3 |
| SmolLM-135M (600B tokens) | 35.2 / 42.7 | 61.6 / 57.2 | 26.0 / 29.0 | 37.7 | 32.3 |

**Reading (FACT for the numbers, GUESS for the interpretation).**

1. **Our model improves with data.** From session 11 to 36 every column is higher: HellaSwag by 1.6 points normalised, ARC-Easy by 4.6, LAMBADA accuracy by 10.0 points and LAMBADA perplexity falls from 536 to 80. The two ARC-Challenge columns moved by 0.3 and 0.5 points, less than the noise.
2. **At session 36 our model is level with the two standard 124M to 162M models on three tests.** HellaSwag normalised 30.5 against 31.2 (GPT-2 small) and 30.2 (Pythia-160M); ARC-Challenge 24.5 against 22.4 and 23.5 (normalised); ARC-Easy 46.4 against 43.8 and 43.6, which is about 1.8 to 1.9 standard errors of the difference above: suggestive, not clear. It is clearly above Pythia-70M on HellaSwag, ARC-Easy and LAMBADA.
3. **LAMBADA is our weak spot:** 26.8 percent against 32.6 and 35.4 for GPT-2 small and Pythia-160M. GUESS: LAMBADA is built from passages of unpublished novels, our training mix is mostly educational web text, and fiction makes up about a tenth of the pool.
4. **SmolLM-135M is far ahead of us:** HellaSwag normalised 42.7 against our 30.5, ARC-Easy 61.6 against 46.4, ARC-Challenge 29.0 against 24.5 (normalised), LAMBADA 37.7 against 26.8. It saw about 320 times as many tokens, partly synthetic textbook text.
5. **Run A, a different and cooled model, scores higher than our current uncooled snapshot** (HellaSwag normalised 32.2 against 30.5, ARC-Easy 49.7 against 46.4, LAMBADA 29.7 against 26.8). It saw 2.29 billion tokens on different data and ended with a cooldown session, so the difference mixes cooling, data and token count; it is the best hint we have of what a cooldown may add (GUESS), not a measurement of it.
6. **The script reproduces published numbers for other people's models.** For Pythia-160M the published ARC-Easy is 43.5 and 43.2 (two papers) against our 43.6; HellaSwag normalised is 30.2 against our 30.2; LAMBADA perplexity is 38.1 against our 37.3; for Pythia-70M the published ARC-Easy is 37.4 against our 37.5; for SmolLM-135M the published HellaSwag is 41.2 against our 42.7 (normalised), within about a point and a half. LAMBADA accuracy differs more between sources (32.8, 33.0 and 38.9 for Pythia-160M; ours 35.4; Pythia-70M published 18.5, ours 22.7), as section 4 shows. So differences of a point or two on ARC and HellaSwag between our table and a paper are within the spread between published sources.

## 3. Bits per byte on the same held-out text (FACT)

Bits per byte is a measure of how well a model predicts a text, independent of the tokenizer, so models with different vocabularies can be compared on one fixed text. Lower is better.

**Text.** 234 documents from the held-out part of our validation file (a held-out part of the FineWeb-Edu sample that no model of ours trained on). The documents come from a region of that file that our own training-time validation does not use. Each document is cut to its first 3,500 characters, which gives 821,698 bytes in total. The brief was for 500 documents; only 234 qualified, and all 234 were used.

**Scoring.** The same rule for every model: the text is preceded by the model's end-of-text token, the model's own tokenizer is used, sequences longer than 1,024 tokens are cut, the log-loss of every token of the text is summed (the end-of-text token only provides context), and bits per byte is that sum divided by ln 2 and by the number of bytes of the text covered. 32-bit arithmetic, one document at a time. A repeat run of GPT-2 small on 20 documents gave identical numbers. Two SmolLM documents were cut at 1,024 tokens; leaving them and the same documents out for all models changed no number by more than 0.002 and did not change the ranking.

| Model | Bits per byte | 95% interval | Difference from ours (session 36), paired, with 95% interval |
|---|---|---|---|
| SmolLM-135M | 0.846 | 0.829 to 0.864 | -0.235 (-0.243 to -0.227) |
| GPT-2 medium (355M) | 0.898 | 0.883 to 0.916 | -0.182 (-0.189 to -0.176) |
| Pythia-160M | 0.982 | 0.966 to 1.001 | -0.098 (-0.104 to -0.092) |
| GPT-2 small (124M) | 0.986 | 0.969 to 1.004 | -0.095 (-0.100 to -0.089) |
| **Ours, session 36** | **1.080** | 1.062 to 1.101 | reference |
| Pythia-70M | 1.113 | 1.096 to 1.133 | +0.033 (+0.027 to +0.039) |
| Ours, session 20 | 1.150 | 1.130 to 1.170 | +0.069 (+0.066 to +0.072) |

Intervals are 95 percent bootstrap intervals over documents (1,000 resamples). GPT-2 small and Pythia-160M are not separated here (0.003 apart). The step from session 20 to 36 is 0.069, the same as our trainer's own change on its validation region (1.1925 to 1.1238), a rough consistency check.

**Reading and caveats.**

1. **We are 0.095 bits per byte (about 9 percent) behind GPT-2 small and 0.033 ahead of Pythia-70M**, on text of the kind we trained on.
2. **The comparison is tilted in our favour and, for SmolLM, in its favour.** (a) Our model trained on the same family of text (FineWeb-Edu pages), so this text is in-distribution for us and not for GPT-2 or the Pythia models. GUESS: on text from outside our training family the gap to GPT-2 small would be larger. (b) SmolLM-135M was trained on about 220 billion tokens of a deduplicated FineWeb-Edu subset, so some of our evaluation pages, or near copies of them, may well be in its training data. We did not check, and its lead of 0.235 may partly come from that and from its synthetic textbook text.
3. **234 documents** is a small sample. The intervals are about plus or minus 0.02 for one model and 0.003 to 0.008 for a paired difference.
4. **Context** was 1,024 tokens for every model. Pythia and SmolLM accept 2,048, and might score differently with more.
5. **Tokenizers differ** (32,768 entries for ours, about 50,000 for GPT-2 and Pythia, 49,152 for SmolLM). Bits per byte removes this on the same text, but the number of tokens each model predicts differs.
6. **One kind of text.** Nothing here measures code, mathematics or dialogue.
7. **Our 1.080 is not the 1.1238 the trainer reports for session 36.** They differ through the sample (different documents) and the normalisation (the trainer divides by a fixed average of 4.56 bytes per token; this sample has 4.65). Numbers from the two must not be compared.

## 4. What other authors published (PUBLISHED, setups differ)

The table collects zero-shot scores published for small models. The setups are not the same (harness versions, normalisation and prompt formats differ and are often not stated), so differences of a few points between rows are not meaningful. "n/f" means no source we opened gives the value.

| Model | Parameters | Training tokens | HellaSwag | ARC-Easy | ARC-Challenge | LAMBADA accuracy (perplexity) | Source |
|---|---|---|---|---|---|---|---|
| GPT-2 small | 124M | not stated | n/f | n/f | n/f | 45.99 (35.13), the paper's own setup | [5] |
| GPT-Neo-125M | 125M | 300 billion | 29.7 | 40.7 | 24.8 | n/f | [4] |
| OPT-125M | 125M | not stated for this size (corpus 180 billion) | 29.2 | 43.5 | 18.9 | 37.9 | [3] |
| OPT-125M, other source | 125M | as above | 31.1 | 41.3 | 25.2 | n/f | [4] |
| Pythia-70M | 70M | 299.9 billion | n/f | 37.4 | 18.1 | 18.5 | [1] |
| Pythia-160M | 160M | 299.9 billion | n/f | 43.5 | 18.8 | 32.8 | [1] |
| Pythia-160M, three more sources | 160M to 162M | as above | 30.2 / 29.9 / 29.3 | 43.2 / 40.0 / 45.2 | 24.1 / 25.3 / 18.1 | 33.0 (38.10) / n/f / 38.9 | [2] / [4] / [3] |
| SmolLM-135M | 135M | 600 billion | 41.2 | average of the two ARC sets 42.4 | | n/f | [6]; tokens [8] |
| SmolLM2-135M | 135M | 2 trillion | 42.1 | average of the two ARC sets 43.9 | | n/f | [6] |
| Cerebras-GPT-111M | 111M | 2.2 billion | 26.8 | 38.0 | 16.6 | 19.4 | [3] |
| MobileLLM-125M | 125M | 1 trillion | 38.9 | 43.9 | 27.1 | n/f | [4] |
| Mamba-130M (not a transformer) | 130M | 300 billion | 35.3 | 48.0 | 24.3 | 44.3 (16.07) | [2] |

Notes.

1. **The same model gets different numbers in different sources.** Pythia-160M's LAMBADA accuracy is 32.8, 33.0 and 38.9, its ARC-Easy 43.5, 43.2, 45.2 and 40.0, and its ARC-Challenge ranges from 18.1 to 25.3. The sources do not say why; GUESS: harness version, prompt format, normalisation. A paper on evaluation reports that changing only the prompt format can move scores by more than 20 percent in some cases [7].
2. **Published token budgets run from 2.2 billion to 2 trillion.** Only Cerebras-GPT-111M (2.2 billion tokens, 20 per parameter) is close to our budget. Our session-20 snapshot (1.05 billion tokens) scores above its published numbers on all four tests (ARC-Easy 43.9 or 41.9 against 38.0, depending on normalisation; ARC-Challenge 18.9 or 23.3 against 16.6; HellaSwag 27.5 or 28.9 against 26.8; LAMBADA 23.3 against 19.4), but the data, harness and normalisation differ, so this is a hint, not a ranking.
3. **Where our model sits.** Our session-36 numbers lie inside the range of the published 124M to 162M models on HellaSwag (30.5 against 29 to 31) and ARC-Challenge (24.5 normalised against 18 to 25), slightly above it on ARC-Easy (46.4 against 40 to 45), below it on LAMBADA (26.8 against 33 to 39), and well below SmolLM and MobileLLM, which trained on 600 billion tokens or more.
4. **The GPT-2 paper's LAMBADA numbers** (45.99 and 35.13) come from the paper's own evaluation setup and are not comparable with the 32.6 and 40.1 that the standard setup (and our script) gives for the same model.

## 5. What the comparisons show, and what they do not

**They show:**
- Where an unfinished 143M model trained on 1.85 billion tokens stands: level with GPT-2 small and Pythia-160M on three standard small-model tests and 0.1 bits per byte behind them on held-out web text, above Pythia-70M, and far from SmolLM-135M.
- That HellaSwag, ARC-Easy and LAMBADA scores rise steadily with tokens (session 11, 20, 36), without the cooldown.
- That a model with 160 times fewer training tokens than Pythia-160M can match it on several tests. GUESS: much of this comes from the quality and educational slant of the data (FineWeb-Edu is filtered for educational content, which should help science-question tests such as ARC), not from our recipe.

**They do not show:**
- That our recipe is better than the standard one. The models differ in data, token count and architecture; we have no standard-recipe baseline on our data at an equal budget. This would be the real test of the recipe (document 11).
- What the finished model will score. The planned 9 billion tokens and the cooldown have not run.
- Anything about other skills: instruction following, reasoning, code, dialogue.
- A fair contest on bits per byte: our evaluation text is in our training family, and SmolLM's training data may contain it.

## Open questions

1. How does the model compare at the end of the planned run (9 billion tokens, after the cooldown)? This page's comparison is to be rerun then.
2. What does a standard recipe (AdamW, cosine schedule, large batches) score on our data at the same 1.85 billion tokens? This needs a second training run and would separate the recipe from the data.
3. How do Pythia's public intermediate checkpoints, at the same number of tokens as ours (about 2 billion), compare? Not run.
4. How much of SmolLM-135M's advantage on our text is overlap with its training data? A page-overlap check would answer it.

## Sources

All PUBLISHED, opened by us unless marked.

1. Biderman et al. 2023, "Pythia: A Suite for Analyzing Large Language Models Across Training and Scaling", arXiv 2304.01373 (zero-shot Table 7; 299,892,736,000 training tokens in Section 2.4).
2. Gu and Dao 2023, "Mamba: Linear-Time Sequence Modeling with Selective State Spaces", arXiv 2312.00752 (Table 1).
3. Dey et al. 2023, "Cerebras-GPT: Open Compute-Optimal Language Models Trained on the Cerebras Wafer-Scale Cluster", arXiv 2304.03208 (Tables 1 and 8; Section 2.3).
4. Liu et al. 2024, "MobileLLM: Optimizing Sub-billion Parameter Language Models for On-Device Use Cases", arXiv 2402.14905 (Table 3).
5. OpenAI 2019, "Language Models are Unsupervised Multitask Learners", zero-shot table as copied on the Hugging Face model card of GPT-2 (the paper PDF itself could not be read with our tool).
6. HuggingFaceTB, model card of SmolLM2-135M on Hugging Face: comparison table of SmolLM2-135M and SmolLM-135M, "all evaluations are zero-shot unless stated otherwise ... lighteval", 2 trillion tokens for SmolLM2.
7. Biderman et al. 2024, "Lessons from the Trenches on Reproducible Evaluation of Language Models", arXiv 2405.14782.
8. Ben Allal, Lozhkov and Bakouch 2024, Hugging Face blog "SmolLM" (16 July 2024): 600 billion tokens for the 135M model; corpus parts of 220, 28 and 4 billion tokens.
9. Hoffmann et al. 2022, "Training Compute-Optimal Large Language Models", arXiv 2203.15556.
10. Zellers et al. 2019 (HellaSwag, arXiv 1905.07830); Clark et al. 2018 (ARC, arXiv 1803.05457); Paperno et al. 2016 (LAMBADA, arXiv 1606.06031).
