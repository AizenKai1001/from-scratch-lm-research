# 30. A frozen ear, a frozen language model, and a 10-million-parameter adapter between them

## What this is

Snapshot dated 7 November 2026. We wanted the small language model of our watcher pipeline (Qwen3.5-0.8B, frozen) to hear speech without training either the audio encoder or the language model. The only trained part is a small adapter that turns the audio encoder's features into inputs the language model can read. This document records the recipe, the runs that failed on the way, and the result on a held-out test.

The rules had fixed bars and were registered (a checksum and a time) before the scripts they governed ran. They were not frozen: seven addenda were written during the work, several after a result (listed in section 2), and one of them let a run continue past a registered stop. Treat every result as an indication, not a proof, and misses are reported as plainly as passes.

The training targets are human-written text: the transcripts of LibriSpeech (**PUBLISHED**: audiobook recordings with transcripts, licence CC BY 4.0), which are the book text read aloud. The corpus builders cut the recordings and matched them to the text with an automatic alignment procedure (**PUBLISHED**, LibriSpeech paper), so a machine step touched the segmentation, not the words. The fixed input prompts ("Transcribe the audio.", "Transcript:") were written by us with an AI assistant and sit in every input, not in the targets. No language model wrote, translated or relabelled any target text.

## Summary

| Question | Verdict, as the rules read it | Main number |
|---|---|---|
| Can a 10-million-parameter adapter between a frozen Whisper-small encoder and a frozen 0.8B language model transcribe read speech? | Gate B PASS (one reading on a held-out test) | Word error rate 5.54 percent on 2,032 clean test utterances of 3 to 16 seconds (bar: at most 12) |
| Does it use the audio, or just guess likely text? | It uses the audio, by the wrong-audio control | With another clip's audio the output scores 118.8 percent error against the true text (bar: at least 90) |
| Does it stay quiet on generated non-speech? | Yes, on the families we tested | Zero words on every one of 100 clips in each of nine generated families, including two never seen in training (mains hum, chirp); 0.000 words per minute over 21 generated clips (bar: at most 2) |
| Does it get stuck in loops? | No | Loop rate 0.049 percent, one clip of 2,032 (bar: below 5) |

## 1. The recipe (FACT)

- **Ear:** the encoder of openai/whisper-small, frozen, run in half precision (a check on 10 clips gave a relative difference of at most 0.32 percent against full precision). Its frames (50 per second) are stacked 5 at a time, so the language model receives 10 audio vectors per second.
- **Adapter ("projector"):** a linear layer (3,840 to 2,048 numbers), a GELU, a linear layer (2,048 to 1,024) and a layer norm, 9,966,592 trainable parameters.
- **Brain:** Qwen3.5-0.8B, frozen, in half precision. The input is the fixed text "Transcribe the audio.\nAudio:", the audio vectors, a fixed suffix, then the transcript; the loss counts only the transcript tokens and the end token.
- **Data:** LibriSpeech train-clean-100 (28,539 utterances, 100.6 hours; the 3 to 16 second band has 26,165 clips, 92.1 hours), dev-clean (2,703 utterances) and test-clean. Plus clips of **generated non-speech** made by code (silence, white, pink and brown noise, a tone, a click train, low-pass noise) with an empty transcript, so the target is the end token only. In the final recipe 8 percent of training slots were such clips, and their end-token loss carried a weight of 38 (the mean number of transcript tokens of a speech clip). Mains hum and a chirp sweep were kept out of training for testing. One seed (20261107) fixed every random draw.

## 2. The road to a passing run (FACT)

1. **First pilot (2,834 clips, 10 hours).** It stopped at its registered alignment check: 112 percent word error on 50 development clips at step 2,000 (the rule says stop at 50 percent or more). A separate check that trains on 50 clips and compares true with shuffled audio passed (the true-audio loss fell to 0.02 while the shuffled-audio loss rose to 7.4), so the audio path carries clip-specific information into the language model; because those 50 clips are memorised, it does not show generalisation. Our guess, untested: the pilot had not aligned yet.
2. **Crashes and memory stops.** A full-precision language model crashed with out-of-memory (7.27 GiB allocated on a 7.6 GiB card; the 6.0 GiB guard never fired). The first attempt of the next run crashed on a gradient-scaler assertion (a bug, fixed). A half-precision run with a micro-batch of 4 passed its memory pre-test and went over the 6.0 GiB guard at step 265. A micro-batch of 2 with two accumulation steps (same update) fixed it. The batch size changed only after a written addendum.
3. **Run R2 (half precision, peak learning rate 1e-3).** A long plateau at a loss near 4.2, then a sudden drop: development word error 124 percent at step 1,000, 9.4 percent at 2,000, 7.5 percent at 3,000. Gate A on 300 development clips: word error 8.03 percent, loops 0, wrong audio 119.5 percent, but **silence failed**: the model produced 14.7 "words" per clip from all-zero audio (the prompt label "Transcript:" repeated, counted by whitespace) and no clip stayed at 3 words or fewer. The loss scaler skipped 11 steps (overflow guard, not NaN).
4. **Run R4 (same recipe plus the generated non-speech clips from the start).** It ran past its registered stop at step 2,000 (128 percent word error): an addendum written after that result let it continue. It failed the alignment check at step 4,000 (111 percent). Our guess, untested: the heavily weighted non-speech slots compete with speech alignment early.
5. **Run R5 (start from the aligned R2 adapter, add the non-speech clips, learning rate 3e-4, warm-up 100 steps, 1,418 steps = 2 epochs).** Passed the repeated Gate A, which we call Gate A-prime because its word-error bar (10.03 percent) was set after the R2 result of 8.03: word error 8.31 percent, loops 0, wrong audio 119.4 percent, at most 3 words on all nine non-speech families including the two never seen in training. Skipped scaler steps: 12.
6. **Full run.** All 26,165 clips of the band, 3 epochs from the R5 adapter, about 4.3 hours on an RTX 2060 SUPER (a 0.81 hour feature cache plus training; 17 skipped scaler steps). The first-epoch check on 300 development clips gave 5.94 percent (limit 15). Development loss by epoch: 0.222, 0.216, 0.227. The weighted training loss (the non-speech end-token weight is included, so it is not comparable with the development loss) fell 0.19, 0.10, 0.05. Our reading (GUESS): the adapter starts to fit the training clips in epochs 2 and 3.

**Rule history.** Addenda to the rules were written after results at about 07:30, 07:51, 08:02, 08:53, 08:56, 09:20 and 10:02 on the day, each before the run it governs; the 09:20 addendum let R4 run past its step-2,000 stop. The development clips chose R2 and R2 is the start of R5 and of the full run, and the R5 bar was set from R2's result. Only the test-clean number below was not tuned on.

## 3. The held-out test, read once (FACT)

The final adapter was saved with its checksum before the test. Test-clean, all 2,032 utterances of 3 to 16 seconds (4.01 hours), decoded greedily. The single reading was decoded in two sessions with a checkpoint (1,492 utterances, then the remaining 540) and never re-read.

| Reading | Result | Bar |
|---|---|---|
| Word error rate (corpus, 39,824 reference words) | 5.54 percent | at most 12 |
| Loop rate (a 3-word sequence repeated 4 or more times) | 0.049 percent | below 5 |
| Wrong audio (100 clips scored against another clip's text) | 118.8 percent error | at least 90 |
| Generated non-speech, share of 100 clips per family with at most 3 words | 1.0 in each of 9 families (0 words on every clip) | at least 0.90 |
| Generated clips, words per minute (21 clips: 7 training families at 3 levels, silence included) | 0.000 | at most 2 |

**What it means (GUESS).** A small adapter on two frozen models transcribes clean read English at about 5 to 6 percent word error, uses the audio, and stays quiet on the generated sounds we tried, including two it never saw. The result came from one run.

## What we could not conclude, and caveats

- **One kind of speech.** Audiobook readers, clean recordings, 3 to 16 seconds. We did not test noisy, overlapping or accented speech, music with words, other languages, or utterances shorter or longer than the band. The non-speech result covers generated signals only: real room noise, music and quiet speech are untested.
- **No baseline.** We did not run Whisper's own decoder on the same clips, so we cannot say how this adapter compares with it.
- **Choices came from development data.** See the rule history in section 2. The silence and noise families were seen in training; only mains hum and chirp were not.
- **Controls are not random.** The wrong-audio control and the first 100 clips of each control use the first 100 test-clean clips in file order, which are speaker-clustered. We did not check test-clean against the training speakers ourselves (the corpus design separates them).
- **One adapter, one run.** One seed, no repeat.
- **Code review.** A read-only AI-agent review of the run's code (before the test) found no error that changes a result. It listed deviations: no temperature rule inside two loops, a numpy generator for the positions of non-speech clips, a half-precision output layer where the rules said full precision, a smaller memory pre-test, the first three non-speech steps not logged, a thin 240-second margin before the wrapper's time limit at the epoch-1 check, a time cap checked per session, and one defect that would only matter after a failed test (a re-run would print PASS by mistake; ours passed and we read the verdict from the result file).
- **Overfitting.** The development loss rose slightly in the third epoch; fewer epochs or a lower learning rate would probably do as well (GUESS, not tested).
- **AI-assisted, not peer reviewed.** Done with an AI assistant, as the README describes. A separate AI-agent check of every number in this document against the logs was made; no human has reviewed them.

## Label legend

- **FACT**: measured by our runs, with setting and sample size where we have them.
- **COMPUTED**: arithmetic on measured numbers.
- **PUBLISHED**: from a named published source or dataset description.
- **GUESS**: our reasoning, or anything beyond the tests.
- Verdict words (Gate A, Gate B, PASS) are the written rules' own labels. A missed bar is reported as a miss.

Where a claim could take two labels, we use the more cautious one.
