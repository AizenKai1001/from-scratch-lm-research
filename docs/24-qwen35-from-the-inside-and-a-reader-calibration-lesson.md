# 24. Qwen3.5 from the inside, and a reader calibration lesson

## What this is

Snapshot dated 6 November 2026. We report one day of tests on Qwen3.5-0.8B, a small open language model with a vision part (Apache-2.0 licence): its stored weights, its memory and speed on one 8 GB graphics card, and how well it names the colour of a small square in a picture. We also report a colour-reader bias in a smaller vision model, which changed some of our earlier numbers. This continues document 23. Treat the results as indications: one machine, one synthetic task, a few hundred pictures.

Labels used throughout. **FACT** is measured by us, with its setting and sample size. **COMPUTED** is arithmetic on stated inputs, not a measurement. **GUESS** is our reasoning, not tested.

## 1. What was measured and how

- Each test had a written rule with fixed thresholds, fingerprinted (a checksum and a time) in a log we only add to, before its script existed. Thresholds were not changed after results. Two steps went the other way round (sections 6 and 8). **FACT.**
- Released models were used as released: nothing was trained and no weight was changed. The calibration in section 8 changes only how scores are read. **FACT.**
- The picture tests use one synthetic task: a coloured square (eight colours, so chance is 0.125) on a photograph stretched to 1,920 by 800 pixels, with matching controls without a square. Intervals are 95 percent Wilson intervals. **FACT.**
- A separate AI agent rebuilt the whole-picture colour test from its written rule and got the same counts, though it had read our headline figure first. Six of our notes have not yet had an independent pass; five of them support the results here. **FACT.**
- A third step went the other way round (section 6): after the first long-context run failed at 262,144 tokens, we added 163,840, 196,608 and 229,376 tokens by a later addendum. These lengths are descriptive; no pre-set threshold changed. **FACT.**

## 2. The architecture in numbers

The model has about 873 million parameters. Their shares, from the stored weights (**FACT**):

| Part | Parameters | Share |
|---|---|---|
| Feed-forward blocks in the language model | 264.2 million | 30.3% |
| Vocabulary table (input and output share it) | 254.3 million | 29.1% |
| 18 linear-attention layers | 189.8 million | 21.7% |
| Vision tower | 100.6 million | 11.5% |
| 6 full-attention layers | 44.0 million | 5.0% |
| Multi-token-prediction head | 20.5 million | 2.3% |

The 24 layers repeat three linear-attention layers and then one full-attention layer, six times (**FACT**). A linear-attention layer keeps one fixed-size memory per head. A full-attention layer keeps a cache that grows with the text. Full-attention layers have 8 query heads and 2 key/value heads, and can address up to 262,144 positions.

From these values (**COMPUTED**): the 18 linear layers hold about 19 MB however long the text is. The six full-attention layers add about 12 KB per token, or 3.2 GB at 262,144 tokens.

## 3. The weights are dense

- For almost every kind of matrix, keeping the top quarter of its directions leaves an error of 48 to 71 percent of the matrix's own size (the median per kind). Keeping half still leaves 25 to 54 percent. **FACT**, for the 0.8B model.
- Under our written rule only two parts compress: the vision tower's position table and its patch embedding. Everything large is "dense" or "in between". **FACT.**
- So plain low-rank cutting is probably not the first experiment for this model. It measures weights, not accuracy. Methods that use the inputs can cut more, while quantisation (fewer bits per weight), layer changes and fewer image tokens do not depend on rank. **GUESS.**
- A few entries exceed 30 times the root-mean-square size of their matrix; the largest, 127 times, is in the multi-token-prediction head. Such entries can set the scale when weights are rounded. **FACT** for where; **GUESS** for why it matters.

## 4. How long the recurrent memories last

Each of the 288 linear-attention heads (18 layers, 16 heads each) keeps a memory that fades with every token. From the stored rates at zero input (no text), we computed each head's half-life: the number of tokens until half of its memory has gone. **COMPUTED.**

- Most heads forget fast, and the slow ones are few and mostly early (table). **COMPUTED.**

| Measure (stored rates at zero input) | Value |
|---|---|
| Heads that lose half their memory within 100 tokens | 183 of 288 (63.5%) |
| Median head's half-life | 14.5 tokens |
| Heads lasting beyond 1,000 tokens | 8 (2.8%), six of them in layers 0 to 4 (four in layer 0) |
| Practically permanent heads | 1 |

- The input term is large (median size 0.78), so the stored rate is only a starting point. The words of a text probably change how fast each head forgets. **COMPUTED** for the size; **GUESS** for the rest.
- Our prediction (**GUESS**): facts older than a few hundred tokens are mostly kept by the full-attention caches, unless the input slows a head's forgetting for the words that matter. So the zero-input half-lives may be a lower bound (**GUESS**). The input term could also shorten a head's memory. Our recall test cannot separate the two routes, because the caches hold every token.

## 5. The vocabulary: most of it is unused in English, but the rows are not dead

- The vocabulary has 248,320 tokens and makes up 29.1 percent of the model. In 15.8 million tokens of public-domain children's books (about 80 MB), only 47,913 tokens (19.3 percent) occur at all. **FACT**, for one narrow English domain.
- Rarely used rows are not smaller, duplicated or lower-dimensional than common ones. Median row length is 0.624 for the rarest tokens and 0.627 for the common ones. Near-duplicates appear in 0.2 percent of a rare-row sample and in none of the common sample (the exception is the 338 base tokens, which are special, byte, prompt and colour tokens). All three pre-stated tests read "no". **FACT.**
- The rarer the token, the more its row leans toward the table's average direction: the median cosine (how closely two rows point the same way) rises from 0.16 for the 338 base tokens (special, byte, prompt and colour tokens) to 0.48 for the rarest. That fits rows that were trained little, but it does not make them smaller. **FACT** for the trend; **GUESS** for the explanation.
- So sharing or low-rank compression of the vocabulary is not supported, and removing tokens is the candidate for saving memory (**GUESS**; a trimming test is queued, not run).

## 6. Long context on one 8 GB card

Memory grows with text length, so long prompts were fed in pieces of 2,048 tokens, carrying memory from piece to piece ("chunked prefill"). In every test the chunks gave the same next word as one pass, with one near-tie miss (table). **FACT.**

| Length (tokens) | Peak GPU memory (GB, decimal) | Tokens per second | Same next word as one pass | Code found |
|---|---|---|---|---|
| 8,192 | 2.04 | 10,993 | 20 of 20 | 15 of 15 |
| 16,384 | 2.26 | 9,224 | 19 of 20 | not run |
| 32,768 | 2.70 | 7,007 | not tested | 15 of 15 |
| 65,536 | 3.57 | 4,707 | not tested | 15 of 15 |
| 131,072 | 5.31 | 2,853 | not tested | 9 of 9 |
| 163,840 | 6.19 | 2,343 | not tested | not run |
| 196,608, 229,376, 262,144 | out of memory as implemented | - | - | not run |

**FACT** for every cell.

- The longest prompt that fitted (163,840 tokens) ran at 2,343 tokens per second, about 70 seconds (**COMPUTED**). Above 65,536 tokens there was one timed run and no warm-up, still in 2,048-token pieces.
- Memory grew by about 27 KB per token between 65,536 and 131,072 tokens (**COMPUTED** from the measured peaks). At that rate, 262,144 tokens would need about 8.8 GB, more than the card has (**COMPUTED**, extrapolated from the chunked runs; a single pass would extrapolate to about 23 GB).
- The failed runs showed about 1.5 GiB reserved but unused (1.47 GiB as PyTorch reports it), which looks like memory that cannot be placed in one piece (**GUESS**). The full context did not fit as implemented. Our earlier "about 6 GB at 262,144 tokens" figure counted weights and cache, plus a guessed 1 GB, and is not supported by measurement. **FACT.**
- Recall (**FACT**): a four-digit code sits in one sentence of children's-book text, and the question asks only for it. This is the easy case of "needle in a haystack", and it does not show that the model uses the rest of a long text. The 95 percent lower bounds for those cells are 0.80 and 0.70.
- Scoring bug, found and fixed: the first run allowed only 12 new tokens, so answers were cut before their last digit. It scored 34 of 45, and all 11 misses were cut answers. We found this by reading raw outputs, so the fix (32 new tokens, exactly four digits) is post hoc. Both versions are reported; results use the second. **FACT.**

## 7. Reading small squares

On the whole picture, Qwen3.5-0.8B names the colour of a 40-pixel square in 238 of 240 pictures (0.992). Smaller squares are harder (table). **FACT.**

Crops are 256 tokens, placed at the true square position (an "oracle" choice that skips the search). The 40-pixel crop was 512 by 512 at native scale. The last two columns use a 256 by 256 crop enlarged 2 or 4 times, still 256 tokens. **FACT.**

| Square side | 0.8B, whole picture | 2B, whole picture | 0.8B, crop enlarged 2 times | 0.8B, crop enlarged 4 times |
|---|---|---|---|---|
| 16 px | 0.954 (0.920 to 0.974) | 0.988 (0.964 to 0.996) | 0.996 | 1.000 |
| 12 px | 0.904 (0.860 to 0.935) | 0.983 (0.958 to 0.994) | 0.996 | 1.000 |
| 8 px | 0.813 (0.758 to 0.857) | 0.888 (0.841 to 0.922) | 0.992 | 0.996 |

- At 8 pixels the whole picture scores 0.81, just above our 0.80 zoom cut, so the zoom rule did not fire (0.79 would have fired it). Enlarged crops score 0.99 or better at every size (table). **FACT.**
- The 2B model is clearly better on the whole picture only at 12 pixels, by our rule. Its whole-picture pass took 672 ms, against 333 ms for the 0.8B model (both on the slow code path). **FACT.**
- With the optimised kernels, picture passes were about 1.3 times faster, short of our 1.5 times bar. Long text gained more, up to 1.73 times at 32,768 tokens (table). **FACT.**

| Pass | Reference code | Optimised kernels | Speed-up |
|---|---|---|---|
| Whole picture (1,500 tokens) | 332 ms | 259 ms | 1.28 |
| Crop (256 tokens) | 80 ms | 60 ms | 1.33 |
| Text (8,192 / 16,384 / 32,768 tokens) | - | - | 1.56 / 1.69 / 1.73 |

## 8. A measurement lesson: a colour reader with a systematic bias

Document 23 reported SmolVLM2-500M, a 0.5-billion-parameter vision model, at 0.88 with a true-position crop. Stored results afterwards showed systematic colour errors. Given a correct crop, the reader called cyan "blue" and misread white (table below). Red, blue, yellow and purple were always right. **FACT**, found after the fact.

| Colour (correct crop, uncalibrated reader) | Right |
|---|---|
| Cyan | 20 of 30 |
| White | 21 of 29 |
| Red, blue, yellow, purple | every picture |

- Not the tokenizer. Scoring each colour by its whole spelling (cyan is two tokens) gave exactly the same 240 answers, so the scores already rank blue above cyan for many cyan squares. **FACT.** A shift per colour fixes that ranking, so the fault looks like calibration rather than seeing (**GUESS**).
- The fix is one fixed number per colour, added to the reader's scores and fitted without labels: over 240 unlabelled pictures, each colour should get the same average probability. This is valid here because the generator draws the colours equally often. **FACT** for the method.
- The fitted offsets are in the table below. **FACT.**

| Colour | Red | Green | Blue | Yellow | Orange | Purple | White | Cyan |
|---|---|---|---|---|---|---|---|---|
| Offset | -0.28 | -0.42 | -1.48 | -1.41 | +0.78 | +1.55 | +0.54 | +1.85 |

- We tested the numbers under a pre-registered rule on 240 new pictures, none used in the fit. Eight fixed numbers raised pipeline accuracy from 0.74 to 0.82, and accuracy on a correct crop from 0.89 to 0.97. **FACT.**

| Reader | Pipeline accuracy | Accuracy given a crop with the square | No-square controls |
|---|---|---|---|
| Uncalibrated (before) | 0.742 (178 of 240) | 0.888 | 0.108 |
| Eight fixed numbers (after) | 0.817 (196 of 240) | 0.975 | 0.088 |

No-square controls: the share of no-square pictures whose answer matches the colour the square would have had. Chance is 0.125; lower is better, and the bar is 0.20. **FACT.**

- The McNemar test gave p = 0.0001: more pictures were fixed than broken. Labels added nothing: fitting with them gave 0.821, as did fitting on only 60 unlabelled pictures. **FACT.**
- Limits (**GUESS** for natural scenes): the fit assumes equal colour frequencies, true here by construction, while natural pictures have an unknown mix. One reader and one task are not enough to generalise.
- What it changes: every SmolVLM2 colour figure in our notes and in document 23 used the uncalibrated reader (**FACT**). Calibration raised accuracy on one new set. We did not re-score the earlier sets, so these figures are probably floors (**GUESS**). The Qwen reader was not checked for this bias, though it is near the ceiling. The main limit is now finding the square (**COMPUTED**): the crop held it in 196 of 240 pictures here, against 0.871 to 0.879 on earlier sets.

## 9. What we take from it, and what is open

What we take (**GUESS** unless marked):

1. On this synthetic task, Qwen3.5-0.8B is a viable base for a small eye, and it needs no search for 40-pixel squares. **FACT** for the test; natural pictures are untested.
2. Chunked runs, in pieces of 2,048 tokens, completed on one 8 GB card up to 163,840 tokens. Full single passes were measured only up to 32,768 tokens. **FACT** for those ranges.
3. In our case (**FACT**), a reader had a per-colour bias that its accuracy did not show. We would calibrate and re-check on new pictures before comparing models (**GUESS** for other readers).
4. For this model, memory is more likely saved by removing vocabulary, quantising or using fewer image tokens than by low-rank cuts.

What is open: harder and natural targets; zoom that needs a position, with a selector good enough for this model (untested); recall with many similar facts, and a test that separates the two memory routes; going past 163,840 tokens (an allocator setting and an 8-bit cache are untried); an independent rebuild of the calibration; and the second-look and two-square tests, to be re-run with the calibrated reader (planned, not run).

## Corrections and honesty notes

- The recall scoring bug was found and fixed after reading raw outputs, so the fix is post hoc (section 6).
- Three lengths (163,840, 196,608 and 229,376 tokens) were added by a later addendum after the 262,144-token run failed (section 6).
- Our earlier guess that the kernels make the model "several times faster" was too high (section 7).
- The "about 6 GB at 262,144 tokens" figure was never measured (section 6).
- Every SmolVLM2 colour figure here and in document 23 used the uncalibrated reader, so these figures are probably floors (GUESS) (section 8).
- The zoom rule missed firing at 8 pixels by 0.0125. The 4-times zoom used bicubic enlargement, which the rule named only for 2 times (section 7).
- The colour replication was not fully independent: its agent saw the headline figure and part of the drawing code first, and it covered only the whole-picture view.
- The calibration idea came from a post hoc look at stored results; the test on new pictures was registered beforehand (section 8).
