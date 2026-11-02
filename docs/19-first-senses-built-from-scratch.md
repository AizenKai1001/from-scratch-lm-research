# 19. First senses, built from scratch

Snapshot dated 2 November 2026. Document 16 lists a body for the model: eyes (vision), ears (speech recognition), a voice (speech synthesis) and an imagination (a small image generator). The project's rule is that each of these is built from scratch, the same way the language model was, with no pretrained weights in the finished part. This document reports the first two steps: a tiny image model, and a bridge that lets the language model read what it sees. Labels: **FACT** is measured in our own logged runs; **GUESS** is reasoning not yet tested. Each test had its pass and kill conditions written before it ran, and the verdicts apply them as written.

## The design in one paragraph

Each sense is its own small model, trained from scratch on open data, that turns raw input (a picture, a sound) into a few vectors. A small learned "bridge" turns those vectors into the same kind of vectors the language model uses for words, and they are placed at reserved positions in the model's input. The language model itself is not changed. The point of keeping the senses separate at first is that each can be tested alone, and that a new sense cannot disturb the others while we still lack a way to keep it from doing so (document 18 is the first step toward that).

## 1. A tiny image model (the eyes), two seeds

**FACT.** A 2.24-million-parameter image model (a small convolutional front end feeding a six-layer transformer, with no pretrained weights) was trained for 60 epochs on the 50,000 training pictures of CIFAR-10 (ten classes of small photographs, open data), with random crops and flips, in about 10 minutes on a small consumer GPU. The 10,000 test pictures were used only for the final number.

| Seed | Test accuracy | Training time |
|---|---|---|
| 1 | 90.4% | 627 s |
| 2 | 90.1% | 624 s |

The pass line, written first, was 90 percent in under an hour. Both seeds pass, with a margin of only 0.1 to 0.4 points, so the honest summary is that this model sits just around the line. It is a first version of an image model, far from the best published results for this data set.

## 2. The bridge: the language model reads the picture

**FACT.** With the image model and the language model both frozen, a small network (two layers) was trained for about 5 minutes to turn the image model's output into four vectors that stand in place of four word embeddings at the start of a prompt. The prompt was those four vectors followed by the words "It is a", and the answer was one of ten words (airplane, automobile, bird, cat, deer, dog, frog, horse, ship, truck). For each of the 10,000 unseen test pictures we picked the answer word to which the language model gave the highest probability.

- **Accuracy: 90.5 percent**, against a written pass line of 85 percent. The image model on its own scores 90.4 percent, so the language model reads the picture about as well as the image model's own classifier does.
- **The language model's text behaviour was unchanged.** Before any training, we checked that with the bridge installed and nothing placed in the reserved positions, the model's outputs on a real text batch were exactly the same as before (largest difference 0.0, text loss identical to six digits), and it was still identical after the bridge was trained. This was a condition of the test: if it had failed, the run would have counted as void.
- **Limits.** The answer is a choice among ten single words, not a free description. The bridge was trained with the labels. One seed. The language model cannot yet describe a picture in its own words.

## 3. The ears: in progress

A small speech recogniser (about 7 million parameters, built the same way) is being trained from scratch on 100 hours of open, read English speech, with its pass line (a word error rate of 40 percent or lower on a held-out test set that is looked at only once, at the end) written down first. As this snapshot is written the run is not finished, and no result is reported here.

## 4. What is next, and the open design question

- **GUESS.** The senses will be built in this order: ears and a voice together (the voice can be checked by the ears reading back what the voice said, so no human listening is needed), then the eyes in their next versions, then the imagination, which could reuse the eyes' encoder.
- **Open question: separate or combined.** We plan to build the senses separately first, then test two ways of combining them against the separate version at the same total size: one shared vocabulary of discrete tokens for every sense, and a single model with specialist parts per sense that a router chooses between. Separate first keeps each test clean and keeps interference measurable. This is a plan, not a result.
- **Open question: learning new senses without damage.** Keeping the language model frozen (as in section 2) protects its skills trivially. The harder, more useful case, where the language model changes as it learns a new sense, will need the learn-then-heal method of document 18, and it is untested.
