# 14. Compression with the model

Snapshot dated 2 November 2026. Measured numbers come from our logged runs, with the setting given, one run each unless stated. Labels: **FACT** means measured in our runs; **GUESS** means our reasoning, not tested.

## Summary

A model that predicts the next byte well can also be a lossless compressor. Our 142.9-million-parameter base model, paired with a standard arithmetic coder, stores a 2.29 MB held-out text in 314,132 bytes. The best classic tools we ran (bzip2 and xz) needed about 692,000 bytes for the same text, so the coded output is about 2.2 times smaller. The cost is the model itself. Its weights take 286 MB at half precision, so the method only pays over very large or very many files. The headline number leaves that decoder out.

## The idea

A coder that gives each next symbol probability p spends about -log2(p) bits on it, and an arithmetic coder gets close to the sum of those costs. A model that predicts the text better therefore produces a shorter file. The encoder and decoder run the same model in the same order, so the decoder recomputes every probability from the bytes it has already decoded.

The score is bits per byte (bpb). On the same held-out text, the model's ideal code length (what the coder would reach with no overhead) is 1.082 bpb, or 309,082 bytes for 2,285,959 bytes of text (FACT, from our scoring script). The real coder is the test of that estimate.

## Measured results

Settings: a session-92 checkpoint of the chained training run (the model is unfinished), 256-token windows, 64 parallel segments for the full file, and an exact decode check after each run. Sizes are in bytes.

| Sample | Original | Our coder | xz -9e | bzip2 -9 | Decode check |
|---|---|---|---|---|---|
| First 30 KB | 30,000 | 4,218 (1.125 bpb) | not run | not run | exact |
| First 200 KB | 200,000 | 27,036 (1.081 bpb) | 68,084 | 64,793 | exact |
| Full held-out text | 2,285,959 | 314,132 (1.10 bpb) | 696,600 | 692,138 | byte-identical |

The full file is about 2.2 times smaller than the best classic tool. Encoding it took 306 seconds. Other settings that changed the numbers:

- **Probability precision.** The probability total was raised from 2^20 to 2^24 steps. On the 200 KB sample the output fell from 27,036 to 26,921 bytes (0.4 percent), with an exact decode check. The full-file run came before this change.
- **A small file.** A 1,500-byte README coded to 368 bytes (1.96 bpb), against 814 for gzip -9 and 908 for xz -9e. The model's size was not counted, and a file this small cannot pay for a model.

**Speed, and how it improved (FACT):**

1. A pure-Python loop that turned model outputs into coder probabilities was the first bottleneck: 3 KB took 366 seconds. Vectorizing that step with numpy cut it to 11 seconds, with identical output.
2. The model recomputed each 256-token window for every byte. A key-value cache (it keeps earlier attention results, so each new byte costs one new step), with the probability conversion moved to the GPU, cut the 200 KB encode from 923 to 53 seconds (about 17 times faster) and the decode from 463 to 33 seconds. Against a full recomputation, the largest logit difference over 601 tokens was 2.2e-5.
3. A first try with 1,024-token windows and 32 segments ran out of GPU memory after 51 minutes. The windows were reduced to 256 tokens.

## The decoder problem

The model has 142.9 million parameters: 59.0 million in the transformer block weights, and 83.9 million in four tables of about 21 million each (token embedding, output head, and two value-embedding tables). That is 286 MB at half precision and 572 MB at full precision.

We rounded the weights after training, with no retraining, and measured on 923 KB of held-out text (full-precision baseline 1.0738 bpb) (FACT):

| Setting | Bits per byte | Change | Packed size |
|---|---|---|---|
| int8, all weights | 1.0749 | +0.001 | 143 MB |
| int8 tables, int6 blocks | 1.0810 | +0.007 | 128 MB |
| int6, all weights | 1.0926 | +0.019 | 108 MB |
| int4 tables, int8 blocks | 1.3604 | +0.287 | 101 MB |
| int3 tables, int4 blocks | 1.4816 | +0.408 | 61 MB |
| int4, all weights | 1.5592 | +0.485 | 72 MB |

Without retraining, 6 bits per weight is the floor: quality is about 1.7 percent worse at 108 MB, and below that it collapses. Getting under about 60 MB would need training at low precision. A proxy test (document 15) found that ternary training matched normal training at 24 million tokens, but this has not been tried on the full model.

**Break-even (GUESS, arithmetic on the figures above).** A 2.29 MB text saves about 380 KB against bzip2. With the 108 MB decoder, the saving pays the decoder back after about 280 such files (about 640 MB of text). With the 286 MB half-precision decoder it takes about 750 files (about 1.7 GB). One file of the same size never pays back. These figures assume the unrounded model's bits per byte.

## Review findings

A separate AI review of the coder code raised three points. The archive's format tag was not unique. Some settings the decoder needs were not written into the archive header. And the savings figures were estimates from ideal code length rather than sizes of real archives. At this snapshot the archive tag and the missing header settings are NOT yet fixed (a new archive format that records them is being built); only a check that rejects non-finite model outputs was added. The measured archive sizes above are the figures to use, and archives written by the current coder must be decoded with the same settings and the same code version (files written before the precision change will not decode with the current code).

**Ideal estimates, not coder runs (FACT, from the scoring script):** a 2 MB extract of one novel has an ideal rate of 1.258 bpb, 1.47 times smaller than the best classic tool (461,436 bytes). A 116 KB Python source file has 2.037 bpb, 1.12 times smaller than 33,021 bytes.

## What this does not show

- **That the method works well in general.** Coder runs cover one held-out text from our own data and one small README. The other figures are ideal estimates.
- **A finished model.** Its final cooldown and remaining training are not done, so every number here will change.
- **The total size with a packed decoder.** The 314,132-byte result leaves the model out. No packed decoder has been checked for a round trip.
- **Speed elsewhere, or the full decode time** (not recorded), **or results against published neural compressors** on standard benchmarks (none run).
- **Robustness.** Each setting was run once.
