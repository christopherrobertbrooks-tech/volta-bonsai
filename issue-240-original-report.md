# Report: Ternary kernels work on Volta (sm_70) — token-identical with sm_89, ~50 t/s on a V100

Not a bug — a datapoint for an architecture outside the tested set
(`75-virtual;86-real;89-real;120-real`). Since the docs describe missing
architectures as "untested, not merely slower," here is a tested one.

## Result

**Ternary-Bonsai-2-27B (PQ2_0) runs correctly on a Tesla V100 (compute 7.0)**
built from source against CUDA 12.9. Greedy (temp 0, 600 tokens) output is
**token-identical** to the same build running on an RTX 4070 (compute 8.9,
inside the tested set) — diff of the two outputs differs only in the
loading spinner and the perf banner. Reasoning trace and final answer both
coherent.

| GPU | compute cap | prompt eval | generation |
|---|---|---|---|
| Tesla V100-PCIE-32GB | 7.0 | 132.5 t/s | 50.5 t/s |
| RTX 4070 (control)   | 8.9 | 358.2 t/s | 52.9 t/s |

Generation speed is effectively at parity (memory-bound; HBM2 holds up).
Prefill is ~2.7x faster on Ada.

## Environment

- Fork: `prism` branch, commit `3ae4f51` (version 0.2.0-dev, build 1)
- Model: `prism-ml/Ternary-Bonsai-2-27B-gguf` — `Ternary-Bonsai-2-27B-PQ2_0.gguf` (7.2 GB)
- GPU: Tesla V100-PCIE-32GB (Volta, sm_70), driver 580.173.02
- CUDA toolkit 12.9.86 (nvcc off PATH; passed explicitly — see below).
  Note this route dies with CUDA 13, which dropped Volta support.
- gcc 13.3.0, cmake 3.28.3, Ubuntu 24.04, host: i7-13700KF / 15 GB RAM

## Build

```
cmake -B build -DCMAKE_BUILD_TYPE=Release -DGGML_CUDA=ON \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda-12.9/bin/nvcc \
  -DCMAKE_CUDA_ARCHITECTURES="70;89"
cmake --build build -j 4
```

Compiled clean — no sm_70-related warnings or errors. Load reports
`loaded 402 Hadamard-folded weight(s) (1 inverse-lookup) using 1 rotation(s)
and 3 sign vector(s)`; all 65/65 layers offloaded; model buffer 6539.67 MiB.

## Test

```
CUDA_VISIBLE_DEVICES=<n> ./build/bin/llama-cli -m Ternary-Bonsai-2-27B-PQ2_0.gguf \
  -ngl 99 --temp 0 -n 600 --single-turn \
  -p "List the first five prime numbers, then explain in one sentence why 9 is not prime."
```

Ran once per card; diffed the outputs. Correct answer on both
(`2, 3, 5, 7, 11` + a correct sentence about 9 = 3 × 3).

Happy to run additional validation (perplexity run, longer contexts, or
PTQ1_0 — untested here) if useful. Filed in case it saves the next person
with a Volta card the guesswork; also fine to close if sm_70 remains
officially unsupported.

---
*Disclosure: testing and this writeup were done with the help of an AI assistant (Claude); all figures are taken directly from the run logs on the machine described above.*
