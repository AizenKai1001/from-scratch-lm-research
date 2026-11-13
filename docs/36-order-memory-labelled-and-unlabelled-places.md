# 36. Order and memory over a stream: a registered test, the gates that failed, and what the audit allows us to say

> **Later note (13 November 2026).** A registered follow-up, [document 37](37-order-2c-labelled-recall-and-the-re-shown-copy.md), added the labelled arm, a re-shown copy and the stream-swap control that this document lacked. It replicated the unlabelled misses, closed the 'provisional' status of the verdicts here, narrowed 'the unlabelled result is a counting result' to an operational reading, and left the 'no memory claim' conclusion standing.

Version 1. Snapshot dated 12 November 2026. The run and its audit took place on 12 November 2026.

## Abstract

We asked whether two frozen language models can say which item sat at a named place in a stream, with and without a label on each item. A trial shows a stream of N items and asks about each place once. A rule with pass and kill lines was written and checksum-logged before the run, and an independent AI audit recomputed every number from the stored answers. Without labels, the 2B model got 32 of 60 whole trials right at four items, a result the rules leave unresolved, and 14 of 60 at five items, a kill; the 0.8B model got 5 of 60 and 1 of 60. With labels, at 22 and 72 items, both models got 98 to 99.5 percent of single questions right, which the audit reads as a lookup of a named label, not as memory. The main limit is that counting and memory were not separated, so this document makes no memory claim and two of the run's gates failed.

## Introduction

### Why this question

A language model reads its input as one stream of tokens. A question about which item sits at a given place tests whether the model keeps track of order inside that stream. We take this as one probe of order in context (GUESS). It is not a test of long-term memory.

### Terms

- **Stream.** A fixed list of N items, one item per place. In the cells reported here N is 4, 5, 22 or 72.
- **Place q.** The position q in the stream, counted from the first item (place 1) to the last (place N).
- **Trial.** One stream plus its questions. A trial asks about every place once.
- **Unlabelled cell (Level 1).** The items carry no place label. The question gives the place in words, so the model must count to find it.
- **Labelled cell (Level 2).** Each item carries a label such as "<<q" with its place number, and the question names the same label. The label removes the need to count.
- **Per query.** The share of single place questions answered right.
- **Full trial.** The share of trials in which every place question was answered right.
- **Four-ask readout.** Each question offers four options. The model is asked four times with the options in a rotated order. The audit calls the combined answer the primary readout, or the ensemble. A single ask is called RAW.
- **BARE.** The plain arm. The items are in context exactly as written, with no added hints.
- **Lookup baseline.** A score that a model can reach by matching a named label to an option, without counting.

### What this document reports

Only audited numbers are reported. Each cell is reported as one run, and the audit gives no repeat. Where an audit took a claim back, we say so and give the reason. The labels used are the same as in documents 34 and 35.

**FACT (audited)** means we measured it and an independent AI audit recomputed it. **GUESS** means our reasoning, not tested. **INFERENCE** means a conclusion drawn from audited facts. It was not tested directly.

The audit gives its own status for the run: AUDITED-CORRECTED. Each cell has a verdict of PASS, KILL, UNCLEAR or UNGRADEABLE, as the registered rules define them. These words name outcomes under the rules. They are not claims about memory.

## Methods

**FACT (audited).** Two frozen models, with weights unchanged during a run, were tested: the 0.8B and the 2B members of one public family. Each item is a token drawn from a fixed list. Items are in context verbatim in the BARE arm.

### The two kinds of question

**FACT (audited).** Unlabelled cells (Level 1) ran at N = 4 and N = 5. The question asks for the item at a place named in words. The items carry no label.

**FACT (audited).** Labelled cells (Level 2) ran at N = 22 and N = 72. The question names a label such as "<<q", and the stream carries the same label on the item at that place.

**FACT (audited).** A control named B1 shows one labelled block, then the unlabelled question. It checks that the model can match a label to an option. It does not test counting.

**FACT (audited).** Each question offers four options. Option j sits at label (j - v) mod 4 in ask v, where v is the rotation of the ask, one of four. The model is asked four times, and the four answers are combined.

### The registered lines

**FACT (audited).** The full-trial line for Level 1 is 0.50. The per-query line for Level 2 is 0.925. Chance for a four-option question is 0.25 per query.

**FACT (audited).** Under independent errors, a full trial has accuracy p to the power N, where p is the per-query accuracy. The full-trial line of 0.50 matches a per-query rate of 0.5 to the power 1 over N. That is 0.841 at N = 4 and 0.871 at N = 5.

**INFERENCE.** The verdict follows where the line falls against the interval. KILL means the 99 percent upper bound is below the line. PASS means the 95 percent lower bound is at or above the line. Otherwise the verdict is UNCLEAR. Every cell listed in the audit fits this reading.

**FACT (audited).** For a task, the verdict is KILL if any decision cell is KILL. Otherwise it is UNGRADEABLE if any cell is ungradeable, then UNCLEAR, and PASS only if every cell passes.

**FACT (audited).** Full-trial counts use exact binomial (Clopper-Pearson) bounds at 95 and 99 percent, over 60 streams at N = 4 and N = 5. Per-query accuracy at N = 22 and N = 72 uses a bootstrap over 40 and 30 streams.

### Gates and checks

**FACT (audited).** The rules set gates that must pass before a cell may be graded. They include an identity check on item tokens, a wording check on the words "first" and "last", a repeat check, a tie check, and the B1 control on each task.

**FACT (audited).** The secret main streams were not published. The audit did not read the master seed, and the seed value is not given here.

### Independent audit

**FACT (audited).** The audit wrote its own scorer from the rule text. It did not import the engineer's scorer. It recomputed the four span cells for both models.

**FACT (audited).** The audit's scorer agreed with the stored prediction on every query in the four cells. The audit also rebuilt the input tokens and recomputed the digests of all 2,160 asks for the two Level 1 cells. All 2,160 matched the logged digests.

### Procedure order

**FACT (audited).** The points below are taken from the audit's own list. Each is stated plainly, and none changes a registered verdict.

- The rules were registered before the gates ran and before any calibration result. The launch scripts were hash-logged after they ran.
- The readings were registered after the first scripted runs, a self-test and a smoke run. The rules forbid new addenda after the first scripted run, except bug fixes. The readings were registered before any gate or model result.
- One reading set a registered n-back item to NOT RUN. The audit calls this a disposal of a registered item by reading.
- The scorer was re-logged after the gate results and before any calibration or control result. The change added final-mode handling and a label. The audit found no change to a threshold or readout.
- The 2B's BARE run started before its main control result was scored or logged. The audit says this had no effect on any number.
- The 0.8B main block control was skipped. Its memory-limit wording therefore rests on one check only.
- The N-MISMATCH check, rule 7.7, has not been run. Every span verdict is provisional until it is.

## Results

### Table 1. Registered lines and the audit's counts

| Level and cell | Line | Model | Right / total | Estimate | Bounds | Verdict |
|---|---|---|---|---|---|---|
| Level 1, N = 4, full trial | 0.50 | 2B | 32 / 60 | 0.533 | 95% lower 0.420; 99% upper 0.685 | UNCLEAR |
| Level 1, N = 4, full trial | 0.50 | 0.8B | 5 / 60 | 0.083 | 99% upper 0.204 | KILL |
| Level 1, N = 5, full trial | 0.50 | 2B | 14 / 60 | 0.233 | 99% upper 0.384 | KILL |
| Level 1, N = 5, full trial | 0.50 | 0.8B | 1 / 60 | 0.017 | 99% upper 0.106 | KILL |
| Level 2, N = 22, per query | 0.925 | 2B | n/a | 0.982 | 95% lower 0.976 | PASS |
| Level 2, N = 22, per query | 0.925 | 0.8B | n/a | 0.981 | 95% lower 0.973 | PASS |
| Level 2, N = 72, per query | 0.925 | 2B | n/a | 0.995 | 95% lower 0.993 | PASS |
| Level 2, N = 72, per query | 0.925 | 0.8B | n/a | 0.984 | 95% lower 0.980 | PASS |

*Table 1.* Right answers and verdicts for the four span cells, from the audit's own recomputation (status AUDITED-CORRECTED, one run per cell). Full-trial rows count streams fully right out of 60. Per-query rows give the estimate and its 95 percent lower bound, from a bootstrap over streams. The task-level verdict for the span task is KILL, because the 2B's N = 5 cell is a KILL. The UNCLEAR at N = 4 cannot soften it. Source: the audit note, its reproduction table.

### Gates

**FACT (audited).** Table 2 gives the gates and their outcomes. Two gates failed, and both concern how the question is read.

### Table 2. Gate outcomes

| Gate | What it checks | 2B | 0.8B | Outcome |
|---|---|---|---|---|
| G-TOK-ID | Identity check on item tokens | 400 of 400 | 400 of 400 | Passed |
| G-WORD | Answer to "first" questions, and to "last" questions | first 0.96; last 0.08 | first 0.80; last 0.35 | FAILED for both |
| G-REPEAT | Repeat check | 0 flips in 100 | 0 flips in 100 | Passed |
| G-TIE | Tie check | 0 ties in 500 | 0 ties in 500 | Passed |
| B1, n-back | Control for the n-back task | AUC 0.24 and 0.26 | AUC 0.35 and 0.29 | Not passed for both |

*Table 2.* Gate results for the run (status AUDITED-CORRECTED). The AUC values are for the n-back control, where a value of 0.5 is chance and a value below 0.5 means the answers run the wrong way. Source: the audit note, sections 1 and 3.

**FACT (audited).** The G-WORD failure is the main reading problem. The 2B answers "last" questions as if they said "first". The 0.8B does not show that pattern. Its "last" score of 0.35 fails the gate, but the audit does not say it reads "last" as "first".

**FACT (audited).** Because the n-back control was not passed, the n-back task was not run in BARE form. Because G-WORD failed, the order task was not run.

**INFERENCE.** The n-back control fails because its wording inverts, not because the models cannot read. The audit calls this a wording or polarity defect in the control question.

### Unlabelled trials (Level 1)

**FACT (audited).** At N = 4 the 2B got 32 of 60 whole trials right. The 95 percent lower bound is 0.420 and the 99 percent upper bound is 0.685. The line of 0.50 lies inside that range, so the verdict is UNCLEAR.

**FACT (audited).** At N = 5 the 2B got 14 of 60 right, with a 99 percent upper bound of 0.384. That is below the line, so the verdict is KILL.

**FACT (audited).** The 0.8B got 5 of 60 at N = 4 and 1 of 60 at N = 5. Both are KILL.

**FACT (audited).** Whole-trial scores fit independent errors. The 2B's per-query rate at N = 4 is 0.854. Its whole-trial rate predicted from that is 0.854 to the power 4, or 0.532, against 0.533 observed. At N = 5 the 2B's per-query rate of 0.753 predicts 0.243, against 0.233 observed.

**INFERENCE.** At N = 4 the 2B sits on the line. Its per-query rate is 0.854, just above the 0.841 needed for a whole-trial rate of 0.50. At N = 5 it misses, with a per-query rate of 0.753 against the 0.871 needed.

**FACT (audited).** The four-ask readout helps the 2B. A single ask (RAW) gives 0.62 per query at N = 4, against 0.854 with the four-ask readout. The audit says the ensemble helps the model and does not hurt it.

**FACT (audited).** The letter bias was checked. Each answer letter was used about a quarter of the time, and accuracy was flat across letters. The audit concludes that letter bias is not the cause.

### Place by place

**FACT (audited).** Table 3 gives the 2B's accuracy at each place. The first place is almost always right. The last place is the weakest at N = 4. This is the opposite of a recency curve, which would make the last place the easiest.

### Table 3. The 2B's accuracy by place, unlabelled

| Stream length | Place 1 | Place 2 | Place 3 | Place 4 | Place 5 |
|---|---|---|---|---|---|
| N = 4 | 1.00 | 0.92 | 0.85 | 0.65 | not used |
| N = 5 | 1.00 | 0.98 | 0.70 | 0.50 | 0.58 |

*Table 3.* Share of place questions answered right by the 2B, unlabelled, from the audit's place-by-place reading (status AUDITED-CORRECTED, one run). The 0.8B's place 1 score is 0.93. Its other places are near chance, and the note gives no per-place figures for them. Source: the audit note, section 3.

**FACT (audited).** Errors cluster next to the true place. At N = 5, 74 percent of the 2B's wrong answers are an adjacent place. A random wrong pick would give 41 percent. The wrong answers skew to the next item (+1, 38 times) more than the previous item (-1, 17 times).

**INFERENCE.** The audit reads this as ordinal addressing. The model finds a place by counting, and its count drifts near the end of the stream. On the audit's reading, the items themselves are not lost.

**FACT (audited).** The 2B's answers to SPAN-B, the backward version, point to the first item in 60 percent of queries at N = 4, where chance is 25 percent. The audit reads this as the same wording failure as G-WORD. A label-free centring did not rescue the backward readout, which scored 0.16 to 0.24.

### Labelled cells (Level 2)

**FACT (audited).** With labels, both models score near the ceiling. The 2B got 0.982 per query at N = 22 and 0.995 at N = 72. The 0.8B got 0.981 and 0.984. All four cells pass the registered line of 0.925.

**FACT (audited).** The B1 control gives 0.98 to 1.0 on single labelled blocks. This is a label-to-option match, so it does not test counting. The audit says the labelled result is a lookup baseline.

**FACT (audited).** The 2B's errors at N = 22 are mostly a label collision. Ten of its 16 errors ask for the label "<<1", and the model answers with a place from 10 to 19. The labels share a prefix, and the model confuses them.

**FACT (audited).** The 0.8B's errors at N = 72 are mostly the neighbouring label. Its asked label 71 is answered as 72. Its errors cluster on certain items: flagged items make up 26 percent of its errors, against 11 percent of queries.

### Table 4. Item-prior check

| Cell | 2B p-value | 0.8B p-value |
|---|---|---|
| N = 4, unlabelled | 0.090 | 0.036 |
| N = 5, unlabelled | 0.024 | 0.029 |
| N = 22, labelled | 0.94 | 0.51 |
| N = 72, labelled | 0.68 | 0.0023 (variant 0.0026) |

*Table 4.* Item-prior check, which asks whether answers lean toward particular items regardless of the stream. One p-value per cell (status AUDITED-CORRECTED, one run). Only the 0.8B's N = 72 cell is below 0.01. Source: the audit note, section 1 and section 3.

### Claims the audit took back

**FACT (audited).** The audit took back these claims. Each is listed with its reason.

- A memory limit as the reason for the Level 1 KILL. Counting was not excluded, and for the 0.8B not even the B1 control was run.
- "Reader limit" for the n-back task. The audit calls it a wording inversion, shown by AUC values of 0.24 to 0.35.
- "SPAN-B at chance" as a memory result. The audit calls it a readout and direction problem.
- "Beyond the human band" for the labelled cells. Level 1 is a KILL, so this cannot be claimed. A scorer flag that read "beyond" is not a result.
- Any statement that four or five items are forgotten, or that the model does not remember. The audit found no test of recognition, because the options always occur in the stream.
- Any span verdict as final. Every span verdict is provisional until the N-MISMATCH check runs.

## Discussion

### What the data say

**FACT (audited).** Without labels, neither model reaches the registered whole-trial line at five items. The 2B sits on the line at four items. The 0.8B is below it at four items.

**INFERENCE.** The unlabelled result is a counting result. The labelled result is a lookup result. The two are not the same test of order.

### What the data do not say

**FACT (audited).** Counting and memory were not separated. The run has no labelled arm at N = 4 or N = 5 and no counting-only control. Because the options always occur in the stream, item recognition was never tested.

**INFERENCE.** The best-supported reading is that these models cannot count places among unlabelled items at these lengths. The audit says "does not remember" is not shown.

### Alternative explanations

**GUESS.** A counting drift explains the place-by-place curve. A label collision explains the labelled errors. A wording failure explains the backward result and the G-WORD gate.

**GUESS.** A repaired "last" wording may change the backward results. The run cannot tell this apart from counting, because the backward readout was run in RAW form only and is descriptive.

## Limitations and threats to validity

- **Scale.** Two small models from one family. The result does not show anything about larger models.
- **One run.** Each cell ran once. The audit gives no repeat.
- **Few streams.** Each unlabelled cell used 60 streams. The 2B's N = 4 interval runs from 0.42 to 0.68, so that cell is wide.
- **Items.** Items are tokens from one fixed list. The result may not carry to other items.
- **Counting versus memory.** No counting-only control and no labelled arm at N = 4 or 5. This is the main limit.
- **Gates.** Two gates failed. The order task was not run, and the n-back task was not run in BARE form.
- **Procedure.** The readings were registered after the first scripted runs. One registered item was set to NOT RUN by reading. The scorer was re-logged after the gate results. The 0.8B control was skipped. The N-MISMATCH check has not run.
- **Ceiling.** The labelled scores are near 1.0, so they leave little room to show anything beyond a lookup.
- **Reviewer.** The audit is one pass by a separate AI reviewer. Nothing here has been peer reviewed.
- **Human comparison.** The audit ties any human comparison to conditional lines. Level 1 is a KILL, so no human comparison is made here.

## Reproducibility and data availability

**What is shared.** The lines, the verdict reading, the four-ask formula, the gate outcomes, the table counts, and the audit's method of recomputing each number from stored answers.

**What is not shared.** The stream contents, the master seed, the model weights, and the machine details. The audit did not read the seed, and the seed value is not given here.

**What a reader can check now.** The arithmetic in Table 1. For example, 32 of 60 is 0.533, and 0.854 to the power 4 is 0.532. The per-query equivalents of the 0.50 line are 0.841 at N = 4 and 0.871 at N = 5, each computed as 0.5 to the power of one over N.

A reader cannot yet rerun the test exactly, because the stream generator and the registered rule text are internal.

## Conclusion and open questions

**Measured (audited).** Without labels, the 2B is unresolved at four items and fails at five. The 0.8B fails at both. With labels, both models reach 98 to 99.5 percent per query, which the audit reads as a lookup. Two gates failed, and the audit took back the memory reading.

**Guessed.** Labelled span tests at N = 4, 5 and 9, with a counting-only control on the same blocks, would show whether counting is the limit. A repaired "last" wording and a repaired n-back question would test the two failed readings. A run of the N-MISMATCH check would show whether any span verdict holds.

**Guessed.** A human reference would need a registered line and its own source. The audit note gives no human figure in the text used here, so none is set here. None of this shows anything about how people or any other system keep order.

## References

1. Internal audit note on the ORDER-2b run, skeptic audit, dated 12 November 2026, status AUDITED-CORRECTED. Not published. Every number in this document comes from it.
2. Document 34 in this repository, "What we tested on 12 November and what the audits took back". Read for the audit practice.
3. Document 35 in this repository, "Eyesight, a second try: a repaired ring test, a mirror-image test and a zoom test". Read for the form of the rules and audits.

The audit note does not name a published human reference in the text used here, so none is cited.
