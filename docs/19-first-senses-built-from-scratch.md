# 19. First senses, built from scratch

Snapshot dated 2 November 2026. Document 16 lists a body for the model: eyes (vision), ears (speech recognition), a voice (speech synthesis) and an imagination (a small image generator). The project's rule is that each of these is built from scratch, the same way the language model was, with no pretrained weights in the finished part. This document reports the first steps: a tiny image model, a bridge that lets the language model read what it sees, a first speech recogniser, and a first imagination that draws from the language model's own state. Labels: **FACT** is measured in our own logged runs; **GUESS** is reasoning not yet tested. Each test had its pass and kill conditions written before it ran, and the verdicts apply them as written.

## The design in one paragraph

Each sense is its own small model, trained from scratch on open data, that turns raw input (a picture, a sound) into a few vectors. A small learned "bridge" turns those vectors into the same kind of vectors the language model uses for words, and they are placed at reserved positions in the model's input. The language model itself is not changed. The point of keeping the senses separate at first is that each can be tested alone, and that a new sense cannot disturb the others while we still lack a way to keep it from doing so (document 18 is the first step toward that).

## 1. A tiny image model (the eyes), two seeds

**FACT.** A 2.24-million-parameter image model (a small convolutional front end feeding a six-layer transformer, with no pretrained weights) was trained for 60 epochs on the 50,000 training pictures of CIFAR-10 (ten classes of small photographs, open data), with random crops and flips, in about 10 minutes on a small consumer GPU. The 10,000 test pictures were used only for the final number.

| Seed | Test accuracy | Training time |
|---|---|---|
| 1 | 90.4% | 627 s |
| 2 | 90.1% | 624 s |

The pass line, written first, was 90 percent in under an hour. Both seeds pass, with a margin of only 0.1 to 0.4 points, so the honest summary is that this model sits at the line, within the roughly 0.6-point error of a 10,000-picture test. It is a first version of an image model, far from the best published results for this data set.

## 2. The bridge: the language model reads the picture

**FACT.** With the image model and the language model both frozen, a small network (two layers) was trained for about 5 minutes to turn the image model's output into four vectors that stand in place of four word embeddings at the start of a prompt. The prompt was those four vectors followed by the words "It is a", and the answer was one of ten words (airplane, automobile, bird, cat, deer, dog, frog, horse, ship, truck). For each of the 10,000 unseen test pictures we picked the answer word to which the language model gave the highest probability.

- **Accuracy: 90.5 percent**, against a written pass line of 85 percent. The image model on its own scores 90.4 percent, so the language model reads the picture about as well as the image model's own classifier does.
- **The language model's text behaviour was unchanged.** Before any training, we checked that with the bridge installed and nothing placed in the reserved positions, the model's outputs on a real text batch were exactly the same as before (largest difference 0.0, text loss identical to six digits), and it was still identical after the bridge was trained. This was a condition of the test: if it had failed, the run would have counted as void.
- **Limits.** A control with no language model (the same bridge feeding a plain classifier) was not run, so this shows that the language model can read the picture, not that it adds anything beyond the image model. The answer is a choice among ten single words, not a free description. The bridge was trained with the labels. One seed. The language model cannot yet describe a picture in its own words.

## 3. The ears

**FACT.** A small speech recogniser (7.1 million parameters, a small convolutional front end feeding eight transformer blocks, trained with a standard alignment-free loss, no pretrained weights) was trained from scratch for 40 epochs on 100 hours of open, read English speech, in about 3 hours on a small consumer GPU. The pass line, written first, was a word error rate of 40 percent or lower on a held-out test set that is looked at only once, at the end.

- **Result: word error rate 24.6 percent on the test set** (24.7 percent on the validation set used to pick the checkpoint). That passes the line. About three words in four are right (word error rate also counts inserted words).
- **Limits.** One seed. No language model is used to clean up the output, which is plain greedy decoding. The speech is clean, read audiobook speech; noisy or spontaneous speech would be much worse. It is a first version, far from the best published results.

## 3b. The imagination, first version: pictures from the brain's own thoughts

**FACT, one seed.** The aim for the imagination is that it works without being told to draw: the language model reads ordinary conversation, and what it is currently "thinking about" decides what appears. As a first test, the frozen language model read a sentence that never mentions drawing ("Yesterday I saw a cat near the old bridge."), and its inner state (1,280 numbers taken from its last layer) was the only input to a small image maker (a conditional diffusion network of 13 million parameters, no pretrained weights, 24,000 steps, about 54 minutes on a small consumer GPU, trained on the 50,000 CIFAR-10 training pictures). The sentence-to-picture link was learned from 20 sentence templates for each of the ten objects. It was judged on 8 different templates, by the frozen image model of section 1, which names what it sees; no test sentence or test picture was used in training.

| Test | Result | Written pass line |
|---|---|---|
| Held-out sentences that name the object (2,000 pictures) | **83.95% named correctly** | 70% (chance 10%) |
| Descriptions that never name the object, for example "it purrs, curls up on the sofa" (1,800 pictures) | 46.7% | none (report only) |
| Unrelated sentences (weather, homework): share of the most common judged object | 23.8% | none (should not collapse to one object) |

Verdict by the rule written first: **pass**. Cars, horses, ships and frogs are clearly recognisable in the sample grid; planes and birds are the weakest. The descriptive result is the more interesting one for the goal: with templated and hand-written descriptions, the language model's own state still carries enough meaning that a description with no object word produces the right kind of picture almost half the time. The judge itself is right only about 90 percent of the time on real pictures, which bounds what these numbers can show. Limits: 32 by 32 pictures, ten objects, templated sentences, one seed. It is a still image per sentence, not a continuous live stream, and it does not yet use reference pictures.

**Following a change of topic (FACT, one seed each).** Next the imagination was tested on invented conversations of six sentences: three about one object, then three about another, with the brain reading the whole conversation so far. The written pass line was: at least 70 percent right after sentence 3, at least 60 percent right after the first new-topic sentence, and at least 70 percent after sentence 6. The first attempts failed, and the failures are part of the result.

| Version | Sentences 1 to 3 | Sentences 4 to 6 (after the switch) | Verdict by the written rule |
|---|---|---|---|
| Trained on single sentences only | 88, 73, 78 percent | 31, 34, 34 percent | fail (24 percent of the first new-topic pictures still showed the old object) |
| Continued training on conversations with 0 to 4 earlier sentences | 92, 74, 73 | 86, 68, 56 | fail (switches at once, then fades) |
| Training conversations built as topic runs | 96, 83, 86 | 79, 74, 68 | fail by 2 points |
| Same, a little longer with longer runs | 94, 86, 86 | 84, 79, 73 | pass, but tuned on these same test conversations |
| The last version on 100 fresh conversations nothing was tuned on | 90, 91, 88 | 78, 80, 79 | **pass** |

So the clean result is the last row. Reading each sentence alone, with no earlier context, gives about 88 to 93 percent, so a gap to the single-sentence level remains. Limits: each tuning step used the same first set of test conversations, one seed, templated sentences, ten objects, 32 by 32 pictures.

**Can the imagination improve itself? (FACT, one seed, confirmed with a second judge).** The imagination drew four candidates for each of 4,000 training situations, the image model of section 1 kept the best one when it was at least 90 percent sure of the right object, and the imagination trained further on a half-and-half mix of its own pictures and real pictures. A control trained for the same number of steps on real pictures only. On new sentences the self-trained version was 92.5 percent right against 88.6 percent. The worry was that the same image model both chose the pictures and scored the result, so part of the gain could be the imagination learning to please that one judge. The experiment was repeated and every picture was also scored by a second image model trained with a different seed that never touched the loop: 92.45 percent against 89.0 percent, a gain of 3.5 points against a written line of 2. Picture variety stayed at 94 percent of the control's (the line was 85 percent), and the conversation score at sentence 6 was 71 against 69 percent. Limits: the second judge has the same design and training data as the first, so it is only a partly independent check; one seed; ten objects; 32 by 32 pictures; the gains are a few points.

## 3c. A better judge for a higher-quality imagination (FACT, in progress)

To move the imagination from 32 by 32 pictures of ten objects to 64 by 64 pictures of 100 kinds of things, two new image classifiers were first built from scratch (11 million parameters each, trained for 30 epochs, two different seeds) on a 100-class subset of the 64 by 64 version of ImageNet (research-use data, not redistributed). On the 5,000 official validation pictures of those classes, which were never used for any selection, they score 77.3 and 78.3 percent (chance is 1 percent; the written line was 60 percent). The first judge missed a written 45-minute training-time clause by about five minutes (49.6 minutes), which is reported here because it was part of the rule. These two judges, which will never be trained on generated pictures, are used to score the 64 by 64 imagination, one as the main judge and one as the hold-out. The imagination itself is still training and no result is reported yet.

## 4. What is next, and the open design question

- **GUESS.** The senses will be built in this order: ears and a voice together (the voice can be checked by the ears reading back what the voice said, so no human listening is needed), then the eyes in their next versions, then the imagination, which could reuse the eyes' encoder.
- **Open question: separate or combined.** We plan to build the senses separately first, then test two ways of combining them against the separate version at the same total size: one shared vocabulary of discrete tokens for every sense, and a single model with specialist parts per sense that a router chooses between. Separate first keeps each test clean and keeps interference measurable. This is a plan, not a result.
- **Open question: learning new senses without damage.** Keeping the language model frozen (as in section 2) protects its skills trivially. The harder, more useful case, where the language model changes as it learns a new sense, will need the learn-then-heal method of document 18, and it is untested.
