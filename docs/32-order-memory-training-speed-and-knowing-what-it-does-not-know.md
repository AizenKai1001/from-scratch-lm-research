# 32. Order memory in a bigger model, a faster training step, and knowing what it does not know

## What this is

Snapshot dated 9 November 2026. This document reports four sets of tests on the small language models used in the earlier documents (Qwen3.5 0.8B and 2B, weights frozen unless stated). All test data was made by code or taken from a public human-written quiz. No film, no downloaded footage and no model-written text was used for training or testing.

Every test had a written rule with fixed bars, registered with a checksum and a time before the script it governed ran. Each result was then re-checked by a separate AI agent that only reads (the "audit"). Treat every number as an indication, not a proof. Misses and mistakes of our own are listed in the last section.

## In short

- A bigger model (2B) keeps the order of events better than the small one (0.8B) on the same code-made streams. A control showed that numeric precision does not explain the gap. Part of the answers is still a habit of picking the first event shown.
- The order of events is stored in a small group of the model's slow memory heads and can be read out with a simple linear readout. The readout works for new event pairs, three events and longer streams, but only partly for event wording it never saw.
- Compiling the training step made a small adapter-training step about 1.8 times faster with the same learning. Recording the whole step as a CUDA graph did not help.
- Whether the models know when their answer is wrong: the model's own word probabilities are a weak sign. A readout of the hidden state is clearly better, but a large part of its signal is already in the question wording alone, so the extra is small.

## Summary

| Question | Verdict, as the rules read it | Main number |
|---|---|---|
| Does the 2B beat the 0.8B on "which event came first" for pairs that do not include the first event? | SIZE-HELPS on the main run; SIZE-CONFIRMED after the precision control | +0.20, +0.18, +0.16 on three sets of 100 streams; each 95 percent interval above zero |
| Is that gap caused by running the 0.8B in a different precision? | No: SIZE-CONFIRMED | 0.8B in 16 bits minus 0.8B in 32 bits: +0.013, +0.004, +0.006; 2B minus 16-bit 0.8B: +0.186, +0.175, +0.157 |
| Can a readout of the 35 slowest memory heads tell which of two events came first? | Positive, with caveats | 0.79 on held-out streams (guess is 0.50; random head sets 0.575; scrambled-answer control 0.48) |
| Does one shared readout, told which two events are meant, work on wording it never saw? | QCO-PARTIAL | 0.64 (0.8B) and 0.62 (2B) on new wording, against 0.80 and 0.83 on trained wording |
| How much faster can a small adapter-training step be made? | TRAIN-COMPILE-WINS | 281 ms to 154 ms per step (0.55 of the time) with the loss and weights unchanged |
| Do the models' word probabilities show when an answer is wrong? | CONFIDENCE-WEAK | pooled AUROC 0.57 to 0.64 (0.8B) and 0.52 to 0.69 (2B) over 6,000 questions (0.50 is a coin flip) |
| Does the hidden state show it better? | HIDDEN-BETTER, then STATE-BEYOND-TEXT | 0.81 (0.8B) and 0.87 (2B) against 0.64 and 0.69 for the word probabilities; the question wording alone scores 0.71 and 0.67 |

## 1. Order memory: bigger model, same streams

A stream is a series of code-made events (coloured shapes appearing in order). After the stream the model is asked, for example, which of two events came first, or whether a given event occurred at all. The test measures what the model's own state keeps, with no external memory.

On three sets of 100 streams, the 2B beat the 0.8B on order for pairs without the first event by 0.20, 0.18 and 0.16, all with 95 percent intervals above zero. The 2B also said "yes" to events that never occurred only 4 to 5 percent of the time, against 50 to 80 percent for the 0.8B, so part of its gain on "did it occur" is a change of bias and not better memory.

The first run had the 2B in 16 bits and the 0.8B in 32 bits, so we ran a control: the 0.8B in the same 16 bits on the same card. It scored within 0.013 of the 32-bit run on every set, so precision does not explain the gap. The gap is the same size at every spacing between events, which points to a better read-out of the order and not a longer memory.

Caveats from the audits: the comparison is between two different models, not size alone; part of the answers is still the first-event habit; a smaller secondary set of very long streams hit our own memory limit four times and is not used for the verdict; and the list of sets used for the verdict was changed once without registering it (the verdict held either way).

## 2. Where the order lives

In the 0.8B, 288 of the recurrent memory heads were checked. A plain linear readout of the 35 slowest ones (the ones that keep a trace for 128 or more tokens) told which of two events came first 0.79 of the time on held-out streams. Random sets of 35 heads averaged 0.575 (range 0.49 to 0.71), the 35 fastest 0.57 and all 288 together 0.64. A control with scrambled answers scored 0.48. Single heads in two other blocks reached 0.96 to 1.00, but they were picked after the test scores were seen, so those numbers are optimistic.

A second test chose the heads on training streams only and used new event pairs. It scored 0.79 to 0.83 on new pairs, 0.77 on three events (all three in the right order 0.475 of the time, chance 0.17) and 0.87 on streams of 400 tokens. The 2B scored 0.89 to 0.90 on new pairs. A readout trained on one pair scored at chance on another pair in the 0.8B (0.60 in the 2B), so each event pair needed its own readout. The audit found that the three-event and longer-stream tests reused the sentences the readout was trained on.

A third test asked whether one shared readout, told which two events are meant, works on wording it never saw. It scored 0.64 (0.8B) and 0.62 (2B), the 0.8B missing its 0.65 line by 0.008. An in-stream version that scored 0.95 turned out to be an artefact: it scored 0.94 even with another stream's memory. A later, after-the-fact hint (0.8B only, not a registered result) is that calibrating each event against its own reference gave 0.85 on new wording. None of this shows that the model itself uses the order information.

## 3. A faster training step

For a small adapter on the 0.8B on an 8 GB card, one training step took 281 ms. The GPU was busy 92 percent of the time, so it was limited by its own work and not by launching kernels. Compiling the step cut it to 154 ms (1.83 times as many examples per second). Loss differed by at most 0.0004 per step and the adapter weights by 0.0012, and three deliberately wrong runs (doubled learning rate, shuffled batches, a stale batch) were all caught. Recording the whole step as a CUDA graph worked but gave no gain (284.5 ms), unlike decoding, where it helped a lot. The audit relabelled the result: the winner is the compile step, not the graph. The baseline used a batch of 2 without mixed precision, not the batch of 8 in the written rule.

## 4. Does the model know when it is wrong?

The questions come from the TriviaQA validation set (human-written). The model answers, and we ask whether its confidence separates right answers from wrong ones. The score is AUROC: 0.50 is a coin flip, 1.0 is perfect.

- The model's own word probabilities were a weak sign: pooled over 6,000 questions, 0.57 to 0.64 for the 0.8B and 0.52 to 0.69 for the 2B. The usable bar (0.70) was not reached.
- A simple linear readout of the hidden state, trained on 3,000 questions and tested on 3,000 different ones, scored 0.806 (0.8B) and 0.865 (2B). The state read before the model starts to answer already scored 0.75 to 0.76.
- A control that sees only the question text (word counts) scored 0.706 and 0.670. So the hidden state adds +0.057 (0.8B, a thin margin) and +0.082 (2B) over the wording. Inside question types (who, what, which) the state still scored 0.60 or more.
- Allowed claim: the hidden state holds some information about whether the answer will be right beyond the question wording. Not claimed: a working "I do not know" behaviour.

Caveats: the exact-match scoring could not be used (too few right answers), so the lenient scoring was used; TriviaQA is probably in the training data of these models; the wording baseline is simple; half of the test questions were also used in an earlier run.

## What went wrong, and what we changed

- **A test of ours broke the first calibration run.** After the prompt, the model writes a newline and an empty thinking block before its answer. Our extraction stopped at the first newline, so 60 to 87 percent of the answers were empty and scored wrong. The run was recorded as broken and left unedited. A written fix and a rerun followed.
- **A limit we guessed was too tight.** The second run was ruled broken because two samples of 600 questions differed by 0.056 and 0.058 against a guessed limit of 0.05. We then measured the spread (about 0.033 to 0.044 for the difference between two such samples) and set a new limit from it for a new, larger test. The old results stand as "broken"; the new limit moved after a failure, which we say here.
- **Samples that should not overlap did.** An audit found 434 duplicate questions shared between the two 3,000-question samples (duplicates inside the quiz file itself). Later tests removed duplicates by question id and text.
- **A shell quoting slip** left one rules file unwritten once; the engineer found no file, ran nothing, and the file was rewritten and registered before any script ran.
- **Memory limits.** The memory guard stopped four parts of the secondary long-stream set; they were recorded as halts and not retried.

## What this does not show

These tests were run on generated streams and on a public quiz, with frozen models. They do not show that the order information in the memory heads is used by the model, that the confidence signal makes a working "I do not know", anything about the 4B model, or anything about longer real-world video.
