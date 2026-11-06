# 23. Reading the weights of real models, and a first small "eye" pipeline built from a released model

Snapshot dated 6 November 2026. This document reports a day of work on the stored weights of public models and on one released small vision model. It covers: how much instruction tuning and "learning to see" change the weights of Gemma 3, Qwen3, SmolLM2 and SmolVLM2; what is inside two open image generators; how a released 0.5 B vision model does on a tiny-detail test of ours; what happens when its weights are edited without training; and a first, small pipeline in which the same model decides where to look and then reads a crop.

Labels used throughout:

- **FACT**: read from a logged result file of ours (with its setting).
- **GUESS**: our reasoning or an estimate, not tested.
- **UNRESOLVED**: a reading that its own written rule left open (stated where it occurs).

All results come from one machine, one task family and a few hundred pictures, so treat them as indications.

## How the tests were run

- For the weight comparisons we fetched only the parts of the public model files that a test needed (single weight matrices, or the first rows of them) with ordinary range requests; about 3.4 GB in total, no login or token. For the vision tests we downloaded three complete released models (SmolVLM2-500M about 2.0 GB, SmolVLM2-256M about 1.0 GB, SmolLM2-360M-Instruct about 0.7 GB). The Gemma 3 files were read from public mirrors that carry the Gemma terms of use; the other models are listed as Apache-2.0 on their pages.
- Every test had a written rule with fixed thresholds. The rule file was fingerprinted (sha256 plus the time) in a log we only ever append to, before the script that tested it was copied to the machine, and the script was fingerprinted before its first run. Thresholds were never changed after results. A few rule files were written after earlier results of the same series were known; each says so in its background section. Where a rule turned out to be badly chosen we say so and keep the literal reading.
- "Relative distance" means the size of the difference between two weight matrices divided by the size of the original matrix: a distance in weight space, not a measure of behaviour.
- Three read-only AI review passes checked our notes and this text against the raw result files and found about 80 wording and number slips in total; every finding was corrected. We do not claim that none are left. One of our own scripts had a bug in its first run (a numerical re-check, found by a control and fixed).

## 1. Instruction tuning: how big is the change?

| Model pair | Median relative distance | Setting | Label |
|---|---|---|---|
| Gemma 3 4B, pretrained to instruction-tuned | 0.29 (0.23 if only the direction counts) | 15 matrices: 5 of 34 layers, query, value and MLP-output matrices | FACT |
| Qwen3-4B-Base to Qwen3-4B | 0.09 | 5 tensors: four layer matrices in 2 of 36 layers plus a slice of the embedding table | FACT |
| SmolLM2-360M to its Instruct version | 0.07 | 35 matrices in 5 of 32 layers | FACT |

- Gemma 3's post-training moved the sampled matrices three to four times more than Qwen3's or SmolLM2's. **FACT** for these samples.
- The Gemma change is not concentrated in a few directions: the 64 strongest directions of each matrix hold a median of 17 percent of the change's energy (our rule calls that "mixed"); 19 percent for the attention matrices and 6.5 percent for the MLP-output matrices. A rank-64 adapter would therefore capture roughly a fifth of an attention matrix's change and about a fifteenth of an MLP-output matrix's. **FACT.**
- Part of the Gemma change is a pure rescale of the matrix (median 38 percent of the squared change; individual matrices grew or shrank between 0.73 and 1.45 times). Gemma 3 normalises right after these matrices, so much of that scale change is probably harmless. **GUESS.**

## 2. Teaching a language model to see

- SmolVLM2-500M's language backbone ended 0.23 from SmolLM2-360M-Instruct and 0.24 from the base model (median over 35 sampled matrices; every sampled layer and matrix type moved about the same, 0.23 to 0.26). Its vision encoder is 0.28 from Google's released SigLIP-base encoder (18 matrices; none identical). **FACT.**
- Qwen3-VL-4B's language model is 0.25 from Qwen3-4B and 0.24 from Qwen3-4B-Base (five sampled tensors including a 2,048-row slice of the embedding table), or 0.27 and 0.26 without that slice. The parent cannot be told apart from the distances alone. **FACT.**
- So these released recipes did not leave the donor parts as they were: the backbone moved by about three times the instruction-tuning step and the vision encoder by a similar amount. The distances are **FACT**; what they mean is a **GUESS**.

## 3. What is inside two open image generators

- FLUX.2-klein's text encoder equals the released Qwen3-4B in all six tensors we compared in full (two attention matrices, two MLP matrices, 2,048 embedding rows, the final norm), and in a second pass, which compared only the first 8 rows of every matrix and every small tensor whole (398 tensors), it equals it everywhere. **FACT** (rows beyond the first 8 of most matrices were not compared).
- Z-Image-Turbo's text encoder was called different by our rule: one tensor, the last layer's MLP output, differs a lot (relative distance 0.39); the other five tensors we compared in full are identical. In the 8-row probe all 398 tensors equal Qwen3-4B's (the probe cannot see rows beyond the first 8, so it does not see that difference), and none of three other Qwen3-4B checkpoints (the base model and two later updates) matches any probed tensor. **FACT.**
- So two current open image generators appear to use an off-the-shelf language model as their text encoder (Z-Image with at least one edited matrix). Whether the encoder was frozen during training, and how the image network uses its states, was not measured. **GUESS.**

## 4. A released 0.5 B vision model on a tiny-detail test

Test: a 40-pixel coloured square (eight colours, chance 0.125) drawn on a photo stretched to 1920 x 800, 240 pictures, plus the same photos without the square as a control. Models used exactly as released, no training. "Oracle" means the crop is centred on the true square position.

| Setting | SmolVLM2-500M | SmolVLM2-256M |
|---|---|---|
| Default processing (picture split into 8 tiles plus a whole view, 576 image tokens) | **0.71** | **0.60** |
| Whole picture alone (64 tokens) | 0.45 | 0.11 |
| Oracle: one 512 x 512 crop at the true position (64 tokens, about 35 ms per pass with default attention) | **0.88** | 0.33 |
| Random 512 x 512 crop alone (64 tokens; the square lies fully inside it in 39 of 240 pictures) | 0.27 | 0.13 |
| No-square control, every setting | 0.10 to 0.13 | 0.09 to 0.15 |

All **FACT** (Wilson intervals are about plus or minus 0.04 to 0.065). Notes:

- Our own task-trained eye arm (frozen SigLIP so400m encoder, whole picture plus eight tiles = 441 tokens, a trained 2.7 M-parameter connector, 2,400 training steps) scored 0.78 on 300 other pictures; the same kind of arm with the picture squashed to one small view scored 0.14. Different pictures and training, so not directly comparable.
- The 500M oracle crop beats the default 576-token processing, but it uses the true position. **GUESS** that the default processing loses accuracy mainly by having to find the right tile among nine.
- The 256M model gains from the oracle crop (0.33 against 0.11 for the whole picture and 0.13 for a random crop), but less than our written bars (0.30 over a random crop, and 0.20 for the paired comparison), so the literal reading is "no crop effect". **GUESS** (reason): it was only trained on split-image input.
- **UNRESOLVED** by our own rule: two written conclusions about the 500M crop result (that one crop is enough, and that it is the crop that carries the information) were left open, because the control we required to stay under 0.45 already scored 0.46 (the whole picture alone scores 0.45; our bar was set too low). The differences between settings, which are large, are in the table.

### 4b. Deciding where to look, with no change to the model

We tested whether the released 500M model can also decide where to look. The whole picture is shown as one 64-token image; the attention of the last position over those 64 image tokens (per layer, averaged over heads) is read off; a 512 x 512 crop is taken around the chosen cell and read by the same model. The thresholds were fixed before the result run. A geometry check on separate synthetic pictures, run before the result run, caught a wrong assumption about how the processor places the picture (it stretches it to fill the frame rather than padding it); the correction was logged first.

| Selector | Image tokens (all passes) | Crop contains the whole square | Colour accuracy through the pipeline | No-square control |
|---|---|---|---|---|
| Random crop | 64 | 0.16 (39 of 240) | 0.27 | 0.12 |
| Training-free: strongest cell of the raw attention | 128 | 0.20 | 0.28 | 0.13 |
| Training-free: strongest cell of the attention divided by that under a generic prompt (fixed layers 12 to 20) | 192 | 0.54 (0.47 to 0.60) | 0.53 | 0.11 |
| Learned head, two prompts (128 numbers: one weight per attention feature, one bias per cell; trained on 2,400 program-made squares) | 192 | **0.88** (0.83 to 0.91) | **0.78** (0.72 to 0.83) | 0.13 |
| Oracle (true position) | 64 | 1.00 by construction | 0.88 | 0.10 |
| Default tiling | 576 | - | 0.71 | 0.10 |

All **FACT** (240 test pictures; the head was trained on photos disjoint from the test photos and the model was used unchanged, with no gradient through it). Notes:

- On the same 240 pictures the learned two-prompt pipeline got 187 right and the default tiling 170 (learned only 38, default only 21; exact McNemar p about 0.04), so "above the default tiling" is a modest margin with overlapping intervals, at a third of the image tokens.
- When the chosen crop contains the square the colour answer is right 87 percent of the time (oracle crop: 88 percent); when it misses, 14 percent. About 85 percent of the gap to the oracle comes from the 12 percent of pictures where the selector misses.
- Taking the strongest cell of the raw attention is useless: it picks the bottom-right cell in 208 of 240 pictures (probably an attention sink; not tested). Dividing by the attention under a generic prompt removes that, and a learned head can then use the raw features well (see the next point).
- The head needs few labels: trained on 120, 240, 480, 1,200 and 2,400 pictures its validation hit rate is 0.79, 0.83, 0.84, 0.86 and 0.88; training and validation agree (0.87 and 0.88). One prompt is enough: a head on the raw question-attention features alone keeps a validation hit rate of 0.877; reading only layers 12 to 20 costs about 9 points. A control with shuffled training labels gives 0.27, so the head is not fitting noise. **FACT** (validation pictures).
- **Confirmation on 240 fresh pictures** (no photo shared with training, validation or the first test) with the shortened one-prompt head: the crop contains the square in 0.87 (0.82 to 0.91), the colour accuracy through the pipeline is 0.83 (0.78 to 0.88), no-square control 0.13; the two-prompt head is not better (0.89 and 0.84). Median time per picture on our card: about 60 ms for the selector pass (eager attention) plus 35 ms to read the crop, about 96 ms in total, 128 image tokens (64 + 64). The fresh set happened to be easier for the reader than the first test set (right 93 percent of the time when the crop contained the square, against 87 percent), so the two accuracies are not directly comparable. **FACT.**
- **Other square sizes without retraining** (the head had only seen 40-pixel squares): the crop contains the square in 0.73 for 24-pixel squares, 0.95 for 56-pixel and 0.98 for 80-pixel squares (0.87 for 40 pixels on the same photos); the pipeline's colour accuracy is 0.65, 0.87 and 0.90 against oracle-crop accuracies of 0.85, 0.94 and 0.92. Our rule called the 24-pixel case "partly" and the other two "generalises". **FACT.**
- Limits: one synthetic task (a high-contrast square on photos), squares from the same generator in training and test, one model, indicative timings only. **GUESS:** for natural objects the same recipe would need labelled examples of the real target.

## 5. Editing the weights without training

- **Moving the "vision change".** We took the difference between SmolVLM2-500M's backbone and SmolLM2-360M-Instruct and added a fraction of it to the plain Instruct backbone (the eyes, connector and embeddings stay the model's own): 0 percent gives 0.33 (the plain Instruct backbone mostly answers "red", 155 of 240 times, with some real signal), 25 percent 0.33, 50 percent 0.32, 75 percent 0.58, 100 percent 0.71 (the original model). Adding the whole change to the base model gives 0.26 (confounded by the base model's missing instruction behaviour). A shuffled copy of the change gives 0.21. **FACT.** Under our written rule the change could neither be dialled down nor moved to the base model in this recipe; for our own model we take that as a reason to train the eyes rather than copy a difference in. This rests on one recipe and one task; other recipes (a transplant followed by a short fine-tune) were not tried.
- **Deleting layers.** Of the 32 backbone layers, 6 can be deleted one at a time at no cost and 11 are critical; deleting the 4 least harmful together costs nothing and saves about 7 percent of the time; deleting 6 together costs 17 points. None of the 12 vision-encoder layers can be deleted without losing at least 5 points. **FACT.** Deletion alone is not a route to a much faster eye.

## 6. Corrections and honesty notes

- Cosines computed in 32-bit floating point over tens of millions of values differed from the 64-bit values by up to 0.003 for the large matrices (up to 0.007 for the embedding table), the largest deviations making the 32-bit value too high. Relative distances moved by at most 0.0002; the median direction-only distance moved from our first estimate of 0.22 to 0.23; the rescale share and every verdict did not move. **FACT.**
- The rule for the Qwen3-VL drift said "medians over the sampled matrices" and the script counted an embedding slice among the five tensors. With it the drift from the closer parent is 0.24 (moderate under our thresholds), without it 0.26 (large). We report both. **FACT.**
- Our own review passes also caught: a first re-check script that compared two sets of numbers computed in the same precision; an overstated claim about image generators; a download tally that double-counted; and a token count that omitted one of the passes. All are corrected above.

## 7. What we take from it (GUESS unless stated)

1. A released 0.5 B vision model is a strong reference for our own small "eye": it reads a tiny detail at 0.71 with its default processing, 0.88 when a crop is placed at the true position (oracle), and 0.78 to 0.83 when our small learned head chooses the crop, at about one fifth to one third of the image tokens.
2. In this recipe the vision ability was not something to copy in as a stored difference; for our own models it comes from training.
3. Instruction tuning can be a heavy, spread-out change (Gemma) or a light one (Qwen3, SmolLM2); starting from an already instruction-tuned base is probably cheaper than reproducing it.
4. Image generators appear to be built on an unmodified or lightly edited language model; whether a separate image network reads its hidden states is how such designs are usually described, and was not measured here.

## 8. Open

- Refitting the selector for 24-pixel squares (the learning curve suggests about 120 pictures), then harder targets: shapes, several objects, natural targets.
- An optimised attention readout (last row only) instead of the full eager attention pass.
- Why the 256M model does not benefit from a crop.
- Whether a short repair fine-tune after deleting layers changes the picture.
- Repeating the instruction-tuning distances on more layers and matrix types.
