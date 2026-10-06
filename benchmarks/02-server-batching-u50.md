# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.82 of 4 slots (95%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 2136 |

Highest sampled value was **3.82 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

The peak average batch width was **3.82 of 4 slots (95%)**. This agrees with the
effective server concurrency: `requests_processing` reached all 4 slots while
`requests_deferred` reached 46, so the 50 Locust users saturated the four decode
slots and the remainder queued. I trust `n_busy_slots_per_decode` for actual batch
occupancy because it is measured inside llama.cpp during decode; Locust's 50-user
setting is offered concurrency, not the number of requests decoded simultaneously.
Together, the near-four busy-slot average and non-zero deferred queue are direct
evidence that continuous batching was active under load.
