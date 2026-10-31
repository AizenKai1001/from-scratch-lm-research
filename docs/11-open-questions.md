# Open questions

## Summary

This page collects what we do **not** know, grouped by area, with a note on what would settle each question and roughly how cheap that is. Questions that other documents list as their own open items are repeated here in one place. "Cheap" means days on the hardware we have; "expensive" means it needs more compute than one person's two consumer graphics cards provide.

## The reader (documents 01, 08)

| Question | Why it matters | What would settle it | Cost |
|---|---|---|---|
| Does learning while reading help answer questions about the text, or only predict it? | Prediction gains alone could be memorisation of style and names | A harder quiz: our first one (document 08, section 9) was too easy and showed no change beyond noise, so the next version needs wrong options that fit the sentence, questions about names and events, and more items | cheap |
| Which safeguard mix keeps the model stable over weeks of reading? | Always-on learning drifts; replay worked for one book but weeks are untested | A long run through many documents with replay shares of 6 to 25 percent, a fixed drift monitor, and a rule for returning to the last good state | cheap to moderate |
| How much of a book is kept after many later books, and can revisiting raise it? | About 60 percent survives one or two later books at the careful step | Retention curves over 5 to 10 books with and without replay and with spaced revisits at gaps of 50, 300 and 1,000 steps | cheap |
| Does revisiting surprising spans help when the task is answering questions about them? | In our test it strengthened memory of the spans but did not change prediction | The same revisit arms scored with a harder quiz (the first quiz was too easy to show any effect of revisits) | cheap |
| Does a dictionary entry help the model learn a rare word when the learning also covers the entry itself? | In our first design (entry as context, loss on the sentence only) the entry made prediction about 7 nats easier but made learning worse than another word's entry (document 08, section 10) | A second design with its own rules: learn the entry text too, and a control that gives the word the same number of exposures | cheap |
| Can a model choose which dictionary sense fits a sentence by seeing which entry lowers the sentence's surprise most? | A rule-based sense choice reached only 55 percent on a small labelled set (the set and the test are not published here) | The same labelled set scored with this "let the brain choose" rule | cheap |
| Can the model learn to issue the actions (look up, revisit, take a note) itself, better than fixed reflexes? | The long-term idea is a brain that gives orders to a body | Train on its own logged successful traces and compare with the reflexes | moderate |
| Can the model's own margin notes or quiz answers ever be used as training text without collapse? | Using its own output risks collapse (published: Shumailov and colleagues 2023, Gerstgrasser and colleagues 2024) | A three-generation test with a ten percent share of its own notes and a real-text held-out loss | cheap |
| Do the gains hold on data the model has partly seen, and over much longer streams? | The eight-domain test (section 8 of document 08) covered code and mathematics with 80,000 tokens each, all held-out text | A test on data from the training mix, and streams of millions of tokens | cheap |

## Training (documents 03, 05, 06)

| Question | What would settle it | Cost |
|---|---|---|
| Is the step of weight decay from 0.003 to 0.0015 worth taking? Small-scale tests say the optimum is flat between production-equivalent 0.0001 and 0.0003, which lies far below 0.0015 and so leans towards stepping lower | A cooled branch of the main run at each value, compared at equal tokens | moderate |
| How long should the final cooldown be: 18 sessions or about 36? | A cooled branch from the planned arming point for each | moderate |
| Is the large loss drop at the weight-decay step a partial learning-rate cooldown, as we guess, or better learning? | An arm that lowers the learning rate by the matching factor at constant decay | cheap to moderate |
| One of the model's scalar gains (the largest of a layer-1 mixing scalar) keeps growing, about +1 per session over the last four sessions (30.45 at session 36, 35.2 at session 41, already above the monitor's trigger of 25), while loss improves. Does it saturate, and does it matter? | Continue monitoring; a test that clamps it | cheap |
| Is the data order (a shuffled pool plus fresh data against a file-ordered pool) a real effect or an artefact of how snapshots lean on their last slices? | A clean comparison with a shuffled copy of the pool at equal tokens (the later headline check was given up when the comparison chain was stopped) | expensive for one person |
| Do the head-size monitor and its band (0.70 to 0.78) carry over to other sizes? | Repeat on another model size | expensive |
| How does the model compare after the full planned nine billion tokens with other small models? | Rerun the comparison of document 04 at the end | cheap once trained |
| Is our recipe (Muon, small batches, constant rate with decay step-downs) better or worse than the standard recipe on our own data? | A second run with AdamW, a cosine schedule and large batches on the same data at the same token count; no such baseline exists yet, so document 04 cannot separate recipe from data | moderate |
| Is the gap to Pythia's public checkpoints at the same number of tokens (document 04, section 5: 0.30 bits per byte in our favour) a data effect, a warm-up effect or a batch-size effect? | A Pythia-style run on our data at a matched schedule, and a run of ours with a large batch | moderate |
| How much of SmolLM-135M's lead on our evaluation text is overlap with its training data? | A page-overlap check between our held-out documents and its training set | cheap to moderate |

## Small-scale tests and the curriculum (document 09)

| Question | What would settle it | Cost |
|---|---|---|
| Does "easy text first" still hurt at larger scale? At 4.5M parameters it did | The same four arms at 40M to 140M parameters | moderate |
| Why does ending on easy text collapse the model so badly (a rise from 1.48 half way through the run to 1.77 at its end)? | Check whether a mixed final phase avoids it | cheap |

## Interpretability (document 07) and biology notes (document 10)

The open questions of each study are listed at the end of its document. The common ones: whether findings on this one recipe at 143M parameters hold at other sizes, and whether any of the "plasticity" indicators we measured predict anything about later training.

## Method (document 12)

- How many independent seeds does a production-scale result need before a 0.01 bits-per-byte difference should be believed?
- How independent can an AI reviewer be of the AI that wrote the experiment?
