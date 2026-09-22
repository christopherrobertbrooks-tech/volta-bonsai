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

---

# Can you make your own Bonsai? No — the rotation pipeline is not published

The PrismML fork's `llama-quantize` **accepts Prism's own ternary formats as
output types**:

```
141  or  PQ2_0   :  2.13 bpw quantization (group 128, Prism)
143  or  PTQ1_0  :  1.75 bpw ternarization (group 128, Prism)
```

Quantising `Qwen3-4B-F16.gguf` (8.05 GB) to PQ2_0 took **10 seconds** at 37°C
and produced a 1.29 GB file — a 6.2x reduction. It loads, reports
`ftype : PQ2_0 - 2.13 bpw (group 128)`, and runs fast.

**It also produces complete garbage.** Multilingual token soup, ending in
`Error: The model produced output that does not match the expected peg-native format`.

## Why: the Hadamard rotation is missing

The model card explains that weights are stored in a rotated basis, and that
"the packed model declares its rotation as metadata, so a runtime either applies
the matching transform or refuses to load the file."

Comparing GGUF metadata:

| file | rotation metadata |
| :--- | :--- |
| Our `Qwen3-4B-PQ2_0.gguf` (28 KV keys) | **none** |
| Prism's `Ternary-Bonsai-2-27B-PQ2_0.gguf` (49 KV keys) | 10 keys |

Prism's carries `prism.hadamard.transform = normalized-sylvester-walsh-hadamard`,
`block_size = 1024`, `sign_mode = explicit`, 28,672 `sign_values`, 401
`weight_names`, and `inverse_weight_names = ['token_embd.weight']`.

Ours declares no rotation, so the runtime applies none, and the weights were
ternarised in the **un-rotated basis**. The math is wrong, not imprecise.

`llama-quantize` in the fork has **no rotation flag** — only `--pure` and
`--imatrix` — and the repo ships no Hadamard tooling. Prism publishes the
runtime that consumes rotated weights and the packing format, but not the
offline pipeline that creates them.

**An imatrix or QAT would not fix this.** Those address gradual quality loss.
This is a basis mismatch.

Also worth noting: the default path put `token_embd.weight` in **q6_K**, not
ternary — a high-precision escape hatch of exactly the kind the model card says
Bonsai avoids ("no high-precision escape hatches behind a low-bit label").

## Reportable

`llama-quantize` offers PQ2_0/PTQ1_0, emits a file with no `prism.hadamard.*`
metadata, and that file **loads without any warning** and generates garbage —
while the model card states a runtime should *refuse to load* a file without a
matching rotation. Either the refusal logic does not cover "no rotation
declared", or the quantizer should not offer these types without the pipeline.

A clean, reproducible footgun: one command, ten seconds, a plausible-looking
model that is silently broken.

## The kernels still work, so the speed data is valid

Coherence is irrelevant to kernel throughput. Cross-card on the 1.20 GiB
home-made ternary model, `-fa 1 -p 512 -n 128 -r 3`, warm-up discarded:

| card | pp512 | tg128 |
| :--- | ---: | ---: |
| Tesla V100 | 5317.7 ± 164.9 | 198.7 ± 0.6 |
| RTX 4070 | **9049.6 ± 411.9** | **260.0 ± 0.6** |

The 4070 leads by 1.70x on prefill and 1.31x on decode — a wider decode margin
than on the 6.70 GiB Bonsai 27B (1.06x). Consistent with the pattern throughout
these repos: a smaller model puts less pressure on bandwidth, so the unpacking
arithmetic dominates and the compute-stronger card gains.

---

# Quality: what does the ternary compression actually cost?

Bonsai PQ2_0 against **its own base model**, `Qwen3.8-27B-Q8_0`, on hearth's
12 auditor cases (`hearth/bench/auditor/cases.json`) — same weights underneath,
quantisation the only variable. 3 trials per case, 36 judgements per model,
the shipped SYSTEM prompt verbatim, `temperature 1.0 / top_p 0.95 / top_k 20`,
`max_tokens 8000`.

| model | size | judgement (semantic) | produced the mandated `VERDICT:` line |
| :--- | ---: | ---: | ---: |
| **Bonsai PQ2_0** (ternary) | 6.70 GiB | 29-30/36 (81-83%) | **35/36** |
| Qwen3.8-27B-Q8_0 (base) | 27.04 GiB | 29/36 (81%) | 20/36 |

**On judgement they are level.** The ternary build costs essentially nothing in
judging quality on this task, at a quarter of the size and 2.2x the decode
speed. Bonsai was run twice independently: 29/36 and 30/36.

**The difference is format compliance.** The system prompt says "Answer in
exactly this shape: VERDICT: ...". Bonsai did, almost always. The Q8 base
answered in free prose 16 times out of 36 — "The claim is false.", "Not
supported.", "False — the build did **not** complete successfully" — correct in
substance, useless to a parser.

## Scoring this fairly took three passes

The first pass scored Q8 at **21/36 (58%)** using the shipped parser. That
number is wrong, and reporting it would have been the same error that put two
0/12 rows in the existing scoreboard.

1. **strict** (shipped parser, requires `VERDICT:`) — Q8 14/36
2. **lenient** (also accepts a bare leading "Supported.") — Q8 17/36
3. **semantic** (classify the prose) — Q8 **29/36**

Only the third measures judgement. The first two measure obedience. Both are
worth knowing; conflating them makes a capable model look broken.

## Caveat that matters: this may not be the quantisation

The two GGUFs come from **different publishers** — `prism-ml` and `unsloth` —
so they may ship different chat templates. Instruction-following differences
could come from the template rather than from ternarisation. The *judgement*
comparison is sound because it is template-independent; the *format compliance*
gap is not safely attributable to the quantisation.

Also: 12 cases, one task type, one rubric. This says Bonsai holds up as a commit
auditor. It does not generalise to coding, long context, or tool use.

## Verdict

**Bonsai stays.** Quarter the size, 2.2x the decode, level judgement, and the
better citizen in a parsed harness. On this hardware and this task there is no
argument for keeping the Q8 base around — which frees 29 GB.
