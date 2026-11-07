# 27. Where exact facts live in a small hybrid model, and whether they can be written into its weights

## What this is

Snapshot dated 6 November 2026. This document reports one memory test, two controls on it, and one test of writing facts straight into the weights. All three use Qwen3.5-0.8B, the small open language model that documents 24 to 26 use. It is released under Apache-2.0, as its model card metadata states (**PUBLISHED**: [model card](https://huggingface.co/Qwen/Qwen3.5-0.8B)).

Each test had a written rule with fixed bars, fingerprinted (a checksum and a time) before its script ran. Every fact in these tests is made up, so the model cannot know it from training. Treat every result as an indication.

The labels are explained at the end.

## Summary

| Question | Test | Verdict, as the rules read it | Main number |
|---|---|---|---|
| Where do exact facts live: in the fixed-size memories or in the growing caches? | Memory test, with two controls | CACHE MEMORY for the main test. The control on the zero reads PARTIAL | Full model 98 to 100 of 100. Caches with the memories zeroed 87 to 100. Windowed arm 0 of 100 at every count |
| Can one closed-form edit write new facts into one weight matrix? | Weight-edit test | T5-KILL-EDIT in both arms | 0 of 50 new codes recalled, and 0 of 50 paraphrased questions |

## 1. What we asked

Qwen3.5-0.8B has 24 layers of two kinds (**PUBLISHED**: its [config file](https://huggingface.co/Qwen/Qwen3.5-0.8B/resolve/main/config.json). The model card calls the two kinds Gated DeltaNet and Gated Attention. Here they are called linear-attention and full-attention layers).

- **18 linear-attention layers.** Each keeps one fixed-size memory per head, 288 heads in all. Every token fades the memory a little, so its size does not grow with the text.
- **6 full-attention layers.** Each keeps a cache of what it has seen, and the cache grows with every token. From the config: 12 KiB per token (**COMPUTED**: 6 layers, times keys and values, times 2 heads, times 256 numbers, times 2 bytes). A 4,000-token text needs about 49 MB of it (**COMPUTED**).

From the stored decay rates at zero input, 183 of the 288 memories (63.5%) lose half their content within 100 tokens, and only 8 (2.8%) keep half of it beyond 1,000 tokens (**COMPUTED** from stored parameters; the rates were not measured during text). Before testing, we guessed (**GUESS**) that facts older than a few hundred tokens would mostly be carried by the caches.

Two questions followed:

1. When a made-up fact must be recalled from earlier text, how much of the recall rides on the fixed-size memories, and how much on the caches?
2. Can one closed-form edit of one weight matrix, with no training steps, make the model recall new made-up facts?

## 2. How we tested

### 2.1 The memory test

- Each text was about 4,000 tokens of public-domain children's-book text. One line per fact read like "The access code for [made-up name] is [four digits]." A final question asked for one code.
- Each text held 2, 8, 32 or 128 facts. There were 100 texts per count, and the same texts went through three arms:
  - **Full model:** nothing changed.
  - **Windowed:** the six cache layers look back only 256 tokens. By design no fact sits in that window, so a fact can reach the question only through the memories and a relay that reaches about 1,530 tokens back (**COMPUTED**: 6 layers times 255 tokens).
  - **Memories zeroed:** the fixed-size memories are reset to zero at the start of the last 256 tokens. The caches, written while the memories were active, are kept.
- A fact counts as **beyond reach** when its line is more than 1,530 tokens before the end.
- The bars were written before the run, and no threshold changed. Some readings did change, in addenda written before the run. Section 6 lists each step (disclosed).
- Built-in checks passed: the self-test (28 of 28), the data checks, and two plumbing checks, which ran in a debug pass on 20 prompts. With a window larger than the text, and with the reset switched off, the arms gave exactly the full model's answers. The full model reached 1.00 at 2 facts, above its 0.90 floor.

### 2.2 The two controls

- **Control A: the window itself (2 and 8 facts).** Registered after the first verdict was written and before its run. It used 50 texts per cell, one window of 256 tokens, and a third arm that keeps the first four prompt tokens visible. Those kept tokens are called attention sinks (**PUBLISHED**: [Xiao et al., 2023, "Efficient Streaming Language Models with Attention Sinks"](https://arxiv.org/abs/2309.17453)). The question was whether the window itself breaks the model.
- **Control B: longer answers and a ranking read-out.** Registered after an AI code review had questioned the zero, and before any script for it was written. Two changes. Answers got 48 tokens instead of 12. And the model was also asked to rank the candidate codes. For each beyond-reach query, the prompt was followed by the stem of that fact's own sentence, cut just before the code, so the stem adds no information about the code. Each candidate code (2 or 8, all taken from the same text) was scored by the sum of its digit log-probabilities, and the highest score was the model's pick. Chance is 1 in 2 or 1 in 8. A tie counts against the right code.

### 2.3 The weight-edit test

- We chose one middle-layer MLP matrix, the down-projection that writes into the model's main stream. Each fact's target was the model's own output at the name's second mention while it reads the fact, averaged over 20 text prefixes. The matrix change was solved in one closed-form step (a ridge least-squares fit, with statistics from about 100,000 positions of filler text), with no gradient steps.
- The search grid was layers 6, 9, 12 and 15, with ridge strengths 0.1, 1, 10 and 100. All 16 settings were scored on 50 validation facts. The chosen setting was then scored on 50 test facts, a paraphrase set (50 questions), 150 control prompts (C1) and 30 well-known facts (C2).
- Two arms. Arm S took the target at the name's last token. Arm L took it at the final word "is". Arm L was written into the rules in advance as a second arm, to run only if arm S read as a kill. It did.
- The unedited model recalled 0 of 50 validation codes and 0 of 50 test codes, so the test starts from zero.
- Two picture checks from our earlier eye tests were run to see whether the edit disturbs unrelated abilities. Each uses 240 photos from a public photo set (Flickr8k), each with a small 8-pixel coloured square. One view shows the whole picture (D-8). The other is a 512 by 512 crop at the square (O-8).
- Some evaluation sentences (the well-known facts and the wording of the fact prompts) were written by an AI assistant. They were used only as test items, and no weights were trained on them. From now on, test items come from human-written or program-made sources, or from labelled outside models.

Both the memory and the weight-edit tests used bf16 numbers (16-bit floating point) on one consumer graphics card with 8 GB of memory. The linear-attention layers ran on slow reference code, so timings from these runs are not comparable with other notes.

## 3. Results

### 3.1 The memory test

Correct answers out of 100 texts per cell, with 95% Wilson intervals (a range that usually contains the true rate). All cells are **FACT**.

| Facts per text | Full model | Windowed | Memories zeroed |
|---|---|---|---|
| 2 | 1.000 (0.963 to 1.000) | 0.000 (0.000 to 0.037) | 1.000 (0.963 to 1.000) |
| 8 | 0.980 (0.930 to 0.994) | 0.000 (0.000 to 0.037) | 0.870 (0.790 to 0.922) |
| 32 | 1.000 (0.963 to 1.000) | 0.000 (0.000 to 0.037) | 0.870 (0.790 to 0.922) |
| 128 | 1.000 (0.963 to 1.000) | 0.000 (0.000 to 0.037) | 0.970 (0.915 to 0.990) |

Beyond reach, correct out of the queries in that subset (**FACT**):

| Facts per text | Queries beyond reach | Full model | Windowed | Memories zeroed |
|---|---|---|---|---|
| 2 | 41 | 41 | 0 | 41 |
| 8 | 59 | 58 | 0 | 52 |
| 32 | 70 | 70 | 0 | 65 |
| 128 | 68 | 68 | 0 | 66 |

- The windowed arm scored 0 even on the closest queries (36 queries, 449 to 499 tokens back). It also scored 0 at every distance below 1,500 tokens (0 of 59, 0 of 39 and 0 of 29 at 2, 8 and 32 facts). (**FACT**)
- The fixed-size memories did add some correct answers. With the caches present, the full model beat the memories-zeroed arm on 11 texts at 8 facts (exact McNemar p = 0.0010) and on 13 at 32 facts (p = 0.00024). At 128 facts the gap of 3 against 0 was not resolved (p = 0.25). (**FACT**)
- By the written bars, the reading that the memories hold the facts did not hold. The reading that the caches carry the rest did hold: the memories-zeroed arm scored 0.870 at 32 facts against a bar of 0.60, and the windowed arm 0.000 against a bar of 0.30. The combined reading is CACHE MEMORY, meaning that exact facts reached the question through the caches. (**FACT**)

### 3.2 The controls

**Control A.** Correct out of 50 texts per cell (**FACT**).

| Cell | Windowed | Windowed, 4 sinks kept | Full model |
|---|---|---|---|
| Inside the window, 2 facts (fact 40 to 179 tokens back) | 50 of 50 | 50 of 50 | 50 of 50 |
| Inside the window, 8 facts | 50 of 50 | 50 of 50 | 50 of 50 |
| Beyond reach, 2 facts | 0 of 50 | 0 of 50 | 50 of 50 |
| Beyond reach, 8 facts | 0 of 50 | 0 of 50 | 48 of 50 |

- Inside its window the windowed model works, so its zero beyond reach is not a plain sign that the window broke it.
- Keeping the sinks changed no recall count. The failing answers did differ. Without sinks, 16 of 50 beyond-reach answers at 2 facts were digit loops (8 or more identical digits in a row), against 4 of 50 at 8 facts (48-token run, see Control B). With sinks, most failing answers contained one of a few canned digit strings, such as 1234 or 123456. The string 1234 appeared in 50 of 50 failing answers at 2 facts and in 47 of 50 at 8 facts. It was the first exact four-digit run in 38 and 4 of those answers (48-token run). Other exact runs were 1000 and 2020. None matched a fact's code. (**FACT**)
- The 100 failing windowed answers in the beyond-reach cells (12-token run) restated the queried name 78 times and gave something else 22 times. None gave four digits. (**FACT**)

**Control B.** The windowed arm with 48-token answers. Categories are not exclusive (**FACT**).

| Cell | Texts | Correct code | Restated the name | Digit loop | Any four-digit run |
|---|---|---|---|---|---|
| 2 facts | 100 | 0 | 71 | 29 | 0 |
| 8 facts | 100 | 0 | 89 | 11 | 0 |
| 32 facts | 100 | 0 | 95 | 5 | 0 |
| 128 facts | 100 | 0 | 44 | 56 | 0 |
| Beyond reach, 2 facts | 50 | 0 | 33 | 16 | 0 |
| Beyond reach, 8 facts | 50 | 0 | 46 | 4 | 0 |

- The longer answers did not change the zero. The full model, given the same 48 tokens at 8 facts, found the code in 98 of 100 texts, so the test itself works.
- With 2 candidate codes, the windowed arm picked the right one 28 times in 41 beyond-reach queries (0.68 against a chance of 0.50). With 8 candidates it picked the right one 6 times in 59 (0.10 against a chance of 0.125). The one-sided binomial p-value for 28 of 41 is about 0.014 (**COMPUTED** after the run; not a registered test). The rule's bar for a trace of the code was 0.70 with 2 candidates, so the first result was above chance but below that bar. The second was at the no-trace level. (**FACT**)
- The same ranking, applied descriptively to the two 50-query cells (**FACT**): the windowed arm picked the right code 28 of 50 times at 2 candidates (0.56) and 10 of 50 at 8 candidates (0.20). With sinks, it picked the right code 29 of 50 (0.58) and 12 of 50 (0.24). All are WEAK by the same bars.
- The combined label is PARTIAL. The zero stands, and the ranking signal is weak. The registered rule, written before the run, names three combinations: ZERO STANDS with NO TRACE, BUDGET ARTIFACT, and ZERO STANDS with TRACE. It gives PARTIAL for every other combination. This result is ZERO STANDS with WEAK ranking, so the label follows the rule text. The script printed the same label. (**FACT**)

### 3.3 The weight-edit test

Chosen setting: layer 15, ridge strength 0.1. Test set of 50 codes (**FACT**).

| Measure | Arm S (target at the name) | Arm L (target at "is") |
|---|---|---|
| New codes recalled, of 50 | 0 (95% interval 0 to 7%) | 0 (0 to 7%) |
| Paraphrased questions, of 50 | 0 | 0 |
| Control prompts changed, of 150 (C1) | 0 | 0 |
| Well-known facts changed, of the 30 right before (C2) | 2 | 3 |
| Rise in text surprise (higher is worse) | +0.059%* | +0.089%* |
| Picture answers changed, of 240 (two checks; bar at most 3 each; both arms missed) | 5 and 5 | 8 and 4 |
| Reading | T5-KILL-EDIT | T5-KILL-EDIT |

*Measured against the text-probe value read on this card. Against the value stated in the rules, the rises are +0.050% (arm S) and +0.080% (arm L). Both are under the rule's 0.5% bar.

- On validation, all 16 settings in both arms recalled 0 of 50 codes. The rule's tie-break then chose the lowest ridge strength and the deepest layer that qualified. Every setting tied at zero on the first two steps (validation P0 and paraphrase success), so those did not decide. (**FACT**)
- The edit did change answers. In arm S, under the four settings with the smallest ridge strength, 14 to 29 of 50 validation answers changed (arm L: 16 to 26). None changed into the right code. (**FACT**)
- The fit to the model's own targets was almost exact (error 0.000 to 0.068). The arithmetic worked, but the targets did not carry the code. (**FACT**)
- The chosen edit changed the matrix by about a quarter of its size in arm S (0.25, measured) and by about half in arm L (0.51 on the test set). (**FACT**)

## 4. What it means

What the tests support (**FACT** for the readings):

- In this model and this format, exact copy-type facts from older text were mostly carried by the six cache layers. Limited to a window, the fixed-size memories delivered none of them to the question. With the memories zeroed, recall was 0.87 to 0.97 at 8, 32 and 128 facts, and the full model beat that arm by 11 and 13 texts at 8 and 32 facts. So the memories may add something. The gap cannot be separated from the effect of the last 256 tokens, which are computed from zeroed memories (see section 6).
- A one-step closed-form edit of one weight matrix did not plant new made-up codes in this model. This held for all 16 settings on validation and for both arms on test.

Beyond the tests (**GUESS**):

- A design that kept only fixed-size states would probably not hold exact facts in this format. A bounded design that dropped cache entries would probably drop facts that the memories-zeroed result shows the caches carry.
- The registered decision is to keep the full caches and try compressing them next, with 8-bit keys and values and expandable GPU memory segments (**FACT** for the decision). Expandable segments is a GPU memory allocator option, not a compression method. Evicting cache entries is untested.
- The memories may still hold gist or style. We did not measure that. A design that keeps gist in the memories and exact facts elsewhere would fit this picture, but it has not been built or tested.
- The one-step closed-form target may simply be the wrong target for facts. A gradient search for the target vector, the first step of ROME (**PUBLISHED**: [Meng et al., 2022, "Locating and Editing Factual Associations in GPT"](https://arxiv.org/abs/2202.05262)), is the next route. ROME then writes that vector with a closed-form rank-one update. We have not tried it.

## 5. What it does not show

- That the fixed-size memories store nothing. We measured what they deliver to a question, not what they hold.
- Anything about other model sizes or other models, other fact or question formats, or texts much longer than about 4,000 tokens.
- That a bounded memory cannot work. We tested one window size (256 tokens) and no trained design.
- That no way of writing facts into weights works. We tested one closed-form edit on one kind of matrix, at four layers, with 50 test codes.
- Any speed or memory saving from these ideas. The runs were not built for timing.
- That the zero is only about the memories. Answer length and window damage were tested in part, as the caveats below say.

## 6. Honest caveats

- **Registration limits.** The memory test's rules had six addenda, and the window control's rules had three. The weight-edit rules had two. Each addendum was written on 6 November 2026, before the script it governs ran. No threshold changed. Some readings did:
  - Memory test, addendum 6 (written after a consistency check failed, and before any recall number existed). The check compared two ways of splitting the text, which differ slightly in 16-bit rounding. It was replaced by two checks, one for each split. The subset of facts more than 1,530 tokens back became a reading, which decides between STATE MEMORY and RELAY ONLY. A combined reading that could not occur was removed. Addenda 2 and 5 set the order of overlapping readings and corrected kernel wording.
  - Window control, addendum 3 (written after a data check stopped on one text's total length, 4,075 tokens against 4,000 plus or minus 64). It amended the length rule for one cell. No threshold changed.
  - Weight edit, addendum 1 fixed open points before the script existed. Addendum 2 changed how the picture passes are computed, for speed, and added one check.
  - The window control was registered after the first verdict was written. The answer-length control was registered after a code review had questioned the zero.
- **Reviews.** An AI audit recomputed the headline numbers and table cells of the first memory note: no high-severity findings, 8 medium and 11 low. All were applied, and no number changed. An AI code review of the memory scripts rated two findings high. One: the windowed answers were mostly restated names or digit loops, so the zero might not mean "no fact". That is why the answer-length control was run. The other: the sink wording claimed more than the counts showed, and was narrowed. The review also found a race in the control's run script, which could start a run before its check had passed or read an older results file. The reviewer found no sign that it changed any result, and the later script fixed it.
- **The zero is not fully explained.** The longer answers did not change the zero, but the windowed answers are still degenerate. So we cannot say whether the memories hold nothing, or whether the windowed model never gets as far as writing a code. The ranking test does not depend on generation, and its signal is weak.
- **The gap between the full model and the memories-zeroed arm.** The 11 and 13 extra correct answers of the full model could come from the memories. They could also come from the last 256 tokens (question and filler) being computed from zeroed memories. Of the memories-zeroed arm's wrong answers, 11 of 13 at 8 facts carried another fact's code, and 2 gave a code of no fact. Across the 8, 32 and 128 cells, 25 of 29 carried another fact's code. That favours the first reading, but it does not settle it.
- **Layout.** At 128 facts, 37% of the gaps between facts had no filler sentence between them (4,649 of 12,700; **FACT**, a layout count).
- **Scope.** One model size, one fact format, one question format, one window size, 100 texts per cell (50 in the controls), and one kind of text. The results are about exact recall of four-digit codes from made-up names. The memories may carry gist or style that these tests do not measure.
- **Checks.** The numbers in this draft were checked against the primary result and rule files before publication. That check is not peer review. The weight-edit script was reviewed while its run was still in progress. That review found that the picture-cache check could pass without proving the cached path was taken. The final script kept that check unchanged. The check showed that a cached and an uncached pass give identical outputs for one picture per view, and the edited picture passes took about 6 minutes each instead of about 2 hours 20 minutes, which shows the cache was used in the run. It is not proven for every picture that the cache changed no value.
- Nothing here has been peer reviewed.

## Label legend

- **FACT**: measured by our runs, with the setting and sample size given where we have them.
- **COMPUTED**: arithmetic on measured or stated numbers.
- **PUBLISHED**: from a named published source, with a link where one is given.
- **GUESS**: our reasoning, or anything beyond the tests.
- Verdict words such as CACHE MEMORY, PARTIAL and T5-KILL-EDIT are the written rules' own labels. A missed bar is reported as a miss.

Where a claim could take two labels, we use the more cautious one.
