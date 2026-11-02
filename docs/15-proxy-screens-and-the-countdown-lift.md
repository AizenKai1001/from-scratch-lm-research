# 15. Proxy screens and the Countdown lift

Snapshot dated 2 November 2026. Each screen had a rule file with its pass and kill conditions written before its first run, and each rule file has a checksum recorded (document 12 lists them). Labels follow the README: **measured**, **one run**, and **guess** where we reason without testing. Losses are held-out bits per byte, so a negative effect means better.

## Summary

On a small proxy model, ternary weights (each weight restricted to -1, 0 or +1) were slightly worse at 12 million training tokens and slightly better at 24 million, on three seeds. Spiking activations stayed level, and looped layers did not clearly help. Of seven efficiency tricks, only an extra next-token prediction head met the joint rule (the same sign on all four seeds and a mean of at least 0.02 better), at about 1.5 times the time. A copy of the base model fine-tuned on programmatically solved Countdown puzzles went from 0 of 500 solved to 48.2 percent solved in 64 tries; mixing in ordinary text kept its normal-text skill.

## 1. Ternary, spiking and looped variants

**Setting.** A proxy with width 256 and 6 layers, trained on 12 million tokens, compared with a normal-weight base using the same seed. Ternary here means the forward pass rounds full-precision weights to -1, 0 or +1; there is no packed format and no speed gain. Spiking means integrate-and-fire activations over 4 steps, run as dense maths. Looped means the two middle blocks are run twice with the same weights (same parameter count, about a third more compute).

**12 million tokens, three seeds (1, 3, 5), effect against base:**

| Variant | Seed 1 | Seed 3 | Seed 5 | Verdict (pre-written rule) |
|---|---|---|---|---|
| Ternary | +0.034 | -0.051 | +0.041 | Slightly worse (one seed above +0.034) |
| Spiking | +0.004 | +0.021 | +0.012 | Competitive (no clear loss) |
| Looped | -0.001 | +0.052 | -0.006 | Slightly worse (one seed above +0.034) |

**24 million tokens, three seeds, about 21.5 million parameters:**

| Seed | Normal | Ternary | Effect |
|---|---|---|---|
| 1 | 1.9109 | 1.9022 | -0.0087 |
| 3 | 2.1453 | 2.1375 | -0.0078 |
| 5 | 1.9131 | 1.8993 | -0.0138 |

All three effects are negative, and the mean is -0.010. The rule (all negative, mean at or below -0.005) reads this as ternary slightly better. A single seed-1 run at 24 million tokens gave spiking +0.028 and looped +0.001; these are indications from one run only.

Two caveats. The ternary training used full-precision weights with rounding in the forward pass, so its gain in size and speed is not yet measured. And the planned 30-million-token runs failed without a result: each was cut off by a 1,000-second time limit that was too short for runs of that length (they need roughly 17 to 23 minutes). We reran a smaller 24-million-token set with a 25-minute limit, after adding a dated note to the rules before it started. The 24-million-token comparison above is that rerun.

## 2. Efficiency tricks

Seven arms, each one change from the same base, at 12 million tokens. Seeds 1 and 3 ran on two home graphics cards; seeds 5 and 7 ran on a pair of cloud T4 GPUs. Base losses were 2.1745 and 2.5816 at home, and 2.0203 and 3.0011 on the T4. Seed 5 gave bit-identical base results on both kinds of hardware, so the two setups agree. Seed 7 is noisy, with spreads of 0.1 to 0.27 bits per byte between arms.

| Arm | Change | Seeds 1, 3 (home) | Seeds 5, 7 (T4) | Verdict |
|---|---|---|---|---|
| A | Shorter cooldown | +0.0074, -0.0003 | -0.0069, +0.0215 | Fail |
| B | NorMuon optimizer | -0.0015, +0.0343 | -0.0012, +0.0132 | Fail |
| C | Cautious weight decay | -0.0008, +0.0124 | -0.0015, -0.0046 | Fail |
| D | Polar Express | -0.0027, +0.0712 | -0.0019, +0.1234 | Fail |
| E | Hashed bigram table | -0.0004, -0.0027 | -0.0007, -0.0064 | Fail (no effect) |
| F | One extra prediction head | -0.0173, -0.0276 | -0.0156, -0.1276 | Robust under the joint rule; fails speed |
| G | Weight averaging | 0, 0 | 0, 0 | Invalid (see below) |

Only F met the joint rule. (E, the hashed bigram table, was also slightly negative on all four seeds, with a mean of -0.003, far inside the noise.) F's mean effect is -0.047 across the four seeds, and -0.020 without the noisy seed 7. The joint rule (same sign on all four, mean at or below -0.02) is met. The speed condition is not: on the T4 the run took 1,212 seconds against 782 for the base, about 1.5 times as long. Timing on the home cards is not recorded. So F is a real loss gain per step, but at equal wall-clock time it may not win. A test at equal time is the next step (not run).

Arm G was invalid. Its results matched the base exactly on every seed, because the script never evaluated the averaged weights in this setup. It is a no-op and gives no evidence either way.

## 3. Countdown puzzles

**The task.** Four numbers and a target from 10 to 99; each given number may be used at most once; a program checks the answer. The base model solved 0 of 500 held-out puzzles with 64 tries each. That was below the 5 percent floor written beforehand, so reward training waited.

**The training.** A copy of the base model was fine-tuned for 800 steps (batch 64, learning rate 2e-5, about 3.5 minutes) on 60,000 puzzle solutions made by a brute-force program. There was no AI-written text, and the held-out number sets were excluded.

| Model | Puzzles | Solved at pass@1 | Solved at pass@64 | Normal text, bits per byte |
|---|---|---|---|---|
| Base | 500 | 0% | 0 of 500 | 1.0817 |
| Copy, puzzles only | 500 | 4.45% | 48.2% (241 of 500) | 1.1403 |
| Copy, puzzles only | first 100 | 4.1% | 53 of 100 | not separate |
| Copy, 20% ordinary text mixed in | first 100 | 3.4% | 41 of 100 | 1.0773 |

The floor was passed, so the copy is a lift by the rule. Without replay, normal-text loss got worse by 0.0586 bits per byte (5.4 percent). With 20 percent ordinary text mixed into the fine-tune, normal text came out at 1.0773, below the base, and the puzzle score on the first 100 puzzles fell from 53 to 41 of 100. The replay text came from the training file, so the 1.0773 shows that the old skill was kept, not that a new one was learned.

**Caveats.** The held-out puzzles share the training format and number range, so this shows the model can pick up this task, not general reasoning. The replay result covers 100 puzzles, not 500. Pass@64 means at least one of 64 samples was right. The copy is not a production model, and no other tasks were measured.

## What this does not show

- **Transfer to the full model.** The screens used small proxies (the 24-million-token ternary comparison used about 21.5 million parameters). The 142.9-million-parameter model may behave differently.
- **Firm rankings from few seeds.** Most arms have two or four seeds, seed 7 is noisy, and small effects near the rule's bars are not separated from noise.
- **Size or speed gains.** Ternary and spiking ran as dense maths, so no gain of that kind was measured.
- **General reasoning.** The Countdown result is one narrow task with a matching format.
