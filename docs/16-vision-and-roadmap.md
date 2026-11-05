# 16. The goal: an always-learning, self-improving model, and the road to it

Status: a plan and a snapshot, written 2 November 2026. Everything labelled GUESS is untested. Everything labelled FACT points to a result in another document of this repository.

## The end goal (the project owner's description, tidied)

A small language model whose brain is never switched off:

- It keeps learning while it talks to someone, while it does a task, while it idles, and while it "sleeps". Its weights are allowed to change at any of those times.
- It has three working states: **active** (talking or acting), **idle** (thinking quietly, going over what it has seen), and **sleeping** (consolidating short-term memory into long-term memory, as people seem to do).
- Its thinking stays **hidden inside the model**, the way a person thinks in their head, optionally jotting a few notes on scratch paper, and then writes only the final answer on the "chalkboard". People who want to look inside would use a separate viewer built on interpretability tools (a logit lens, or a Jacobian-style lens that shows words the model is considering but not saying). This is deliberately different from models that print their reasoning as visible text.
- If it notices that it has become worse at something, it should **repair** that skill, not roll back to an older copy, because a rollback also throws away what it learned since. It should in some sense be aware of its own competence.
- Its memory should live **in its weights or in a separate compartment that is part of the model**, not in a context that is fed back in again and again. Even the working memory should be cheap enough to keep on a memory card or a disk without making the model slow.
- It should improve its own parts: its tokenizer, its compression, its memory system, and (carefully) its own code.
- It should be **small**, because a small model can run on the user's own hardware and can afford to keep learning. That is why compression is a main research thread (document 14).
- It should ship in a generic state and **tailor itself to its user in its weights**, rather than carrying the user's history in a prompt.
- Later, a body: eyes (image understanding and tool use), ears (speech recognition), a voice, and an "imagination" (image generation used as internal imagery).

## Where each part stands

| Part | Plan | Status (2 Nov 2026) |
|---|---|---|
| Learning while reading | one small update after each stretch, replay of surprising stretches | FACT: reading lowered loss on the rest of a book by 2.4 to 4.6 percent; a foreign-book control made it worse (one control, two books). Whether this is knowledge of the book is open: the quiz was inconclusive. About 60 percent of the gain survived one or two later books (documents 08 and 12) |
| Not forgetting | mix old material into new training; consolidate during sleep | FACT: mixing 20 percent ordinary text into a puzzle fine-tune kept normal-text quality (1.0773 against 1.0817 bits per byte) where pure puzzle training lost 5.4 percent (document 15). GUESS: this scales to continuous learning |
| Repair instead of rollback | a health monitor with held-out skill probes and targeted replay | FACT (toy): repair kept both skills where rollback lost the new one, but the toy's test examples were also in its training set, so it shows retention of memorised cases only. FACT (real model, document 18): a monitor-driven repair as first built barely learned the new skill (2 percent recall on the seed shown), and rollback learned nothing (0 percent). A different design, learning first and then a short healing phase that is mostly rehearsal, learned the new skill (99 to 100 percent) and kept the old skills near their starting values, for 3 to 12 new skills in a row on one or two seeds. The new skills were memorised facts, not reasoning |
| Sleep and consolidation | a fast adapter learns while awake; sleep distils it into the base and empties it | FACT: on a toy the adapters were emptied again and both facts were still recalled. The toy is too easy to prove anything. GUESS for real models |
| Hidden thinking | silent latent steps plus a small private scratch memory; answers accepted only when an exact checker agrees | GUESS. Needs a base model that can already do something checkable. The Countdown puzzle task (document 15) is now above the minimum bar for starting this |
| Reading the hidden thinking | lens viewers | FACT: a viewer exists; its lens readouts are not yet reliable (split-half agreement 0.34 and 0.55 against a pass bar of 0.8) |
| Memory in weights | a small fast-learning compartment, consolidated into the base | GUESS. Not built |
| Own tokenizer | a byte-level model that learns how to group bytes | GUESS. Researched, not built; planned as a compression experiment |
| Small enough | ternary weights, smaller embedding tables, quantization | FACT: ternary training was slightly better than normal training at 24 million tokens on three seeds, and slightly worse at 12 million (document 15). FACT: without retraining, 6 bits per weight costs 1.7 percent quality and anything lower breaks (document 14). The shrinking of the large lookup tables is not done |
| Long memory on a card or disk | spill old key and value blocks to storage, keep a small hot part in fast memory | GUESS. A design and a benchmark are drafted; nothing measured on real storage |
| Per-user tailoring | a bounded per-user adapter on a frozen shared base | GUESS. Not built |
| Self-improving code | an automated propose-test-keep loop | PARTIAL: our process already works this way with written-down-first rules, but a human approves every change to the production model |
| Eyes, ears, voice, imagination | small encoders, first built from scratch; later the from-scratch rule was lifted for some parts | FACT (document 19): a 2.24-million-parameter image model reached 90.4 and 90.1 percent on CIFAR-10 on two seeds, and a small bridge let the frozen language model name the pictured class 90.5 percent of the time with its text behaviour unchanged. The ears (7.1 million parameters, from scratch) reached a 24.6 percent word error rate on held-out clean speech, and a first imagination drew the right kind of picture from the language model's own state 84 percent of the time on held-out sentences, as judged by the frozen image model of section 1 (about 90 percent accurate on real pictures; 32 by 32 pictures, ten objects, templated sentences; document 19 has the limits and the failed first attempts at topic changes). A from-scratch text-to-speech voice was built, then replaced by the imported Bark-small after a listening test; a frozen 360M SmolLM2 brain was given pretrained SigLIP and Whisper connectors (document 19, sections 3g to 3i; document 20). The 143M model is not the brain in those tests |

## Hard problems we expect

1. **Forgetting.** Learning during use damages old skills. The evidence we have for a cure is replay (mixing old data in); whether that is enough for months of continuous learning is open.
2. **Grading its own homework.** A model that checks its own competence could learn to pass the check without being better. The checks must be tests it cannot train on or edit, with a final sealed test set it never sees.
3. **Self-modification.** Anything that edits its own code needs an evaluator and a rules file it cannot change, plus a human sign-off for anything touching safety or the tests themselves.
4. **Changing the tokenizer** changes every embedding. A byte-level model with learned grouping avoids retraining a vocabulary table.
5. **Slow storage.** A disk or card is far slower than memory; the plan only works if a small hot part does most of the work.
6. **Hidden thinking is hard to audit.** Hiding thoughts makes it harder to see what the model is doing. The viewer helps only if it is reliable, and exact checkers on the answers are the backstop.

## The order we plan to test

1. Repair versus rollback on a real model (done at small scale for memorised facts, document 18; next: a skill that needs reasoning, and a larger model).
2. A sleep phase for the reading model, against plain replay.
3. A byte-level model against the word-piece model, using compressed size as the score.
4. Hidden thinking with an exact checker on Countdown, with the lens as the viewer.
5. Shrinking the model to a few tens of megabytes (ternary training plus smaller lookup tables).
6. Per-user adaptation on a synthetic user, with a wrong-user control.

## What this document does not claim

No part of the goal beyond the measured items above has been shown to work. This is a direction and a testing plan, not a result.
