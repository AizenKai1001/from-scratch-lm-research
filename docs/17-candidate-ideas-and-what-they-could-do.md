# 17. Candidate ideas, and what each could do for the project

Snapshot dated 2 November 2026. This is a list of ideas that came out of a broad literature and design review, with our reasoning about what each could give us. **None of these is a result.** Every item is a GUESS or UNVERIFIED unless it points to a measured result in another document. The review itself was done by AI assistants, so the sources behind each idea still need to be opened and checked by a person before we rely on them. Every idea was judged "worth a small test, but revise the test first"; none was judged ready to run as written.

## How to read an entry

Each entry says what the idea is, what it could do for us if it works, what it would cost, and what the first test would be. "Could do" is a hope, not a prediction.

## A. Compression and decoder size (document 14)

| Idea | What it could do | Cost and risk | First test |
|---|---|---|---|
| Choose the vocabulary size by total archive bytes (data plus the cost of storing the vocabulary) | A smaller total file for the same text, and a smaller model table | Cheap to measure with a simple coder; the model must be retrained to use a new vocabulary | Sweep vocabulary sizes on a text sample with an exact decode check |
| Blend the model with simple helpers (a repeated-phrase matcher, word-pair counts, a small probability corrector) | A few percent smaller output on text with repeats, such as code, logs and books | The blend must use integer arithmetic so the encoder and decoder agree exactly; slower | Compare model alone against model plus helpers on held-out text |
| Adapt the model while compressing | A larger gain on one big file, because the model learns that file | Several times slower; the decoder must repeat every update exactly | A single-file test with an exact decode check |
| Shrink the large lookup tables (factor each into two thin ones; store weights in fewer bits) | A decoder several times smaller, which is what makes the method pay off | Quality may drop; needs training that knows the weights will be rounded | Rounding-aware training of a small proxy, then a size and quality check |
| A byte-level model with learned grouping of bytes | Removes the fixed word list, and lets the grouping improve with the model | More complex; may lose to a plain model at small size | Plain bytes first, then learned groups, judged by total compressed size |

## B. Learning without forgetting (documents 08 and 16)

| Idea | What it could do | Cost and risk | First test |
|---|---|---|---|
| Replay old material while learning new (measured in document 15 for Countdown) | Keeps old skills while adding new ones | Slightly slower learning of the new skill | Already shown at small scale; next, at several replay ratios |
| Anchor updates to the original model | A hard limit on how far the model can drift | May slow real learning | Compare drift and new-skill gain with and without the anchor |
| A small fast-learning memory that sleep folds into the permanent weights | Memory in the weights without a growing context | The folding step could lose information | A two-book test: wake-only against replay against sleep |
| A per-user adapter on a frozen shared model, with a different user's adapter as a control | Tailoring to each user in the weights | Needs proof that the gain is really about that user | A synthetic-user test with a wrong-user control |
| Repair a lost skill by targeted replay instead of restoring an old copy | Keeps new learning when something gets worse | Could flip back and forth without damping | Real-model test of repair against rollback (planned) |

## C. Model and training ideas

| Idea | What it could do | Cost and risk | First test |
|---|---|---|---|
| Ternary weights (measured: document 15) | A much smaller decoder at no measured quality loss at 24 million tokens | Only tested on a small proxy | Larger proxy, then packed storage |
| One extra prediction head (measured: document 15) | A small gain in loss | Slower per step, so it may not win at equal time | An equal-time test (running) |
| Tie the input and output tables, grouped-query attention, a different MLP | A smaller or faster model at the same quality | Each may cost a little quality | Single-change proxy screens with pre-written rules |
| Silent thinking steps before an answer, checked by an exact grader | Better answers on checkable tasks without writing out the reasoning | Needs a base model that can already do the task (Countdown now is above the bar) | Pause-token control, then silent steps |

## D. Safety and process

| Idea | What it could do | Cost and risk | First test |
|---|---|---|---|
| Keep the judge outside anything the model can edit, plus a sealed final test set | Stops a self-improving system from passing its own checks without improving | More setup | Red-team the exact Countdown checker before training against it |
| Rules that are checked by software, with checksums chained together | Makes after-the-fact rule changes visible | None of note | In use for our recent screens |

## What would change our direction

If the compression helpers and a smaller decoder both work, the model-as-compressor idea moves from a curiosity to something that could pay off on large collections. If silent thinking works on Countdown, the "thinks in its head, writes the answer" design becomes testable on harder checkable tasks. If repair beats rollback on a real model, continuous learning becomes safer to try. If none of them move the numbers, the plain model with replay is still a sound baseline.

## What this document does not show

Nothing here has been measured except where a row points to another document. The ideas come from AI-assisted reading, so some may duplicate each other, some may already be known to fail at our size, and some sources may be mis-summarized.
