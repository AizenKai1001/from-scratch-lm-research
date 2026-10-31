# Overview, and the idea and theory behind it

## Summary

The project has two layers. The first is a **base model** trained from scratch: 143 million parameters, trained in short chained sessions on two consumer graphics cards, used as a laboratory for testing training ideas with proper controls. The second is a **reader** built on a copy of that model: a model that keeps learning while it reads, goes back over the parts it found surprising, and is checked by quizzes instead of being corrected by them. The second layer is the point of the whole exercise; the first exists so that the "brain" in the second is our own and its behaviour can be measured.

This page explains the ideas and the reasoning behind them. The results are in the other documents; each idea below says where.

## 1. Why train a small model from scratch

- **Mechanics are visible at small scale.** A 143M model can be trained to a useful point in days on one 8 GB card and inspected on a CPU. Questions such as "what does weight decay do to the weights?" or "which heads implement copying?" can be answered with many seeds and controls, which is impossible at larger scale for one person.
- **The brain must be ours.** The reader idea (below) needs a model whose training, state and behaviour we control, because the "body" around it (what it does when something surprises it) is built and tuned against that specific model. A stronger borrowed model would change the problem.
- **Small results are only hypotheses for larger scale.** We say this wherever it matters: nothing here has been shown to hold at billions of parameters.

(How the recipe differs from normal practice: document 02. The recipe itself: document 03. How the model compares with other small models: document 04.)

## 2. The research stance

Claims are written down with their setting, rules for deciding are written before the data exists, a second reader is asked to find mistakes, and the mistakes found are published (document 12). The aim is a record in which a reader can tell what is measured from what is hoped.

## 3. The ideas, and what each one predicts

### 3.1 Weight decay as a clock (documents 05 and 09)

**Idea.** With the optimizers we use, the size of the weight matrices settles where the pull of weight decay balances the push of the updates. That balance depends on the decay value, so changing the decay moves the matrices to a new size over a time that is set by the decay itself. A larger matrix makes every update a smaller relative step, so lowering the decay behaves like slowing the learning rate down, without touching the learning rate.
**What the idea implies.** When decay falls five-fold, the squared matrix scale should settle about five times higher, i.e. the scale itself about 2.24 times higher (the square root of five).
**Prediction we wrote down before the second-session snapshots were scanned.** A further growth of 1.05 to 1.13 times in the second session after the change.
**FACT.** The mean matrix scale rose by 2.02 and 1.96 times in the first session after the change in the two training chains, then by a further 1.11 and 1.13 times (both inside the predicted range). After two sessions the main chain stood at 2.24 times its starting value, which looked like the square root of five, but it did not stop there: 2.31 times after three sessions and 2.33 times after four (the second chain: 2.20 after two, 2.28 after three, still growing).
**Correction (document 12).** We first reported the 2.24 as the equilibrium; that was wrong. The plateau is about 4 percent above the square root of five (GUESS: squared ratio near 5.4, not 5), so the simple balance model gets the first-session jump and the time scale right and the final level slightly wrong.
**What it explains and what it does not (GUESS).** The loss fell sharply when the decay was lowered at constant learning rate. The working explanation is a partial learning-rate cooldown caused by the growing matrices, not better learning. We have not isolated this directly.

### 3.2 Learning while reading (documents 08 and 10)

**Idea.** Gradient steps taken on the text a model has just read should make the rest of that text easier to predict, because texts repeat their own names, topics and style. This is called dynamic evaluation in the literature (PUBLISHED: Krause and colleagues, 2019, report 5.3 percent lower loss for a 277M Transformer-XL on character-level text; Rannen-Triki and colleagues, 2024, arXiv 2403.01518, report about 1.3 percent, 2.0 percent with a reset per book, for 150M to 1B models).
**Prediction we wrote down.** A small, careful step size would give a gain well below 1 percent after a few updates (the 29 Oct pilot guess was 0 to 0.006 bits per byte).
**FACT.** The pilot gave 0.67 percent after three updates; reading whole books with a small step after every 128 tokens gave 2.4 to 4.6 percent, in every tenth of every book; updates taken from another book made the same book worse (-1.27 percent), so the gain is about the text being read.

### 3.3 Replay, because reading changes the model (document 08)

**Idea.** Always-on learning drifts the model away from what it knew. The classic remedy from the study of memory is to interleave replay of older material while new material is learned (the complementary-learning-systems view, PUBLISHED: McClelland, McNaughton and O'Reilly, 1995; we saw this paper only through a summary).
**FACT.** One real training window mixed in after every eighth segment kept the whole gain of the large step and cut the drift of the loss on ordinary web text from +1.39 percent to +0.36 percent. Pulling the weights back toward their starting values did about what a smaller step does.

### 3.4 Surprise as a reason to look again (documents 08 and 10)

**Idea.** People look back at words that surprised them. If a model's own surprise is used to pick which segments to revisit, the revisited segments should be remembered better than segments picked at random.
**FACT.** Revisiting the surprising ten percent of segments made the weights keep those segments better (7.9 percent against 6.5 percent lower loss afterwards); revisiting the same number of random segments gave 6.7 percent. It did not change how well the model predicted later text. Whether it helps answer questions about the revisited text is not yet measured.

### 3.5 Quizzes as a thermometer, not a teacher (document 08)

**Idea.** A separate check, such as cloze questions scored by which option the model finds most likely, can tell us whether the model is getting better or worse while it learns on its own, without feeding the answers back (which would interfere with its learning). If it collapses, the reader returns to the last good checkpoint and does nothing else.
**FACT.** A first version of the quiz measurement was run on two novels (document 08, section 9). The unchanged model already answers 81 to 90 percent of the items, and reading the book did not change that beyond noise, so this quiz is too easy to work as the thermometer. A harder version is the next step.

### 3.6 Brain and body

**Idea.** Instead of teaching one model to emit elaborate tool-call formats, split the system: a **brain** (the language model) that reads and decides, and a **body** (ordinary programs) that carries out small actions: look a word up, go back over a span, write a margin note, continue. The body also has **reflexes**: a rule such as "a word that is rare in our training data and surprising triggers a look-up" fires without waiting for the brain. Later the brain can learn to issue the same actions itself, trained on its own successful traces. Perception (hearing, seeing) would be further bodies that run independently of the brain.
**Status.** Version 1 uses reflexes only. We tested whether the actions help at all (learning while reading: yes; revisiting surprising spans: it strengthens memory of those spans; dictionary look-ups: data prepared, test not yet run) before building anything for the brain to control.

## 4. What this does not show

- That a reader that learns like this is a better language model overall: the gains are measured on the text being read, and the cost (a slow drift on ordinary text) is only partly controlled.
- That prediction gains mean understanding. Loss going down is not the same as being able to answer questions; the quiz measurement is meant to test that.
- That any of this transfers to bigger models or other training recipes.

## 5. Open questions

- Which safeguard mix (replay share, step size, pull-back) keeps a model stable over months of continuous reading? Document 11 lists the full set of open questions.
- Is there a version of "re-read the surprising parts" that improves understanding rather than only memory?
- Can the brain learn to choose its own actions better than the fixed reflexes?
