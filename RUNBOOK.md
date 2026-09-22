# Bonsai 2 27B on ember-gateway

## Running it

    ~/bonsai/bonsai-serve start     # start on the V100
    ~/bonsai/bonsai-serve status    # is it up? how much VRAM?
    ~/bonsai/bonsai-serve stop      # stop it, frees the VRAM
    ~/bonsai/bonsai-serve log       # follow the log

Endpoint: `http://127.0.0.1:8030/v1/chat/completions` (OpenAI-compatible).
Port 8030 was chosen to avoid ComfyUI (8188/8189), stats (8020) and 8443.

It refuses to start twice, and `stop` confirms the process is actually gone
before it returns. It selects the V100 by name, not by index, so it keeps
working if the cards are ever swapped around.

## The one gotcha

This model thinks before answering, at `xhigh` effort by default. If it runs
out of output tokens while still thinking, you get an **empty answer** with
`finish_reason: length`. That looks like a broken server but just means the
budget was too small. Set `max_tokens` to 8000 for hard questions.

`reasoning_effort` accepts `xhigh`, `medium`, `low` — `none` returns a 500.
`{"chat_template_kwargs": {"enable_thinking": false}}` turns thinking off and
is much faster on easy prompts.

## Measured (llama-bench -p 512 -n 128 -r 5, one card visible at a time)

| Card              | PQ2_0 tg128 | PQ2_0 pp512 | PTQ1_0 tg128 | PTQ1_0 pp512 |
| ----------------- | ----------: | ----------: | -----------: | -----------: |
| V100 (sm_70)      | **50.97**   | 766.6       | 34.40        | **817.3**    |
| RTX 4070 (sm_89)  | 53.86       | **1272.4**  | **54.72**    | 619.2        |

PQ2_0 is the serving default on the V100 because decode speed is what you feel
when typing at it, and PTQ1_0 gives up a third of it there.

The two cards disagree, which is the interesting part. The 4070 behaves exactly
as the model card documents (PQ2_0 wins prompt processing 2.05x, PTQ1_0 edges
decode). The V100 inverts both: PTQ1_0 wins prompt processing by ~7%, against
the card's claim that PQ2_0 wins it "everywhere", and PTQ1_0 decode drops to
67% of PQ2_0 — worse than the A100's 74%, the previous worst measured.

Reported upstream at PrismML-Eng/llama.cpp#240 on 2026-09-21.

Note: if you ever compare against numbers from `llama-cli`, don't. A short
interactive prompt does not saturate either card and reports prefill roughly
6x too low. Use `llama-bench`.

## Vision

Works on the V100, using the separate `Ternary-Bonsai-2-27B-mmproj-Q8_0.gguf`
(629 MB, downloaded 2026-09-21). Read a dense table correctly including
leading zeros and small print. Image encode 158-394 ms. The server loads the
mmproj automatically if the file is present.

## Files

    ~/bonsai/Ternary-Bonsai-2-27B-PQ2_0.gguf        7.2 GB  (serving default)
    ~/bonsai/Ternary-Bonsai-2-27B-PTQ1_0.gguf       5.9 GB  (smaller, slower decode here)
    ~/bonsai/Ternary-Bonsai-2-27B-mmproj-Q8_0.gguf  629 MB  (vision)
