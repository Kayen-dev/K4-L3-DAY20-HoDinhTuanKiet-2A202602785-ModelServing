# Bonus - GPU offload sweep

Host `Windows-AMD64` · backend(s) `nvidia_cuda` ·
llama.cpp `b10488` · `threads=4` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 27.2 | 1.00x | 100% |
| 8 | 22.2 | 0.81x | 81% |
| 16 | 21.4 | 0.79x | 79% |
| 24 | 21.2 | 0.78x | 78% |
| 32 | 20.9 | 0.77x | 77% |
| 99 | 20.7 | 0.76x | 76% |

Best: `-ngl 0` at 27.2 tok/s
-- 1.00x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

Full offload was **not** best on this machine. CPU-only (`-ngl 0`) reached
**27.2 tok/s**, while full offload (`-ngl 99`) reached only **20.7 tok/s**, or 76%
of the CPU result. Changing the serving configuration from full offload back to
CPU-only therefore improved decode throughput by **1.31x**.

This is not a VRAM-capacity cliff: the 0.50 GB Q4 model fits comfortably in the
MX110's 2 GB VRAM, and the curve falls nearly monotonically instead of peaking just
before a spill point. The small change from `-ngl 32` (20.9 tok/s) to `-ngl 99`
(20.7 tok/s) also shows that adding values beyond the model's useful layer count no
longer changes much. For this small model, the i5-8265U's AVX2 CPU path and system
memory outperform the low-end MX110's Vulkan kernels; GPU dispatch, synchronization,
and the host/device boundary add overhead that the small workload cannot amortize.

The header's `nvidia_cuda` label comes from accelerator detection in `hardware.json`.
The binary actually measured here is the Vulkan asset
`llama-b10488-bin-win-vulkan-x64.zip` recorded in `runtime/active.json`.
