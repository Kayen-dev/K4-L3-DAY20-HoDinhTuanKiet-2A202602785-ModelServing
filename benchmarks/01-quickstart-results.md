# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 7108 | 758 / 867 | 50.3 / 56.2 | 3927 / 4078 / 4078 | 19.9 |
| UD-Q2_K_XL | 0.39 | 5522 | 1036 / 1119 | 298.9 / 302.9 | 19886 / 20179 / 20179 | 3.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **6.03x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

This is my first complete benchmark run after setup. The script discarded one warm-up
request per quantization, and all 10/10 measured requests succeeded. `UD-Q2_K_XL` is
0.39 GB versus 0.50 GB for `Q4_K_M`, so it saves 0.11 GB (22%) and loads about 22%
faster. However, its decode rate is only 3.3 tok/s versus 19.9 tok/s: the 2-bit model is
6.03x slower, with TPOT P50 increasing from 50.3 ms to 298.9 ms. Its TTFT P50 is also
worse (1036 ms versus 758 ms).

This machine has four physical CPU cores and a low-end 2 GB GeForce MX110 using Vulkan
offload. The result is compute/dequantization-limited rather than memory-bandwidth-
limited: the extra work required by the heavily quantized format costs more than the
bytes it saves.

For the same quality prompt, `Q4_K_M` stayed on the LLM-serving topic and produced three
structured points, although it misdefined TTFT and omitted the requested practical
example. `UD-Q2_K_XL` was substantially worse: it hallucinated TTFT and TPOT as training
concepts, repeated itself, and hit the 192-token limit. Therefore the 2-bit quantization
is not worth using on this machine; its 0.11 GB saving does not compensate for the 6.03x
decode slowdown and the observed loss in answer quality. I would use `Q4_K_M` as the
baseline here.
