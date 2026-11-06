# 22. What we learned on the nights of 5 and 6 November

*Revised 6 November 2026. Replaces the 5 November version.

Snapshot dated 6 November 2026. This document reports seven things from the last two nights of work: our own CPU code for the eye, caption and hearing paths; a text memory store for notes, tested on new seeds; a safety skeleton for a self-improvement loop; a context test that needed no training; a long-context training run that ended invalid; the rehearsal check, whose rule is now met; and the first results of two eye arms (I and J). It also reports two Kaggle CPU measurements, the review passes that checked our plans and code, and how we check the order of rules and code.

Labels used throughout:

- **FACT**: read from a logged result file or log.
- **GUESS**: our reasoning or an estimate, not yet tested.
- **PENDING**: built or planned but not measured yet, or a value taken from our lab notes whose result file has not been re-read.
- **INVALID**: a run that failed its own written rule.

Pass and kill rules were written before each run. Most results come from one or two runs, so treat them as indications.

## 1. Our own CPU code, stage by stage

We rebuilt each stage of the model's pipeline in Rust, with no outside maths library, and compared its output with a reference run in Python at full precision. The stages below have been checked against a reference. Rows marked PENDING come from our lab notes, and their result files were not re-read for this revision. The GPU (CUDA) stage is not yet verified.

| Stage | Test | Result | Rule | Label |
|---|---|---|---|---|
| Sound picture (log-mel) | 5 clips, worst error per clip | Worst error 2.7e-5 | 1e-4 | FACT |
| Sound picture, control | Audio shifted by 10 ms | Error about 1.1, so the check does fail when it should | Must fail | FACT |
| Memory reuse (fixed prompt prefix) | 5 prompts | Output identical at every step, bit for bit | Identical | FACT |
| Full precision (f32) | 10 old prompts and 10 new prompts | 10 of 10 on both sets | 9 of 10 | FACT |
| Mixed 8-bit weights | 10 old prompts | 10 of 10 | 9 of 10 | FACT |
| Mixed 8-bit weights, held-out check A | 10 new prompts, not used for choosing | **7 of 10** | 9 of 10 | FACT, FAIL |
| Mixed 8-bit weights, held-out check B | 12 hand-written prompts from an independent reviewer | 11 of 12 | 9 of 10 | PENDING (lab note; file not re-read) |
| Plain 8-bit weights (every layer) | 10 old prompts | 4 of 10 | 9 of 10 | PENDING (lab note; file not re-read) |
| Eyes (picture encoder) | 3 fixed pictures, worst single value | 4.35e-4 off (values reach about 37) | 2e-3 | PENDING (lab note; file not re-read) |
| Caption path (eyes, connector, brain) | 5 test pictures | Same token numbers on 5 of 5; smallest gap between the top two next-word scores 0.066 | Identical | PENDING (lab note; file not re-read) |
| Hearing (speech to text) | 3 clips | Identical token numbers on 3 of 3; 6 wrong letters in 340 (about 1.8 percent) | 3 of 3 identical | PENDING (lab note; file not re-read) |
| First CUDA stage | Compiles for both cards | Compiles; the one-kernel check did not run because the GPUs were busy | Run on a free card | PENDING |

Speed numbers from this code were taken with the test machine under load (about 12 to 16 on 8 cores), so they are not speed claims. A quiet-machine timing is **PENDING**.

**Tuning sweep.** On 30 prompts, the nine full candidate variants (eight int8 and one int4) reached at most 26 of 30 exact matches with f32, below the bar of 27 of 30 set before the sweep (lab note). So f32 stays the default. Single-matrix probes scored higher (up to 30 of 30 for one projection), but they were sensitivity probes, not deployable variants, and none was adopted.

**Lesson: selection bias.** The mixed 8-bit variant was chosen because it scored 10 of 10 on the prompts we were also using to pick it. On fresh prompts it scored 7 of 10. The correct order is to fix the held-out set before any choice is made, use a set large enough to mean something (we now plan 50 prompts), and log the gap between the top two scores at each point where outputs diverge, so near-ties can be told apart from real errors.

**Two held-out checks of the same mixed variant disagree.** One scored 7 of 10 on new prompts; the other scored 11 of 12 on hand-written prompts from an independent reviewer. Samples this small give a wide range, so the honest reading is that the variant is close to the bar and not shown to pass.

## 2. Text memory store: retrieval tests

The store keeps short notes and returns the most useful notes for a question. The test set has 800 notes spread over a timeline of 8,000 seconds: 60 real notes from our own viewing logs and 740 notes built from public picture and speech datasets. The questions come in four kinds: a synonym ("canine" for a note that says "dog"), a literal word, a time phrase ("a minute ago"), and a question the notes cannot answer. The bars were fixed in advance: synonym recall at 5 of 0.70; summary-path synonym recall of 0.60; abstention of 0.80 on questions with matched wording; and a 95th-percentile latency under 200 ms.

| Round | Method | Seeds | Synonym recall at 5 (95 percent interval) | Summary path | Abstention (matched wording) | Latency, 95th percentile | Verdict |
|---|---|---|---|---|---|---|---|
| First hard test | Hashed word-count embedding plus word match | 1 to 3, pooled | 0.056 [0.021, 0.097] (word match alone 0.021) | not measured | not valid (wording leak) | median 2 ms (95th percentile not reported) | FAIL; kill rule fired (the combination was only 0.035 above word match; 0.05 needed) |
| v2 | Pretrained sentence embedder (MiniLM), fused with word match | 1 to 3, pooled | 0.271 [0.205, 0.349] (embedder alone 0.410) | 0.188 | 0.700 | 7.9 ms | FAIL on synonym, summary and abstention |
| v4, verdict | bge-base embedder, reciprocal-rank fusion, key-word query | 21 to 23 (held out) | 0.557 [0.477, 0.634] | 0.463 | 0.867 | 1,001 ms on a shared CPU | FAIL on synonym, summary and latency |
| v4, same settings | Same as v4 | 31 to 33 (fresh) | 0.473 [0.394, 0.553] | not reported | not reported | not reported | Analysis only |
| v5 (chosen) | v4 plus a cross-encoder re-ranking the top 20, query expansion by a 360-million-parameter language model (synonyms for each query word), and three views of each note | 31 to 33 (fresh; choices made on 11 to 13) | **0.649** [0.569, 0.721] | 0.372 [0.298, 0.452] | 0.873 | 412 ms on one pinned core (median 355 ms) | FAIL on synonym, summary and latency; abstention and time checks pass |
| v5, two-stage | Fast sentence-embedder check first; the full v5 only when unsure | 31 to 33 | 0.615 [0.535, 0.689] | not reported | not reported | 400 ms total (95th percentile); first stage 9.8 ms (95th percentile) | FAIL on the same bar; the cheapest route |

Time checks passed in every round: v5 parsed 144 of 144 time phrases, with no false triggers in 1,503 negative phrasings, and relative-time and minute questions returned the right note.

**Reading.**

- **Trend.** Synonym recall rose from 0.06 (first hard test) to 0.65 (v5). The rounds used different seed sets, so the clean comparison is the same v4 settings on two seed sets: 0.557 on seeds 21 to 23 and 0.473 on seeds 31 to 33. On seeds 31 to 33, the v5 additions gained +0.18 (paired interval +0.08 to +0.26). The seed-set difference (about 0.08) is as large as some of the gains, so single-round differences of 0.05 to 0.10 are not trusted.
- **Two-stage design.** The fast first stage (the sentence embedder alone) answered about 18 percent of the synonym questions on the held-out seeds. On those answers its top result was right about half the time. The other questions go to the full stage. The fast path keeps the median at about 10 ms in total, but the 95th percentile stays near 400 ms, because most questions in a mixed sample still escalate.
- **Abstention and time** pass, but the abstention test uses lists of absent words. It does not show that the store knows what it lacks: "submarine" sits close to boat and ship notes, so no threshold separates "not in the notes" from "the same kind of thing is in the notes".
- **Summary path.** The summary keeps the caption and drops the audio sentence, so summaries work for caption words only.
- **Decision.** We did not lower any bar. Retrieval tuning is paused, because the gains are getting small. The two-stage design is kept as the first memory-read path, with its limits written down. The next memory test will be about memory held in the model's weights and about consolidation, plus a paraphrase test built from real human text.

**Limits.** The synonym list is 200 word pairs typed by hand, and the same list is used for every seed. Latency was measured on a busy 8-core machine (load about 14 to 16), so the timings are upper bounds. The expansion words were not inspected by hand.

## 3. Closed improvement loop, version 0 (safety skeleton)

The loop proposes one small setting change at a time, and accepts it only if two judges of different design agree on held-out items and a canary test shows no loss. It runs on a simulated world with a single quality curve, so it tests control flow, not learning.

Final run (three seeds, 600 cycles each). No bad change was accepted, every judge hack was caught by the canary, and the other safety checks passed. The good-change count was 9 to 11 per seed, below the 40 required, so the verdict is **FAIL on that one check**. The gate behaved as designed: it accepted every good change it saw and no bad one. A GUESS on the cause: the simulated world has little room to improve. The best setting on the grid was only about 0.006 better than the current one on average. These counts are from our lab notes (PENDING: file not re-read). We did not lower any threshold. A fair pass needs a new world with more headroom, under a new written rule.

## 4. Context without training (stage 0, CPU only)

Question: can SmolLM2-360M with layered notes answer questions about a 200,000-word log better than a plain 8k window? Stage 0 only checked whether a planted fact reaches the model's context at all. No model was loaded, trained or run, so no accuracy is claimed.

- The plain 8k window (last 6,000 words) kept about 3.6 percent of planted facts. Flat keyword search over 200-word chunks kept 100 percent (99.2 to 100 percent interval).
- The layered routing rule that was pre-registered kept about 61 percent, and its verdict was KILL-LAYER. It kept none of the facts in the last tenth of the log. A GUESS on the cause: many blocks tie on score and the rule had no tie-break, so later blocks lost. KILL-LAYER kills that rule only; it does not test notes in general.
- Whether the model can read the kept facts is the open question. Only the model stage can answer it, and it needs a free GPU. It has not been run.

These figures are from our lab notes (PENDING: file not re-read).

## 5. Long-context training run: INVALID

**Goal (pre-registered).** Can continued training with a larger position base (rope base 1,000,000) lift the 360-million-parameter model to 16k and 32k tokens of retrieval, while the 4k text loss stays within 2 percent?

**What ran (FACT, from the run's result file).**

| Stage | Planned steps | Done | Note |
|---|---|---|---|
| 8,192 tokens | 200 | 200 | Complete |
| 16,384 tokens | 100 | 44 | Stopped |
| 32,768 tokens | 60 | 0 | Not reached |
| Total | 360 | 244 | **INVALID** |

Training stopped at 181.5 minutes, against a limit of 10,800 seconds (180 minutes). No steps were skipped. The weights were saved but not scored, so this run gives no retrieval number. The run's own built-in check printed 0.875 at 8k for the original weights, but that check has the flaw described in document 20, so the number is not used.

**Earlier attempt (FACT).** A first version crashed after 60 seconds on a configuration argument that the newer library no longer accepts in that form. Its job then showed RUNNING for about 12.5 hours before it was cancelled. A RUNNING status is not a result.

**Resume (PENDING).** The continuation (56 steps at 16k, then 60 at 32k) is written. Its script passed a CPU-only check (the model loaded with the new position setting, and attention and output matched the reference; no training was run). Its GPU push was refused twice because the weekly GPU quota of 30 hours was used up. It is expected to take about 7 hours once the quota resets.

**Data note.** Part of this run's text came from a public book collection. The book texts are public domain in the United States, but the collection's upstream notice does not recommend it for general-purpose language-model training. Whether to keep this source is a decision for the researcher, and it is not settled.

**Lessons.**

1. Set time limits from measured step times, not estimates.
2. Read the job log within minutes of a push. A RUNNING badge means nothing until the log shows steps.
3. Plan the weekly quota before the run.
4. Score checkpoints with a script that checks every loaded key. The needle script does not. A version that does exists, but it has no result yet.

## 6. Replay of old facts (rehearsal): supported, third seed of the no-rehearsal arm still running

Question: does replaying older fact sets during repair keep them, and how much of the retention depends on rehearsal? Same recipe as document 21, section 2.

| Seed | Arm R, weakest older set at stage L (rule: at least 0.90) | Arm N, weakest older set at stage L (rule: below 0.90) | Mean recall at L, R / N | Text drift at L, R / N |
|---|---|---|---|---|
| 1 | 0.95 (pass) | 0.00 (below 0.90) | 0.98 / 0.17 | +1.53 / +1.55 |
| 2 | 0.94 (pass) | 0.00 (below 0.90) | 0.97 / 0.17 | +0.97 / +1.02 |
| 3 | 0.94 (pass) | running (started 6 November, about 4 hours) | not reported | +1.52 (standard error 0.15) / not yet |

- **Without rehearsal the older sets are lost almost at once.** On both completed seeds the weakest older set fell to 0.00 to 0.04 within the first two stages.
- **Rehearsal passes on all three seeds** (weakest older set 0.94 to 0.95, text drift +0.97 to +1.53 percent, inside the 2 percent bar).
- **The written rule is met.** The rule calls the effect REHEARSAL-SUPPORTED when arm N falls below 0.90 on at least two seeds with both arms complete, and arm R passes on all three seeds. Seeds 1 and 2 give the two seeds for arm N, and arm R passes on seeds 1, 2 and 3. Seed 3 of arm N will add a third seed.
- **Rehearsal adds no visible text drift:** the drift is about the same in both arms (+1.53 and +1.55 on seed 1; +0.97 and +1.02 on seed 2).
- **Limits.** Recall is measured on the same 100 pairs per set that the model trains on, so there are no held-out pairs; a fixed version that adds them is built and has not run. The training loss goes through a half-precision path that was not compared with a full-precision path, and drift is measured on one held-out text pool.
- The unbatched training path was used throughout, because the batched path ran out of memory on the 8 GB card. Equivalence between the two paths was not measured.

## 7. Eye arms I and J: provisional, not a pass

Two eye arms add a trained tile selector: arm I (one tile chosen by the selector, 98 image tokens) and arm J (two tiles, 147 tokens).

- **Arm I:** colour 0.96 on seed 11 and 0.95 on seed 12, with captions 0.31 and 0.38. The selector's test accuracy was 0.93 and 0.92.
- **Arm J:** colour 0.97 on seed 11, reusing the seed 11 selector. Its seed 12 run was still running at the time of writing.

By the written rule, arm I meets all three conditions on both seeds. The lab treats I and J as **provisional**, because the audit in document 21 applies: the training canvases share one random stream across seeds, and the same test canvases are used for every arm. Arms I and J are not part of the first clean rerun (arms A, B and H3 on seeds 21 to 23). A separate clean run for arm I (same seeds, selector trained on the training split only, gated on validation, test scored once) is queued behind it; its go-flag was set after five CPU smoke checks passed. Arm J has no queued clean run.

## 8. Kaggle CPU measurements (no GPU)

Two CPU-only measurements ran on Kaggle's free CPU service, so no GPU quota was used. The sample was the first 5 million bytes of ten public-domain English books from Project Gutenberg, in a fixed order. Both measurements ran two complete passes, and the passes agreed exactly.

**Compression (FACT, bits per byte; decoder size not counted).**

| Method | Compressed size (bytes) | Bits per byte |
|---|---|---|
| gzip -9 | 1,883,882 | 3.01 |
| zstd -19 | 1,449,737 | 2.32 |
| zstd -22 | 1,449,681 | 2.32 |
| xz -9e | 1,437,976 | 2.30 |
| bzip2 -9 | 1,399,478 | 2.24 |

Every method decoded back to the exact input. These are classic tools, so their "decoder" is the program itself. A fair comparison with a model-based coder would also count the model's own size. Document 14 covers the model-based coder separately.

**Tokenizer coverage (FACT).** The SmolLM2-360M tokenizer produced 1,247,913 tokens on the sample (4.01 bytes per token).

- The vocabulary has 49,152 ids. The embedding table holds 47.2 million parameters, 13.0 percent of the 361.8 million total. The published total is about 362 million; our count differs by 0.0003 percent.
- Trimming the vocabulary would shrink the model by 4.3 percent (to 32,768 ids), 8.7 percent (16,384 ids) or 10.9 percent (8,192 ids).
- The 8,192 most frequent ids cover 95.8 percent of the tokens, 16,384 cover 99.25 percent, and 32,768 cover all of them. Covering 99 percent needs 15,188 ids.
- **Rule and result.** The rule in the tokenizer test note, written before the run: PASS if 16,384 ids cover at least 99.9 percent of tokens; KILL if they cover less than 99.5 percent. The result on this sample is **KILL** (99.25 percent). Pruning to 16,384 ids would re-map about 0.75 percent of tokens. The run's own reproducibility check passed; that is separate from this rule.

**Limits.** One sample of older English books, and one tokenizer. Our own 8,192-id vocabulary is a different tokenizer and was not measured. Code, web text or our training data would give different numbers. No second, separately pushed kernel has been run, so no reproduction is claimed.

## 9. Review passes: what they found

No code is described here; these are the findings.

- **Experiment plans (15 planned GPU experiments, two independent read-only reviewer passes each).** In the first wave every plan got a "fix" verdict, and none was ready to run. The common problems were queues that could start a job on a card another job was using; temperature checks that passed when a reading came back empty; comparators with unequal budgets; pass rules that could pass without the question helping; memory and time figures that were estimates; and one plan that would have trained on a line of text written by an assistant. After later fix waves, the latest pass recorded in the plan notes (late on 5 November) marked 12 of the 15 OK on their checklists. Two still needed fixes, and one's status was not stated. None has run, and each start still needs a decision by the researcher.
- **Queue scripts on the test machine.** Ten findings from a review of the scripts that run GPU jobs. Several waited on a card without taking the lock; several read the temperature in a way that did not halt on an empty reading; one job took the other card's lock; and some queues had stale headers or were duplicated. A shared locking library with 37 CPU tests was written, and all tests passed. Running jobs were not changed.
- **Code review of the queued GPU scripts.** Fifteen findings, five rated high: (1) a checker could be made to hang by one crafted string, and was replaced by a bounded evaluator with tests; (2) a training run could exceed GPU memory beside another job, so the decision to run it was left to the researcher; (3) one eye comparator received half the updates of the arms it was compared with, which the clean rerun design addresses; (4) a hearing verdict did not check for a clean-speech regression, and a fixed check was added by addendum; (5) a reasoning job could share a card with an eye job; the fix text is written and must be applied before that job starts. Medium and low findings led to guards against non-finite losses in the eye scripts, a gate moved from the test set to a validation set (by addendum), and a planned fix for a fixed random stream in the old scripts.
- **Audit of this repository (before this revision).** More than twenty statements in documents 20 to 22 were stale or overstated. Most are corrected here.
- **Numbers ledger.** A cross-check of 24 numbers between notes and result files found contradictions: one speed figure with no file behind it, an 8k value from an old generator, and finished runs listed as not run. Most are fixed in the notes.
- **Literature, licence and hardware checks.** Eight check notes covered licences, papers and hardware claims in our notes. They corrected several errors: a citation that pointed to a different paper, a licence that covered code but not the books, and a model licence that had been misread.

## 10. Rules before code: how the order is proved

For new work from 5 November onward, the order of rules and code is recorded as follows.

1. Write the rules file first. It states the pass rule, the kill rule, the verdict wording, the seeds, and what is not claimed.
2. Compute the file's SHA-256 hash, and append one line to an append-only log containing the hash and a timestamp with nanosecond resolution.
3. Create code and result files only after that line exists.
4. Later changes go in dated addenda. The original rules file is not edited, so its hash still matches.

Why a hash and not the file's modification time: on the test machine, modification times have one-second resolution, and two files written in the same second cannot be ordered. One loop run had this problem, so its results are labelled exploratory. Some older rules files have only file times, and their order is not proved.

The method shows the order in which rules and code were written. It does not show that the rules were good ones.

## 11. Mistakes we found while checking these results

- The eye glance arms were described as unfinished. Their equal-step runs exist (F 0.21, G 0.27). The two-glance run is no longer described as too short to count.
- The H3 result at seed 12 was described as not run. It is 0.77.
- The 8k needle value of 0.95 came from the old generator. The corrected value is 0.85.
- A speed figure of 173 tokens per second had no result file behind it, and is withdrawn. The measured figures are 190 tokens per second (speed run) and 179 (size run), both on the same card.
- The long-context run was described as pending. It ended INVALID.
- A zero at 32k and 64k needle length had no result file behind it, and is withdrawn.
- An earlier draft said rehearsal is supported "when two of three seeds show it with both arms". The written rule also requires arm R to pass on all three seeds.
- The statement that we do not train on text written by a commercial assistant needed a limit: the filter removes only the marked blocks.
- The statement that our CPU code runs "the eyes, ears and brain without Python" was too broad. Only the checks in section 1 have been run.
- The statement that no int8 variant reached 27 of 30 applies to the full candidate variants. Single-matrix probes scored higher; see section 1.
- One row of an internal verification note mixed up two notes-store numbers (the summary-path score was read as the synonym score). The result file gives 0.557 for synonyms and 0.463 for the summary path.
- Our notes named the notes-store rounds inconsistently. This document uses the names of the result files.

## 12. Still open

- **Notes store.** No further retrieval tuning is planned. Next: memory in the weights and consolidation, and a paraphrase test from real human text (needs a new written rule first).
- **Improvement loop.** A new written world with more headroom, so that the good-change count can be reached.
- **Long context.** The resume, then a key-audited needle test on any saved weights.
- **Our CPU code.** The GPU stage on a free card; quiet-machine speed with medians; a 50-prompt held-out set for the quantised weights, with logged top-two gaps, before any 8-bit variant is used again.
- **Kaggle CPU measurements.** A second, separately pushed kernel for reproduction, and a test on our own training text.
- **Eyes.** The clean rerun (A, B and H3), the queued clean run for arm I (none queued for J), quiet-card timing, and real video frames.
- **Rehearsal.** Seed 3 of the no-rehearsal arm, then the fixed version with held-out fact pairs and a full-precision loss path.
- **Data licence.** The decision on the public book collection used in the long-context run.
