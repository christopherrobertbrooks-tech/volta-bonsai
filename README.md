# Ternary-Bonsai-2-27B on a Tesla V100 (sm_70)

Measured datapoints for a 27B ternary reasoning model on Volta — an
architecture the project does not test. All figures below were taken from run
logs on the machine described in [ENVIRONMENT.md](ENVIRONMENT.md).

Reported upstream as
[PrismML-Eng/llama.cpp#240](https://github.com/PrismML-Eng/llama.cpp/issues/240)
(opened 2026-09-21, follow-up comment the same day).

## What is established

**The ternary kernels work on Volta.** Greedy output is token-identical to the
same build on an RTX 4070 (sm_89), which is inside the tested set. That was the
original report.

**Vision works on Volta**, using the separate `mmproj-Q8_0.gguf` pack. Verified
on a dense five-row table with SKU codes, prices and footnotes: every cell
correct including leading zeros (`07`, `04`, `00`), plus both values in the
small print. Image encode 158–394 ms.

**The two packings disagree between architectures.** This is the finding worth
keeping.

| Card | PQ2_0 tg128 | PQ2_0 pp512 | PTQ1_0 tg128 | PTQ1_0 pp512 |
| :--- | ----------: | ----------: | -----------: | -----------: |
| Tesla V100-PCIE-32GB (sm_70) | **50.97 ± 0.09** | 766.6 ± 6.1 | 34.40 ± 0.04 | **817.3 ± 11.5** |
| RTX 4070 (sm_89, control) | 53.86 ± 0.01 | **1272.4 ± 12.7** | 54.72 ± 0.04 | 619.2 ± 3.8 |

`llama-bench -ngl 99 -fa 1 -p 512 -n 128 -r 5`, one card visible at a time.

The 4070 behaves exactly as the model card documents: PQ2_0 wins prompt
processing by 2.05x, PTQ1_0 edges decode. The V100 inverts both:

- **Prompt processing favours PTQ1_0 on Volta by ~6.6%**, against the model
  card's claim that PQ2_0 wins it *everywhere*. Reproduced with the model order
  swapped and `-r 8` (PTQ1_0 814.9 ± 4.8 vs PQ2_0 724.2 ± 30.2).
- **Volta is the worst case measured for PTQ1_0 decode**, at 67.5% of PQ2_0 —
  below the A100's 74.0%, previously the lowest.

Same box, same binary, same invocation. The disagreement is architectural.

**Therefore: serve PQ2_0 on the V100**, despite the larger footprint. Decode
speed is what you feel interactively, and PTQ1_0 gives up a third of it here.

## Corrections to our own earlier numbers

The prompt-eval figures in the original issue report (132.5 and 358.2 t/s) came
from `llama-cli` on a ~20-token prompt, which is far too short to saturate
either card — roughly 6x low. They were corrected in the follow-up comment.

**Benchmark with `llama-bench`, never with `llama-cli`.**

## Contents

| File | What it is |
| :--- | :--- |
| [RUNBOOK.md](RUNBOOK.md) | How to run it, and the one gotcha that looks like a broken server |
| [bonsai-serve](bonsai-serve) | Serving script: start/stop/status/log, single-instance, card selected by name |
| [issue-240-original-report.md](issue-240-original-report.md) | The original writeup as filed |
| [ENVIRONMENT.md](ENVIRONMENT.md) | Hardware, toolchain, build flags |

## The gotcha, in one line

It thinks by default at `xhigh` effort. If it runs out of output tokens mid-thought
you get an **empty answer** with `finish_reason: length`, which looks like a
broken endpoint. Give it 8000 tokens for hard work. Details in the runbook.
