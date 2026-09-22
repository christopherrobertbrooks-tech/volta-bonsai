# llama-quantize offers PQ2_0/PTQ1_0 but emits no `prism.hadamard.*`; the result loads silently and generates garbage

`llama-quantize` accepts Prism's own formats as output types:

```
141  or  PQ2_0   :  2.13 bpw quantization (group 128, Prism)
143  or  PTQ1_0  :  1.75 bpw ternarization (group 128, Prism)
```

Using one produces a file with **no rotation metadata**. That file then loads
without any warning and generates garbage. The model card says a runtime
"either applies the matching transform or refuses to load the file" — here it
does neither, because there is nothing to detect.

## Reproduce

```bash
./build/bin/llama-quantize Qwen3-4B-F16.gguf Qwen3-4B-PQ2_0.gguf PQ2_0 8
./build/bin/llama-cli -m Qwen3-4B-PQ2_0.gguf -ngl 99 -fa on --temp 0 -n 150 \
  -p "List the first five prime numbers, then explain in one sentence why 9 is not prime."
```

Quantisation takes ~10 seconds and reports normally. The model loads, banners
`ftype : PQ2_0 - 2.13 bpw (group 128)`, and runs at 187 t/s. Output:

```
采购采购[Listтан part2采购プリ2тан QuestionordanABL填2采购 between2采购getList2ansen...
Error: The model produced output that does not match the expected peg-native format
```

## Cause

GGUF metadata comparison:

| file | `prism.hadamard.*` keys |
| :--- | :--- |
| `Qwen3-4B-PQ2_0.gguf` (ours, 28 KV total) | **none** |
| `Ternary-Bonsai-2-27B-PQ2_0.gguf` (yours, 49 KV total) | 10 |

Yours carries `version=1`, `transform=normalized-sylvester-walsh-hadamard`,
`block_size=1024`, `axis=input-last-dimension`, `sign_mode=explicit`, 28,672
`sign_values`, `sign_widths=[5120,6144,17408]`, 401 `weight_names`, and
`inverse_weight_names=['token_embd.weight']`.

So the weights were ternarised in the un-rotated basis, and at load
`src/llama-model.cpp:1197` reads the version key with `required = false`:

```c
if (ml.get_key("prism.hadamard.version", hadamard_version, false)) {
```

Absent, the whole rotation block is skipped silently. Every validation inside it
(block size, transform name, axis, sign mode, sign values) is thorough — but it
is all downstream of a key that is optional, so a file with no rotation at all is
indistinguishable from a file that needs none.

`llama-quant.cpp` contains no Hadamard code, and neither the fork nor
[Bonsai-demo](https://github.com/PrismML-Eng/Bonsai-demo) ships a rotation
pipeline, so there appears to be no supported way to produce a valid file.

## Suggestions

Any one of these would have saved the time:

1. Reject `PQ2_0`/`PTQ1_0` as `llama-quantize` output types while no rotation
   pipeline exists.
2. Warn at load when a PQ2_0/PTQ1_0 model carries no `prism.hadamard.*`, since
   that combination cannot be correct.
3. Emit `sign_mode = identity` metadata from `llama-quantize` if an unrotated
   ternarisation is meant to be usable at all.

## Question

**Is the rotation pipeline available, or planned?** The loader's contract is
specific enough to implement against — and notably it accepts
`sign_mode = "identity"`, which suggests a rotation without explicit sign
vectors is legal. Before attempting that, it would be good to know whether
official tooling is coming.

## Minor

With default settings `token_embd.weight` is quantised to **q6_K**, not ternary.
The model card describes Bonsai as "end-to-end ternary ... no high-precision
escape hatches behind a low-bit label", so the default path here does not match
the released model's construction.

## Environment

Fork `prism` branch, commit `3ae4f51`. CUDA 12.9.86, GCC 13.3.0, Ubuntu 24.04,
Tesla V100-PCIE-32GB (sm_70), driver 580.173.02. Source model
`unsloth/Qwen3-4B-GGUF` BF16, converted locally to F16.

Disclosure: testing and this writeup were done with the help of an AI assistant
(Claude); all figures are taken directly from the run logs on the machine
described above.
