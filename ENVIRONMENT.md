# Environment

Every figure in this repo was measured here.

## Host

- `ember-gateway`, Ubuntu 24.04
- i7-13700KF, 24 threads, 16 GB RAM installed (2 x 8 GB DDR5), 15.4 GiB usable as `free` reports it
- The 13700KF has no integrated graphics, so the RTX 4070 drives the desktop —
  its idle ~788 MiB is the display, not a stray process.

## GPUs

| Index | Card | Compute | VRAM | Role |
| ---: | :--- | :--- | ---: | :--- |
| 0 | NVIDIA GeForce RTX 4070 | 8.9 (Ada) | 12 GB | ComfyUI / Etsy business. Control card only, when idle. |
| 1 | Tesla V100-PCIE-32GB | 7.0 (Volta) | 32 GB | The subject. |

Driver 580.173.02.

### PCIe link (measured 2026-09-22)

| Card | LnkCap | LnkSta |
| :--- | :--- | :--- |
| RTX 4070 | Speed 16GT/s, Width x16 | Width **x16** (speed idles down to 2.5GT/s) |
| Tesla V100 | Speed 8GT/s, Width x16 | Speed 8GT/s, Width **x4 (downgraded)** |

**The V100 runs at x4**, roughly an eighth of the 4070's host bandwidth. This
does not affect normal inference — weights are resident on the card and compute
never crosses the bus — but it will show in model load times and in any path
that moves tensors to the host per token.

Worth remembering before attributing a slowdown to the V100's age: check
whether the work is actually crossing PCIe first. It was the wrong explanation
for the DeepSeek flash-attention regression (see the `volta-deepseek-mla` repo).

## Toolchain

- CUDA 12.9.86, at `/usr/local/cuda-12.9/bin/nvcc`, passed explicitly because
  nvcc is off PATH. **CUDA 13 dropped Volta**, so 12.x is required.
- nvcc 12.9 now warns that offline compilation for architectures before sm_75
  will be removed in a future release. Volta has a visible end date.
- gcc 13.3.0, cmake 3.28.3

## Build

PrismML fork, `prism` branch, commit `3ae4f51`:

```
cmake -B build -DCMAKE_BUILD_TYPE=Release -DGGML_CUDA=ON \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda-12.9/bin/nvcc \
  -DCMAKE_CUDA_ARCHITECTURES="70;89"
cmake --build build -j 4
```

`-j 4`, not `-j 24`: the host has 16 GB of RAM (15.4 GiB usable) and nvcc is memory-hungry.

Compiled clean, no sm_70 warnings. Load reports 402 Hadamard-folded weights,
all 65/65 layers offloaded, model buffer 6539.67 MiB.

## Isolating a card

`-dev CUDA1` restricts *offloading* but still leaves a ~154 MiB CUDA context on
every visible device. To leave the Etsy card genuinely untouched, hide it:

```
CUDA_VISIBLE_DEVICES=<uuid of the V100>
```

`bonsai-serve` resolves the UUID from the card *name* at start time, so it
keeps working if the cards are ever reordered or swapped.

## Models

| File | Size |
| :--- | ---: |
| `Ternary-Bonsai-2-27B-PQ2_0.gguf` | 7.21 GB |
| `Ternary-Bonsai-2-27B-PTQ1_0.gguf` | 5.95 GB |
| `Ternary-Bonsai-2-27B-mmproj-Q8_0.gguf` | 0.63 GB |

From `prism-ml/Ternary-Bonsai-2-27B-gguf`.
