# 10 Adaptation and biology notes: ideas, tests and corrections

How to read the labels. FACT is a value measured in our own runs, with the setting in brackets (model size, tokens, seeds, windows). PUBLISHED is taken from a paper, with authors, year and arXiv identifier or venue; where our notes read only an abstract or a search summary, the grade is given. GUESS is reasoning that was not tested.

## Summary

We collected ideas from how brains, animals, immune systems and colonies learn, and wrote down which ones a 143-million-parameter language model could test. Two CPU tests were run on 2026-10-29: a scan of training snapshots, in which the weight scale doubled in one session after weight decay was cut five-fold (FACT), rarely firing units rose (FACT), and the test's unit gate failed; and a test-time adaptation pilot, in which small gradient steps on earlier text lowered the loss on later text by 0.00805 bits per byte (FACT; 0.67 percent; one checkpoint). Independent review passes changed several of our first claims, including the claim that no small-model evidence existed for test-time adaptation, which was wrong. Most other ideas are designed but not run, or were dropped, and none has a measured benefit for our model beyond the pilot. The human reading findings in section 4 come from published work, are graded by source strength, and have not been tested on our model.

Basis: our own biology and adaptation notes, test designs, two skeptic reviews, an interpretability note that measured the same weight-decay switch, and a note on human re-reading, written 29 and 30 October 2026. Those notes are not published; this document is meant to stand on its own. Where a review disagrees with an earlier note, the review is used and the change is listed in section 3.

## 1. Ideas and their status

Each row is one idea. Test status is tested (a run exists), designed (a written test exists but has not been run) or dropped. Biological sources are as cited in our notes. Expected gains in the notes are GUESS and are not repeated here.

| Idea | Biological source (as cited) | What it would mean for a language model | Test status | Verdict |
|---|---|---|---|---|
| Learn from earlier text while reading (test-time adaptation, called dynamic evaluation in the notes) | Fast, short-lived synaptic change and hippocampal fast learning (background, not tested in animals here) | Take small gradient steps on earlier text, then score later text | Tested: one-checkpoint pilot (section 2) | Keep as a lead. Gain on one checkpoint and one text source; narrow step window; needs a safe rule and a cooled-model check |
| Adapt to a task's own examples at test time (test-time training) | Rule learning from a few examples: honeybees learning same versus different (Giurfa et al. 2001, Nature 410:930) | Per-puzzle fine-tuning on the puzzle's demonstration pairs | Dropped for our model: published only on puzzles, at 1B and 8B (Akyürek et al. 2024, arXiv 2411.07279) | Context only |
| Lesson store: each lesson has a trigger, is kept only if it would have fired on a replayed failure, is not kept if it blocks a good action, and fades by recency | Adaptive immunity: clonal selection and memory cells (background; a computing version is de Castro and Von Zuben 2002, IEEE TEVC 6:239, as cited); negative selection (Forrest et al. 1994, as cited) | A short memory of lessons for our research agents, checked before use | Designed (text-only replay test); not run | Open. No verdict below 40 scored incidents |
| Replay ordered by gain times need | Replay ordered by usefulness (Mattar and Daw 2018, Nature Neuroscience 21:1609); awake reverse replay (Foster and Wilson 2006, Nature 440:680) | Choose training windows to revisit by expected change in the model and by how likely they are needed | Tested in a simpler loss-based form: no gain (FACT, earlier 110M-parameter CPU test with 24 facts: 14 of 24 as taught, the same as plain replay). Gain-times-need form designed, not run | Loss-based form negative; gain-times-need form open |
| Replay of old data during new training | Interleaved replay in complementary learning systems (McClelland et al. 1995, Psychological Review 102:419, as cited) | Mix old data into new training to limit forgetting | Published only: Ibrahim et al. 2024, arXiv 2403.08763 (405M and 10B models) | Context; relevant to the data plan, not tested here |
| Reset rarely used units (continual backprop) | Loss of plasticity in continual learning (Dohare et al. 2024, Nature 632:768) | Reinitialize the least-used units on a schedule | Dropped: our runs are a single pass with no task switch, and the expected gain (GUESS) is below what four paired seeds detect | Dropped |
| Scale the learning rate by surprise | Octopus short-term learning-rate control (Shomrat et al. 2008, Current Biology, as cited) | Multiply the learning rate by a clipped loss-surprise ratio | Dropped: expected gain of 0.001 to 0.003 bits per byte (GUESS) is below what four pairs detect | Dropped |
| Sample high-loss windows more often | Attention to surprising experience; speed to a target error on image sets (Jiang et al. 2019, arXiv 1910.00762) | Back-propagate or sample high-loss windows more often | Designed (equal-token and equal-time arms); not run | Low prior. Run only the equal-time arm, and only after a duplicate check; otherwise drop |
| Dormant units and effective rank as health checks | Critical periods and loss of plasticity (Hubel and Wiesel 1970, J Physiol 206:419; Hernandez-Garcia et al. 2026, arXiv 2606.24752) | Count units that stop firing, and rank collapse, as warnings | Tested (test T1) | Keep as monitoring only; counts are proxies and do not measure plasticity |
| Weight decay and weight scale | Metaplasticity (Abraham and Bear 1996, Trends in Neurosciences 19:126, as cited); stronger decay gives a more adaptable base model (Han et al. 2026, arXiv 2602.11137) | Lower decay late in training rescales the weights, which may change learning speed | Tested for weight scale (test T1); effect on learning not tested | Open. The rescaling is measured; what it does to learning is not |
| Bet-hedged promotion from small screens | Bet-hedging in bacteria (Veening, Smits and Kuipers 2008, Annual Review of Microbiology 62:193) | Keep the top two candidates from 10M-parameter screens instead of one | Designed; not run; the free ranking count is not yet done | Open. Choosing the better of two can add about 0.001 to 0.002 bits per byte with no real gain (GUESS, reviewer arithmetic) |
| Shared-start model soup | Mixed phenotypes as insurance (Veening et al. 2008); model soups of fine-tuned models (Wortsman et al. 2022, arXiv 2203.05482) | Average the final weights of runs that share an early start | Designed; needs forked runs and a two-run check first; not run | Open. Published soups are of fine-tuned models, not from-scratch language models |
| Reinforce shared notes that are reused, and let unused ones fade | Ant trail reinforcement (Goss et al. 1989, Naturwissenschaften 76:579) | Agent notes gain weight when reused and fade otherwise | Designed; the first step is a count and needs no training | Open; relevant to the agents, not the model |
| Quorum for choosing experiments | Honeybee swarm choice by a quorum of scouts (Seeley et al. 2006, American Scientist 94:220) | Start an experiment when enough independent proposals back it | Designed as a text-only retrospective; not run | Low power; the agents are copies of one model, so their votes correlate |
| Early reward-free probe predicts learning speed | Early neurochemical signal tracked learning speed in 18 bees (PUBLISHED: Sands et al. 2026, Science Advances 12(7); correlational) | A probe before each new run might predict how fast it learns | Designed (log a probe before new runs); no verdict before 10 runs | Open |
| One-example binding of a made-up word | Human one-shot concept learning (Lake et al. 2015, Science 350:1332, as cited) | Does a made-up word defined once get reused later, and is that explained by induction heads (Olsson et al. 2022, arXiv 2209.11895)? | Designed (CPU probe across checkpoints); not run | Open. Copying by induction could look like binding, so a control is required |
| Narrow test-time updates: adapt only the output projections; down-weight repeated word sequences | Targeted plasticity of one sub-layer (Feng et al. 2026, arXiv 2604.06169); repeat-aware updates (Wang et al. 2026, arXiv 2607.03441) | Cheaper updates on fewer tensors; ignore repeated word sequences | Designed as later arms of the test T2 harness; not run | Open |
| Grow the model during training | Developmental stages and metamorphosis (background; not evidence for the model) | Start narrower, then widen a trained network without losing function (Chen, Goodfellow and Shlens 2016, arXiv 1511.05641) | Dropped for now: high engineering cost | Dropped for now |
| Sleep-like offline replay | Sleep replay and synaptic homeostasis (Tononi and Cirelli 2014, Neuron 81:12; Tadros et al. 2022, Nature Communications 13:7742) | Offline replay phases between training | Dropped: no small-model evidence found | Dropped |
| Easy-to-hard data order (curriculum) | Starting small (Elman 1993, Cognition 48:71) | Order training data from easy to hard | Dropped: curriculum entries in the 2023 BabyLM challenge were largely unsuccessful (Warstadt et al. 2023, arXiv 2504.08165) | Dropped |
| Per-layer prediction-error signals | Predictive coding (Rao and Ballard 1999, Nature Neuroscience 2:79) | Local error signals inside the network | Not designed; an idea only | Dropped |

## 2. The tests we ran

Two CPU tests were run on 2026-10-29. No GPU was used, and the training snapshots were only read. The probe text was human-written validation text (FineWeb-Edu); no machine-written text was used. The two tests used about half an hour of spare CPU time.

### Test T1: scan of the training snapshots

Question. Does a five-fold cut in weight decay (0.1 to 0.02), made late in training, change the weight scale, the rank, or the use of the MLP units?

Setting (FACT). The 143M model, two training runs (A and B). The cut came at about 653 million tokens in run A and about 621 million in run B. There were 28 snapshots (13 from run A, 15 from run B). Unit statistics used 16 windows of 1,024 tokens (16,384 tokens) of validation text. Weight statistics used all 48 weight matrices (12 layers, four matrix types each).

Pre-set rules. They were written before the new snapshots were scanned, with one exception noted in section 3. The results written after each rule are FACT.
- R1, reproduction: re-running the unit count on two earlier snapshots must reproduce the stored totals within one unit. Result: PASS.
- R2, weight jump: at the cut, the mean weight increase must be larger than each of the three earlier increases. Result: PASS in both runs.
- R3, break screen on seven indicators: a break is a jump above twice the largest earlier increase and above a floor. The same direction in both runs reads as consistent with a weight-decay effect.
- R4, unit gate: units that never fired must number at most 7 at every snapshot, and rarely firing units in layer 0 must stay below 100. Result: FAILED on the layer-0 clause in both runs. Run B was already above 100 two snapshots before its cut.
- Addendum A, written before the next snapshot was scanned: the next session's weight scale grows by 1.05 to 1.13 times. Result: PASS in both runs (1.112 and 1.125).

Results (FACT).
- Weight scale doubled in one session. The mean over the 48 matrices grew 2.016 times in run A and 1.957 times in run B. Each matrix type grew 1.90 to 2.04 times. Tables without weight decay (embedding, output head, value tables) grew about 1.03 times, as in earlier sessions.
- Cumulative growth against the snapshot before the cut: run A 2.242 after two sessions and 2.329 after four; run B 2.201 after two sessions and 2.282 after three.
- Stable rank fell about 2.5 percent in run A and 1.9 percent in run B. Effective rank did not change (under 0.2 percent).
- Units that never fired on the 16,384 tokens: 0 to 4 of 30,720 at every snapshot.
- Units firing on under 1 percent of tokens rose at the cut. Run A: from 61 to 280 in total and from 29 to 240 in layer 0, then 348 (312 in layer 0) one session later. Run B: from 216 to 299 in total and from 148 to 227 in layer 0. This is the one sparse-unit signal. A second text sample gave the same direction (layer 0 in run A: 28, 247, 308; run B: 158, 243, 270).
- Dormant units by a relu-squared score at threshold 0.1 stayed flat, at about 6,600 in run A and 7,900 in run B.
- The share of layer-0 output carried by the top 26 units fell from 0.84 to 0.70 in run A but rose from 0.88 to 0.90 in run B. The two runs diverged, and each cut was bundled with other changes, so no single cause can be named.
- The hidden-matrix gradient norm at the last logged step fell to 0.37 (run A) and 0.29 (run B) of its mean over the three sessions before the cut.

Reading (GUESS, not tested). Doubled weights make a fixed-size orthogonal update a smaller relative change, which would act like a partial learning-rate cut. The gradient drop fits this but does not prove it.

Predictions written before the scan were scored afterwards: three right, one half right, one wrong.

What the reviews changed.
- "Dead units" is dropped as a description. A unit silent on 16,384 tokens is not shown dead, and a review count moved by up to three between window sets.
- The equilibrium claim was wrong. Our first note said the two-session ratio matched the square root of five (2.236), the decay model's equilibrium. Run A kept growing to about 2.33, so the plateau is not settled (section 3, row 4).
- The residual-skip gains (the lambda parameters) rose two to nine times in every layer at the cut. The first note left this out.
- The first rule file was edited after the first scan results (section 3, row 24).

What T1 does not show: learning ability (plasticity) was not measured. The counts are proxies, and a 2026 preprint found proxies like these do not reliably predict plasticity loss (Hernandez-Garcia et al. 2026, arXiv 2606.24752). There are two runs, one seed each, one cut each, and snapshots taken at a constant learning rate.

### Test T2: test-time adaptation pilot

Question. For this model, do small gradient steps on the earlier part of a document lower the loss on its later part?

Setting (FACT). One checkpoint of run A at about 653 million tokens, taken at a constant learning rate, with a validation loss of 1.2120 bits per byte on the trainer's 2-million-token check. Text windows of 1,025 tokens with no end-of-text token, from the first 2 million tokens of validation text: 2 windows to find the step range, 8 to tune the step, 32 to report (three seeds), and 16 for a control.

Protocol (FACT). Each window has four segments of 256 tokens. Each segment is scored with all earlier text as context. After segments 0, 1 and 2, the weights take one normalized gradient step on that segment's loss, scaled to the size of the weights, with no momentum and no decay. The step touched 65 tensors (about 80 million of 143 million parameters). Weights are reset for every window. Gain is the loss with fixed weights minus the loss with steps, over tokens 256 to 1,023.

Pre-set pass rule. Written before any gain was computed; the review checked the file's timestamps against the first gain. A pass needs a gain of at least 0.004 bits per byte, a paired 95 percent interval above zero, a positive gain in each of the three later segments, and a passed causality test. Kill if the best tuning gain is below 0.002. The prediction written beforehand was a gain of 0 to 0.006 and a 25 percent chance of a pass.

Results (FACT, 32 report windows).
- Gain 0.00805 bits per byte (standard error 0.00122; 95 percent interval 0.0056 to 0.0105), which is 0.67 percent of the fixed-weight loss of 1.2043. Loss with steps was 1.1963. Better in 29 of 32 windows. By segment: 0.0037, 0.0072 and 0.0132.
- Rule: PASS. Causality: replacing later tokens changed no earlier loss (largest change 0.0). The prediction was too low.
- Step size: 0.000033 gave +0.0033 (8 of 8 tuning windows better); 0.0001 gave +0.0046 (6 of 8) and was chosen; 0.0003 gave -0.0281 (0 of 8). On two windows, 0.001 gave -0.315 and 0.01 gave -2.456.
- Control: the same steps taken from another document of the same corpus made the loss worse (-0.00361; 1 of 16 better). The document's own steps on the same 16 windows gave +0.00877 (14 of 16 better).

Review (FACT; fresh windows, same checkpoint and step, not re-tuned).
- 48 new documents from the same first 2 million tokens: 0.00840 (standard error 0.00142; 41 of 48 better).
- 48 documents from the part of the validation text the trainer never evaluates: 0.00514 (standard error 0.00107; 38 of 48 better).
- All 128 windows: 0.00709 (standard error 0.00074; interval 0.0056 to 0.0086; 108 of 128 better).
- The growth over segments held in the first region (0.0046, 0.0090, 0.0116) but was flat in the other part (0.0052, 0.0044, 0.0058).
- A neighbouring snapshot (run A at about 703 million tokens) gave 0.00431 on 32 windows of the second part, against 0.00407 for the earlier snapshot on the same windows. The review's own control on 16 windows gave -0.0018 for foreign steps and +0.0045 for the document's own steps.

What the reviews changed.
- The headline is text-specific: about 0.005 to 0.008 bits per byte, 0.00709 pooled, rather than 0.008 alone.
- "29 of 32" is a count of windows that were better, not a per-passage effect.
- The growth over segments is seen in the first region only.
- The "ten times the step" figure rests on two windows.
- Provenance gap: the first-stage log of the pilot was not produced by the analysis version now on disk. A likely cause (GUESS) is that the file was replaced while the stage was starting. An independent recomputation reproduced the logged numbers, but the original version is lost.

GUESS (not tested). If the steps continued across whole documents, the average gain might be roughly 0.002 to 0.006 bits per byte, since short documents gain little. Published context is in section 3, row 1.

What T2 does not show: one checkpoint from one run, uncooled, so a cooled or larger model may gain less (the published work reports smaller gains for stronger models; Krause et al. 2019, arXiv 1904.08378); one text source; a simple update rule, with no decay toward the fixed weights and no reset per document; a narrow step window; step size and position in the document are confounded; no change to the trained model; about 5.5 times the forward compute of plain scoring. A lower loss also does not show later use: a 2026 preprint reported loss gains with zero later recall on invented facts (Song et al. 2026, arXiv 2607.00368; PUBLISHED, abstract only).

## 3. Corrections we made

Each row is a claim from our first notes that a review or a second check found wrong, too strong or unverified, or a process gap in how a test was run. Numbers from published work are PUBLISHED with their source; the reviewers' arithmetic is marked GUESS where it depends on an assumption.

| No. | Earlier claim | What the review found | Corrected wording |
|---|---|---|---|
| 1 | No small-model evidence exists for test-time adaptation; no test was found at 100 to 150M parameters (first dynamic-evaluation notes). | WRONG. Rannen-Triki et al. 2024 (arXiv 2403.01518, PUBLISHED) report about 1.3 percent lower loss for models of 150M to 1B parameters on book-length text, and 2.0 percent with a reset per book; the model size behind those two figures is not stated. Krause et al. 2019 (arXiv 1904.08378, PUBLISHED): a 277M Transformer-XL went from 0.993 to 0.940 bits per character on enwik8. | Small-model evidence exists: a few percent, mostly on long text. Our pilot adds one more data point. |
| 2 | A 1B model went from 5 to 29 percent on an 80-task ARC subset (Akyürek et al. 2024, arXiv 2411.07279, PUBLISHED as printed). | The paper's own ablation percentages equal task counts divided by 29, so 29 is most likely 29 of 80 tasks, or 36 percent (reviewer, about 85 percent sure; GUESS). The same 80 tasks tuned the settings. | Printed as about 5 to 29 percent; probably 29 of 80 puzzles. The same puzzles tuned the settings. |
| 3 | The 2026 plasticity preprint uses large models (first notes). | WRONG on size. Hernandez-Garcia et al. 2026 (arXiv 2606.24752, PUBLISHED) cover 5M to 314M non-embedding parameters, and their fit puts plasticity loss at tens to hundreds of billions of tokens. For a model our size, the reviewer's arithmetic gives about 170 to 360 billion tokens (GUESS). | The preprint studies small models, and its fitted onset is far beyond our 45-million-token runs. |
| 4 | The weight growth matches the equilibrium of the square root of five, so the weights have settled (test-design note). | OVERSTATED. Run A kept growing: 2.307 and then 2.329 times against the pre-cut snapshot (FACT). The plateau is about 4 percent above the square root of five (GUESS). | The weight scale rose about 2.0 times in one session and about 2.3 times in four sessions in run A; the plateau is not settled. |
| 5 | "Whatever else the switch did, it rescaled the network" (test-design note). | OVERSTATED (incomplete). The residual-skip gains rose two to nine times in every layer at the same cut (FACT, review check). | The weights doubled, and the residual-skip gains also rose, two to nine times in every layer. |
| 6 | "No dead units" (scan summary). | A unit that never fired is not thereby dead. The unit gate failed on its rare-unit clause in both runs (section 2). | No unit that never fired on the test text (0 to 4 of 30,720); the unit gate failed on rarely firing units. |
| 7 | The gain is 0.00805 on 29 of 32 passages (pilot note). | "29 of 32" counts windows that were better; the 0.00805 is a mean. The size depends on the text: 0.00840 on 48 new documents of the same region, 0.00514 on the part of the file the trainer never evaluates (FACT). | About 0.005 to 0.008 bits per byte depending on the text; 0.00709 pooled over 128 windows, better in 108 (FACT). |
| 8 | The gain grows with each segment: 0.0037, 0.0072, 0.0132 (pilot note). | Reproduced in the first region only; flat in the rest of the validation text (FACT). | The rise over segments was seen in one region of the validation text. |
| 9 | A bar of 0.004 is a low false-positive filter: with four paired seeds, a true 0.006 effect passes about 95 percent of the time and a null effect almost never (ideas note, statistics). | OVERSTATED. Pass rates depend on the spread between paired seeds, which three pairs estimate poorly. With a spread of 0.004, a true 0.006 effect passes 84.1 percent of the time and a null effect 2.3 percent; with 0.0057, 75.9 and 8.0 percent (reviewer arithmetic; the spread is GUESS). | A rough filter. Measure the spread from all pairs before trusting a pass. |
| 10 | A honeybee brain has about 950,000 neurons in about 1 mm3, and bees learn same versus different without a large brain (brains note). | OVERSTATED. A book-chapter excerpt gives about 850,000 neurons for a worker bee; other estimates reach 960,000; the 1 mm3 volume was not verified. "Without a large brain" was our reading; the authors describe concept learning. | Estimates of about 850,000 to 960,000 neurons; the authors interpret the behaviour as concept learning. |
| 11 | A quorum of dancers triggers swarm takeoff (ideas note). | WRONG. The quorum is about 15 scouts present at the site, which pipe (Seeley and Visscher 2004, search summary; Seeley et al. 2006, American Scientist 94:220, excerpt). | About 15 scouts present at one site form the quorum. |
| 12 | A slime mould slowed after each periodic pulse, which its authors read as anticipation (ideas note). | WRONG wording. After three pulses at fixed intervals, speed fell at the time the next pulse was due, even with no pulse (Saigusa et al. 2008, PRL 100:018101, abstract). | The slime mould slowed when the next pulse was due, even without it; the authors call this anticipation. |
| 13 | The slime mould found the most direct route and skirted longer options (ideas note). | OVERSTATED. Secondary sources say it withdraws from everything except the shortest route; the Nature abstract was not readable (Nakagaki et al. 2000, Nature 407:470). A feedback model reproduces the shape. | It withdraws from all but the shortest route, through physical feedback rather than planning. |
| 14 | Nimodipine abolishes LTD but not LTP (notes on Bi and Poo 1998, J Neurosci 18:10464). | OVERSTATED. It abolished LTD and reduced LTP to 27.3 percent against 48.4 percent (n = 4; p = 0.068; not significant). | It abolished LTD and reduced LTP, not significantly. |
| 15 | Immune and bacterial memory is specific and fades; nothing real gains general immunity from one hit (biology notes). | OVERSTATED. Adaptive memory is mostly specific to one pathogen or close relatives. Antibody half-lives are estimated at 11 years (tetanus) and at about 50 to over 200 years (varicella, measles, mumps) (Amanna et al. 2007, search summary; model-based). Innate cells show a form of memory (Netea 2020, as cited). The bacterial example (Lambert and Kussell 2014, PLoS Genetics e1004556) is dilution of one protein in one system. | Adaptive memory is mostly specific, and how long it lasts depends on the antigen; innate memory also exists. |
| 16 | A primary antibody response peaks in 5 to 10 days; a secondary one in 2 to 5 days (biology notes). | UNVERIFIED. A textbook gives a usual lag of 5 to 7 days to the first antibody, not a peak; no source was found for the secondary peak. | Do not quote peak days. |
| 17 | A crow solved every valid trial within 2 minutes; metatool sample size unverified (brains note). | One bird bent the wire (n = 1; Weir et al. 2002, Science 297:981, as cited). In the metatool task, four of seven crows solved it on the first trial (Taylor et al. 2007, Current Biology 17:1504, abstract). The 2-minute figure was not verified. | One crow bent wire; four of seven crows solved a metatool task on the first trial. |
| 18 | Octopus observational-learning replications failed (brains note). | The failure claim rests on a student newspaper article. The 1992 result (Fiorito and Scotto 1992, Science 256:545) was disputed in a 1993 exchange; no primary replication was found. | Contested; no primary replication found. |
| 19 | The fly mushroom body has about 2,200 neurons in 15 compartments (brains note). | OVERSTATED (count). The abstract says about 2,000 Kenyon cells in 15 compartments (Aso et al. 2014, eLife). | About 2,000 Kenyon cells in 15 compartments. |
| 20 | Starting small helped simple recurrent networks learn complex sentences (Elman 1993), and "start big" is sometimes right (brains note). | OVERSTATED. Later work found starting small unnecessary once meaning constraints were added (Rohde and Plaut 1999, Cognition 72:67, search summary). The "start big" clause was not verified. | Staged input helped in Elman's setting; later work found it unnecessary. |
| 21 | Popular-fiction comparisons: one exposure gives instant immunity; a skill-copying rule appears only on a fan wiki (popular-fiction notes). | WRONG as descriptions of the sources. In the story the adaptation is staged and takes time, and the opponent wins by exploiting that delay. The copying rule is in the official series summary. | Use sourced wording only. These comparisons illustrate; they are not evidence. |
| 22 | After predators reached island lizards, the advantage of longer limbs flipped within about six months, a fast evolutionary change (evolution notes). | OVERSTATED. The study reports a reversal of selection among survivors within one generation; heritable change was not shown (Losos et al. 2006, abstract, as cited). | Selection reversed within a generation; heritable change not shown. |
| 23 | The pilot's numbers come from the analysis version now on disk (pilot note, reproducibility). | Process gap. The first-stage log was not produced by the analysis version now on disk (a likely cause, GUESS: the file was replaced while the stage was starting). An independent recomputation reproduced the logged gains on eight windows to four decimals, and the report windows to within 0.00001 nats per token, but the original version is lost. | Recorded as a provenance gap; the numbers were reproduced independently. |
| 24 | The scan's rules were fixed before any new snapshot was scanned (test-design note). | Partly untrue. The rule file was rewritten after the first scan results, with an addendum appended. Only the second-session prediction was written before its snapshot was scanned. An independent check written before the norms were read predicted 1.103 and 1.116 and observed 1.113 and 1.1255 (interpretability note). | Say which rules were edited; rely on the blind check for the second-session step. |

## 4. Human re-reading and word look-up

These findings come from the reading literature, not from our model. Each is PUBLISHED and carries the grade our notes gave it: A (body text read through a web fetch; figures may differ from the original), B (abstract, methods or landing page read), B- (landing-page summary only), C (search summary or second-hand).

| Finding | Source | Grade |
|---|---|---|
| Regressions (backward eye movements) are about 10 to 15 percent of eye movements by one count and 15 to over 25 percent by another; the denominators differ. | Sturt and Kwon 2018, Frontiers in Psychology 9:1630; Inhoff and Weger 2005, Memory and Cognition 33(3):447-461 (both cite older reviews) | A (body text) |
| Most regressions go to the word just before; about a quarter stay inside the same word. | Inhoff and Weger 2005, Memory and Cognition 33(3):447-461 | A |
| Regressions follow misfit: implausible sentences drew more first-pass regressions (about 50 percent against 31 percent for the plausible version). | Sturt and Kwon 2018, Frontiers in Psychology 9:1630 | A |
| Look-ups track relevance: words central to a passage were checked well before peripheral ones. In one study, learners checked about 12 percent of target words. | Fraser's think-aloud study and Hulstijn et al. 1996, both as cited in a search summary | C |
| The retention benefit of look-ups is mixed; more look-ups did not always mean better comprehension. | Several studies, as cited in search summaries (including Koyama and Takeuchi) | C |
| Blocking look-backs in a trailing-mask study lowered comprehension, though the mask also blocked accidental skips, so the drop may reflect encoding. | Schotter, Tran and Rayner 2014, Psychological Science 25:1218-1226 (via Sturt and Kwon 2018) | C |
| Testing beats restudy: practice testing and distributed practice were rated high; rereading and underlining were rated low. | Dunlosky et al. 2013, Psychological Science in the Public Interest 14(1):4-58 | B- (landing page) |
| Testing beat restudy on tests at 2 days and 1 week; restudy won on a test after 5 minutes. | Roediger and Karpicke 2006, Psychological Science 17(3):249-255 | C (search summary) |
| Recall-type first tests gave bigger gains than recognition, and testing beat restudy overall. | Rowland 2014, Psychological Bulletin 140(6):1432-1463 | B (abstract) |
| The best gap between study sessions grows as the delay to the final test grows (317 experiments, 839 assessments); the ratio is not in the abstract. | Cepeda et al. 2006, Psychological Bulletin 132(3):354-380 | B (abstract) |
| The spacing advantage grows with practice and with delay, in an activation model (no magnitudes in the abstract). | Pavlik and Anderson 2005, Cognitive Science 29(4):559-586 | B (abstract) |
| About 15 percent of unfamiliar words met in ordinary reading were learned incidentally, pooled from 20 experiments. | Swanborn and de Glopper 1999, Review of Educational Research 69(3):261-285 | C (search snippet) |
| Reusing data for up to about four passes is nearly as good as fresh data at fixed compute (machine learning, included for comparison). | Muennighoff et al. 2023, arXiv 2305.16264 | B (abstract) |

The phrase "readers look up only important unknown words" is stronger than the sources. They say look-ups track relevance (row 4, grade C), not that only important unknown words are looked up.

What this could mean for our reader (GUESS, not tested): re-read the few words before a surprise rather than the whole passage; set re-show gaps from the time to the next check; test before re-reading; keep about four passes as a ceiling. These are plans for the reader-experiments document (08), not results.

## What this does not show

- Our two tests cover one model size, one checkpoint family and one text source. They do not show what a larger or cooled model would do.
- Apart from the pilot gain, no idea in section 1 has a measured benefit for our model. The gains in our notes are guesses.
- Learning ability (plasticity) was not measured. The scan measured weight scale and unit counts only.
- The human findings are not evidence about language models, and most come from abstracts or search summaries. The re-reading note has not had a skeptic pass yet.
- Pass rates and expected gains for tests not yet run are GUESS, and the 0.004 bits per byte bar is a rough filter (section 3, row 9).
- The notes were written with an AI assistant, and the reviews were also AI passes, not outside experts. They found errors, and more may remain.
- The lesson store is described only as a design. Its internal data are not reported here.

## Open questions

1. Does the test-time gain survive on a cooled or larger checkpoint, with a safe rule such as decay toward the fixed weights or a reset per document? This decides whether the idea is worth building. Importance: high.
2. Why is the gain smaller on the part of the validation text the trainer never reads (FACT: pooled 0.00826 on the first region against 0.00514 on the rest; z = 2.2)? Document mix is a guess (about 50 percent). More windows split by document type would test it.
3. Does a cache of earlier words, with no gradient steps, explain most of the gain? It is the top-ranked test in our current plan.
4. Does the residual-skip jump cause the rarely firing units (GUESS, about 65 percent)? A matched reset test would settle it.
5. Does the weight-decay cut reduce the ability to learn something new? It was not measured. A direct probe on both runs, with and without the cut, on a new text source would answer it.
6. Does an early reward-free probe predict learning speed in a trained network? The bee study is correlational, with 18 bees, which is too thin to plan around.
7. Does replay ordered by gain times need beat loss-based replay, and do lessons reduce repeated agent mistakes? The lesson test needs at least 40 scored incidents before any verdict.
8. Do runs that share an early start average into a better model at the 45-million-token harness size? The first step is to check the loss along the straight line between two final weights.
9. Does the "29" in the ARC result count puzzles? Settle it by reading the paper's figures.
