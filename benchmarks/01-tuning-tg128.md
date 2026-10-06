# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 21.1 | 100% |
| 2 | 20.7 | 98% |
| 4 | 21.1 | 100% |
| 8 | 20.5 | 97% |
| 16 | 20.9 | 99% |

**Best**: `-t 1` at 21.1 tok/s
**Slowest tested**: `-t 8` at 20.5 tok/s (1.03x spread)
**Against the physical-core default** (`-t 4`, 21.1 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=1 make bench
```

## Your explanation

The effective knee is already at `-t 1`: one thread and the four-physical-core
default both round to 21.1 tok/s, and every point from 1 through 16 threads stays
within 3% of the best result. The best-to-default speedup is therefore only 1.00x.
This is a flat curve rather than the usual CPU-only curve that rises toward the
physical-core count.

The reason is that this run uses `ngl=99`, so decode is predominantly offloaded to
the 2 GB GeForce MX110 through Vulkan. The CPU thread setting mainly affects host-side
scheduling and token-processing work; adding CPU threads cannot make the GPU kernels
or GPU memory subsystem consume model weights faster. The small dip at `-t 8`
(20.5 tok/s) is consistent with scheduling/cache contention or normal run-to-run
noise, but `-t 16` recovering to 20.9 tok/s means there is no defensible monotonic
oversubscription penalty in this measurement. Since `-t 1` provides no measurable
throughput gain over the physical-core default, I keep the existing baseline rather
than rerunning it and claiming a nonexistent speedup.
