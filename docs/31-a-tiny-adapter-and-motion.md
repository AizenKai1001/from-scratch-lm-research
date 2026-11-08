# 31. A trail in one picture, a small adapter, and which way things move

## What this is

Snapshot dated 8 November 2026. This document reports one test, PERC-VID-3. The idea comes from onion-skin drawing, where an animator places faded copies of earlier frames under the current one, so motion shows up as a trail. We asked two questions. Can one picture with that trail drawn in teach a small language model to say which way a shape is moving? And does it do better than plain two-frame video, given to the same model at the same picture budget?

The language model is Qwen3.5-0.8B, with its own weights frozen. Only a small add-on, called an adapter, is trained. Every clip and every correct answer was made by code. No film, no downloaded footage and no model-written text was used.

Each test had a written rule with fixed bars, registered with a checksum and a time before the script it governed ran. Treat every result as an indication, not a proof. Misses are reported as plainly as passes. The test builds on documents 28 and 29, which used the same small picture input on generated shapes and on photographs.

## In short

In our runs, the trail picture did better than plain two-frame video on the main test, with the same model and the same budget. It named the direction well for shapes moving from 8 to 34 pixels per frame, the fastest speed we tested. It failed at 4 pixels per frame. It also did not recognise "standing still" when that answer was worded in new words. The comparison with video is not a fair one, because the trail holds three steps of motion and the video holds one. The still-frame control stayed under its limit, but that control also removes the trail's brightness, so this run cannot show that the trail alone does the work.

## Summary

| Question | Verdict, as the rules read it | Main number |
|---|---|---|
| Does a trail drawn into one picture let the frozen 0.8B model name the direction, with the 16,384-number adapter A? | MOTION-UNLOCKED | 0.986 on the main test (chance 0.20); all five bars passed |
| Same, with the larger adapter B (2,031,616 numbers)? | MOTION-UNLOCKED | 0.994 on the main test; all five bars passed |
| Does plain two-frame video do the same? | A: MOTION-NOT-UNLOCKED. B: MOTION-IN-DISTRIBUTION-ONLY | 0.882 (A) and 0.928 (B) on the main test, against the 0.90 bar |
| Are slow motion (4 pixels per frame) and fast motion read alike? | No. The speed bar passes only on the average | 0.4197 (trail A) and 0.2642 (trail B) at 4 pixels, against 0.9952 and 1.0000 at 30 to 34 |
| Is "standing still" recognised in new words? | No | 7 of 100 (trail A); 0 of 100 (trail B) |
| Does the trail alone carry direction, apart from brightness? | Not shown | The still-frame control gets 0.288, under its 0.40 limit, but it removes trail and brightness together |
| Does the 2B model pass? | Not tested | Its first training step hit the memory guard |

## 1. Why this test

An earlier test (PERC-VID-1) found that the frozen model, without training, cannot name the direction. In this test's own zero-shot runs, with no training at all, it scored between 0.234 and 0.314 on the main test set, against a chance level of 0.20 for five answers. An earlier linear-probe test (PERC-VID-2) found that the frozen picture features carried direction most clearly in the trail picture, then in two-frame video, and least in a still frame. That probe is not repeated here. The question for this test was whether a small trained part could make the language side use that signal.

## 2. The setup in plain words

**Clips.** Each clip has four frames at 12 frames per second on a 448 by 192 pixel canvas. A shape moves left, right, up or down, or it stays still. The five answers are LEFT, RIGHT, UP, DOWN and STATIC. Training clips used a circle, a square or a triangle, sized 32 to 48 pixels, in eight randomly chosen colours. The shape moved at 8 to 24 pixels per frame, on a grey background with noise.

**Three picture forms, all on the same canvas.** Both motion forms use 84 picture tokens, the same budget as the glance in document 28. A picture token is one slot of picture input that the language model reads.

- The trail picture: all four frames drawn into one picture. The newest copy of a moving shape is drawn at about 53 percent of full strength, and the older copies are fainter still. A still shape keeps its full colour.
- Two-frame video: the last two frames of the clip, shown as a short two-frame video.
- Still frame (the control): one frame only, so no motion is visible. It was trained with adapter A on single frames, to show what the adapter can do without any motion.

**Two adapters.**

- Adapter A is a soft prompt: 16 learned vectors placed between the picture and the question. That is 16,384 trained numbers, and it is the only trained part.
- Adapter B is the same soft prompt plus a small set of extra weights inside the attention layers (a LoRA of rank 8). That is 2,031,616 trained numbers.

The question text is the same as in PERC-VID-1: which way does the shape move?

**Scoring.** The model reads the picture and the question. Each of the five answers is then scored by how likely the model finds it. The most likely answer is the pick.

**Training and choices.** There were 2,000 generated training clips, 400 per answer. The learning rate and the number of passes were chosen on a separate set of 500 development clips, from a small grid, with at most three passes over the training clips. One random seed was used.

**Test sets.** Each has 500 clips.

- Main test: clips like the training clips.
- New shapes: a diamond, a star, a cross and a ring, in new colours.
- New speeds: about 193 clips at 4 pixels per frame, which is slower than any training clip; about 207 at 30 to 34 pixels per frame, which is faster than any training clip; and 100 still clips. The split between slow and fast is random, so it is not exactly half and half.
- New backgrounds: a smooth colour gradient, or a coarse random texture.
- New words: the main-test clips, with the answers worded differently: "to the left", "to the right", "upwards", "downwards" and "standing still". Training used "left", "right", "up", "down" and "it does not move".

A second model, the 2B sibling Qwen3.5-2B, was planned with the same recipe. Its training did not run (section 6).

## 3. The rules, written before the runs

- **MOTION-UNLOCKED** for a picture form if the main test is at least 0.90, new shapes at least 0.80, new speeds at least 0.70, new backgrounds at least 0.70 and new words at least 0.70, and if the still-frame control is at most 0.40 on the main test. The control checks that the adapter reads motion and not just where a shape sits.
- **MOTION-IN-DISTRIBUTION-ONLY** if the main test is at least 0.90 but one of the other conditions fails.
- **MOTION-NOT-UNLOCKED** otherwise.

The verdict words compare the point accuracy with each bar. The rules also asked for a 95 percent Wilson interval beside each accuracy. We show the intervals, but the verdicts do not use them. No addendum was written to these rules. The 2B model was to be trained only if the 0.8B showed a gain and the 2B fitted the 6 GiB memory guard.

## 4. Results

### 4.1 Main results

Accuracy on 500 clips per set, with the 95 percent Wilson interval in brackets. Chance is 0.20. Bars are in the first column.

| Test set (bar) | Trail, adapter A | Trail, adapter B | Two-frame video, adapter A | Two-frame video, adapter B |
|---|---|---|---|---|
| Main test (0.90) | 0.986 (0.9714 to 0.9932) | 0.994 (0.9825 to 0.9980) | 0.882 (0.8508 to 0.9074) | 0.928 (0.9019 to 0.9475) |
| New shapes (0.80) | 0.892 (0.8617 to 0.9163) | 0.996 (0.9855 to 0.9989) | 0.712 (0.6708 to 0.7500) | 0.866 (0.8333 to 0.8931) |
| New speeds (0.70) | 0.764 (0.7249 to 0.7991) | 0.716 (0.6749 to 0.7538) | 0.696 (0.6543 to 0.7347) | 0.698 (0.6564 to 0.7366) |
| New backgrounds (0.70) | 0.898 (0.8684 to 0.9216) | 0.940 (0.9156 to 0.9577) | 0.770 (0.7311 to 0.8047) | 0.862 (0.8290 to 0.8895) |
| New words (0.70) | 0.706 (0.6646 to 0.7442) | 0.750 (0.7102 to 0.7860) | 0.500 (0.4563 to 0.5437) | 0.614 (0.5706 to 0.6556) |
| Verdict | MOTION-UNLOCKED | MOTION-UNLOCKED | MOTION-NOT-UNLOCKED | MOTION-IN-DISTRIBUTION-ONLY |

The misses, stated plainly. Two-frame video adapter A missed four of the five bars: main test (0.882), new shapes (0.712), new speeds (0.696) and new words (0.500). Two-frame video adapter B missed two: new speeds at 0.698, one clip short (349 of 500 against the 350 needed), and new words at 0.614. Neither trail adapter missed a bar. Trail adapter A passed the new-words bar by three clips (353 of 500).

In five cells the Wilson interval reaches the bar (COMPUTED from the stored intervals): trail adapter A on new words, trail adapter B on new speeds, two-frame video adapter A on the main test and on new speeds, and two-frame video adapter B on new speeds. These results are close to their bars and should be read that way.

### 4.2 By speed

The main test has clips at 8 to 16 and at 17 to 24 pixels per frame. The speed test has clips at 4 pixels, at 30 to 34 pixels, and still clips. Accuracy by group, as stored in the result files.

| Adapter | Main test, 8 to 16 px | Main test, 17 to 24 px | Speed test, 4 px | Speed test, 30 to 34 px | Speed test, still |
|---|---|---|---|---|---|
| Trail, A | 0.9802 | 1.0000 | 0.4197 (81 of 193) | 0.9952 | 0.9500 (95 of 100) |
| Trail, B | 0.9851 | 1.0000 | 0.2642 (51 of 193) | 1.0000 | 1.0000 (100 of 100) |
| Two-frame video, A | 0.7673 | 0.9495 | 0.2280 (44 of 193) | 0.9855 | 1.0000 (100 of 100) |
| Two-frame video, B | 0.8317 | 0.9949 | 0.2280 (44 of 193) | 0.9903 | 1.0000 (100 of 100) |

On the main test, both trail adapters are near the ceiling at every training speed (0.9802 to 1.0000). Two-frame video is lower at 8 to 16 pixels (0.7673 for A, 0.8317 for B). At 4 pixels per frame, all four adapters are low (0.2280 to 0.4197). At 30 to 34 pixels, all four are at 0.9855 or higher.

### 4.3 Answer words

Accuracy on the new-words test, by answer. About 100 of the 500 clips belong to each answer.

| Answer, in new words | Trail, A | Trail, B | Two-frame video, A | Two-frame video, B | Still-frame control |
|---|---|---|---|---|---|
| "to the left" (LEFT) | 0.56 | 0.77 | 0.25 | 0.51 | 0.00 |
| "to the right" (RIGHT) | 0.93 | 0.98 | 0.35 | 0.62 | 0.00 |
| "upwards" (UP) | 0.99 | 1.00 | 0.98 | 0.97 | 0.93 |
| "downwards" (DOWN) | 0.98 | 1.00 | 0.92 | 0.97 | 0.54 |
| "standing still" (STATIC) | 0.07 | 0.00 | 0.00 | 0.00 | 0.00 |

Trail adapter A picked "to the right" 163 times and trail adapter B 158 times, where about 100 would be expected out of 500. The still-frame control picked "upwards" 363 times out of 500.

### 4.4 Control and zero-shot baselines

Still-frame control, adapter A, main test: 0.288 (0.2500 to 0.3292). That is under the 0.40 limit and above the 0.20 chance level. On the other test sets it scored between 0.222 and 0.294. It never answered "standing still" (0 of 500 in every set).

Zero-shot runs, with no adapter, on the same main-test clips:

| Picture form | Qwen3.5-0.8B | Qwen3.5-2B |
|---|---|---|
| Trail picture | 0.314 (0.2749 to 0.3560) | 0.312 (0.2730 to 0.3539) |
| Two-frame video | 0.240 (0.2046 to 0.2793) | 0.254 (0.2178 to 0.2939) |
| Still frame | 0.234 (0.1990 to 0.2731) | 0.254 (0.2178 to 0.2939) |

Across all test sets, the zero-shot numbers fell between 0.20 and 0.34. The 0.8B zero-shot runs used mixed precision with 32-bit weights, and the 2B zero-shot runs used 16-bit weights (section 6).

### 4.5 What the numbers suggest (GUESS)

Our reading, which was not tested further: in these runs a 16,384-number soft prompt was enough for a frozen 0.8B model to name direction from the trail picture, for shapes moving from 8 to 34 pixels per frame. The trail picture gave more usable direction than two plain frames. But the two forms did not carry the same amount of motion (section 7), so this is not a clean verdict on video against trails.

## 5. The weak points

- **Slow motion.** At 4 pixels per frame, which is below the training range, trail adapter A got 81 of 193 right (0.4197) and trail adapter B 51 of 193 (0.2642). Both video adapters got 44 of 193 (0.2280). Trail adapter A still passes the speed bar at 0.764, because the slow clips are averaged with the fast ones. The average hides the weak group.
- **Standing still.** On the new word "standing still", trail adapter A was right 7 times in 100 and trail adapter B 0 times in 100. Both video adapters were right 0 times in 100. The new-words passes for the trail adapters come from the four moving words.
- **Trails at slow speed.** At 4 pixels per frame the faded copies overlap so much that the trail is hard to see. This may be part of the slow-motion failure (GUESS; see section 7).

## 6. Deviations from the plan

- **Precision (the main one).** The first adapter attempt used 32-bit frozen weights with mixed precision, and it stopped at the 6 GiB memory guard at 6.71 GiB. All later adapter training and testing on the 0.8B used 16-bit frozen weights, with 32-bit adapter weights. A check of the option scores on four clips, using the plain prompt with no adapter, gave a largest difference of 0.0472 nats from the 32-bit result. The same check on the 2B gave 0.2149. The 0.5 nat limit that the check was judged against was set by us, not by the rules, and the check does not test the adapter path (section 7).
- **The 2B model was not trained.** Its first training step reached 6.40 GiB against the 6 GiB guard, with 16-bit weights and micro-batches of four. The rules said to stop there. Its zero-shot numbers were measured.
- **A checkpoint-naming incident.** For two-frame video adapter A at its chosen learning rate, the first attempt saved a checkpoint at step 125 under a name that the end-of-epoch save then tried to reuse. The second save was refused, so nothing was overwritten. That run stopped before any development or test run, and the resumed run started from the step-125 checkpoint. The audit found no sign that this changed any number. A clean repeat was not run.
- **Script revisions.** The training script was revised during the work, and the clip generator was revised once (version 2). According to our notes, the generator change touched only a self-check for the star shape. The rules, the generator and the final trainer and runner were hash-logged before the runs that used them, and the audit found that the current files match those logged hashes.
- **Mixed precision in the baselines.** The 0.8B zero-shot runs used 32-bit weights with mixed precision, and the 2B zero-shot runs used 16-bit weights. The two zero-shot baselines are not in the same setting, so they should be compared with care.

## 7. The independent audit and its three caveats

After the results, a separate AI agent reviewed the work, read-only. It read the rules, the scripts, the logs and the result files, and it recomputed each verdict from the saved numbers. It found the rule arithmetic right in every case. For each reading, the audit's status was "confirmed with caveats" for trail adapter A, trail adapter B, two-frame video adapter A and the still-frame control. "Confirmed with caveats" means the rule gives that verdict, but the headline reads more into it than the numbers show. Two-frame video adapter B was "confirmed". The 2B model was not trained, as the rules required.

The audit's three top caveats:

1. **The speed bar averages two different results.** At 4 pixels per frame the trail adapters score 0.4197 and 0.2642. At 30 to 34 pixels they score 0.9952 and 1.0000. The overall speed number is the one that passes. The honest statement is that the trail adapters work from 8 pixels per frame up to 34, the fastest tested, and fail at 4.
2. **"Standing still" is not recognised in new words.** It scored 0 to 7 percent, so the new-words passes come from the four moving words.
3. **Every adapter number uses 16-bit frozen weights,** and the check that accepted this covers only the plain prompt without the adapter. The audit's guess is that most answers would not change in 32-bit, but that verdicts near a bar could. That has not been checked.

Other findings the audit raised:

- **Brightness.** The trail picture makes a moving shape's newest copy fainter, to about 53 percent strength. On 1,000 sampled training clips (400 still, 600 moving), one brightness number separated still from moving shapes with no mistakes, at a threshold of 0.49. The same number labels about 81 percent of the 4-pixel test clips as still. The still-frame control removes the trail and this brightness difference together, so the run cannot show that the trail alone carries direction. The audit's guess is that the slow-clip failures and the "does not move" answers share this cause. Adapter B answered "does not move" for 119 of the 193 slow clips. Adapter A gave that answer 39 times on moving clips in all, nearly all of them at 4 pixels.
- **Adapter B is not a clean second test.** B started from A's soft prompt and was trained for more passes. The total passes over the training clips were 2 for trail A and 3 for trail B (2 inherited, plus 1), and 3 for video A and 5 for video B (3 inherited, plus 2). B's control is A's control, because no control was trained for B. So the gain of B over A cannot be split between the extra weights and the extra training.
- **The two motion forms do not carry the same motion.** The trail holds three steps of motion (frames 0 to 3). The two-frame video holds one (frames 2 to 3). At 8 to 16 pixels, video adapter A scored 0.7673 and trail adapter A 0.9802. A fair comparison needs the same amount of motion in both forms.
- **Video adapter A was still improving at the cap.** At its chosen learning rate, its development accuracy after passes one, two and three was 0.778, 0.852 and 0.878. Every chosen learning rate sat at the top of its grid, and larger values were never tried. So "not within three passes" is the honest reading, not "cannot".
- **The word answers depend on wording.** The score adds up the scores of word pieces, with no correction for length. The trained answers were 1 to 4 pieces long, and the new answers 1 to 3, so the word test partly measures how an answer is written.

## 8. What we could not conclude, and caveats

- **One kind of motion.** Synthetic shapes on plain or simple backgrounds, five answers, one speed range, one budget.
- **Thin margins and one seed.** One training seed, 500 clips per test set, and several verdicts close to their bars (section 4.1). No per-clip answers were saved, so paired comparisons between runs cannot be made from these files.
- **Development choices are close.** On the development set, the trail-picture settings near the top scored between 98.0 and 99.6 percent. The choice among them was close to noise.
- **Scoring.** Each answer is scored by its raw log-probability. The earlier PERC-VID-1 test subtracted a text-only score for each answer. The rule says "as in PERC-VID-1" and can be read either way. The effect of the difference is not known.
- **Checks that passed.** The audit found no overlap between training, development and test clips. The sets come from separate random streams, no shape, colour, size, start, speed or class design repeats across sets, and new colours are at least 83.8 units (RGB distance) from the training colours. The learning rate and passes were chosen on development data only, each test set was scored once, and the adapter files were hash-logged before testing. The completed 0.8B runs peaked at about 3.6 to 3.8 GiB, under the 6 GiB guard.
- **AI-assisted.** Done with an AI assistant, as the README describes. The audit was done by another AI agent, read-only. No human has reviewed these numbers.

## Corrections made after the audit

- An internal status line said "it is the trail that carries it". It reported only trail adapter A, and it left out trail adapter B's reading and the standing-still failure. After the audit it was updated to carry the three caveats above. This document does not say that the trail alone carries the result.
- The speed result is stated by speed group, not as one average.
- The standing-still failure, the 4-pixel failure and the 16-bit-weight caveat are stated here next to the passes.

## What it does not show

- Real moving footage or real film. All shapes were drawn by code on plain or simple backgrounds.
- Camera motion, several objects, textured or real objects, or other frame rates.
- Speeds between 4 and 8 pixels per frame, or between 24 and 30.
- Sound. No audio was involved.
- That the model's other answers stay the same once an adapter is in place. Only motion questions were tested.
- That our live watcher works. There was no test on its real frames, timing or pipeline.
- Anything about the 2B model, which was not trained.
- That the result holds with other training seeds.
- That the model understands "standing still".
- That the trail alone carries direction, separate from brightness.
- Open description. Each clip was scored on five fixed answers.

## What comes next

Nothing in this section has been registered or run yet.

1. **Repairs the audit asked for, before any stronger claim.** Rerun the trail and video test sets with 32-bit frozen weights, and count the answers that change. Test a trail picture whose newest copy keeps full strength, and a still-frame control that is allowed to answer "does not move", to separate the trail from brightness. Train the soft prompt alone for the same passes as adapter B. Build a two-frame trail, so the two forms carry the same motion. Use three seeds, save the answers for each clip, state each speed group against its own bar, and use answer words of equal length.
2. **A registered test of trails inside our live watcher,** on real moving pictures from the stream, using picture content only and no extra tokens. Its rules are not written yet. Its purpose is to check whether the result holds outside generated shapes.
3. **A 2B training run is low priority.** It would be worth doing only if a larger eye is needed for motion, and it would need a smaller batch to fit the memory guard.

## Label legend

- **FACT**: measured by our runs, with setting and sample size where we have them. Intervals are 95 percent Wilson intervals over clips, as stored in the result files.
- **COMPUTED**: arithmetic on measured numbers.
- **PUBLISHED**: from a named published source. This document uses none.
- **GUESS**: our reasoning, or anything beyond the tests.
- Verdict words (MOTION-UNLOCKED, MOTION-NOT-UNLOCKED, MOTION-IN-DISTRIBUTION-ONLY) are the written rules' own labels, and "confirmed with caveats" is the audit's label. A missed bar is reported as a miss.

Where a claim could take two labels, we use the more cautious one.
