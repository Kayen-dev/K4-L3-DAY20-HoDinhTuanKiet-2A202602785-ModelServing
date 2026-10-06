# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 18 | 0.34 | 25000 | 43000 | 43000 | 8.4 | 0.0% |
| 50 | 20 | 0.38 | 34000 | 53000 | 53000 | 11.2 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.09x** (22% of linear) |
| P95 latency | **1.23x** |
| Effective concurrency at 50 users | 11.2 vs `--parallel 4` slots (occupancy/slot ratio 2.80) |

**Saturated.** Throughput delivered only 1.09x for 5x the offered load, and effective concurrency (11.2) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.09x while P95 moved 1.23x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

> **Small sample.** Only 18 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

The server is saturated somewhere between 10 and 50 users. The strongest evidence
is that a **5x** increase in offered load delivered only **1.09x** the throughput
(22% of linear scaling), while P95 still rose **1.23x** from 43 s to 53 s. Because
the 10-user run has only 18 completions, I give this throughput comparison more
weight than Little's-Law concurrency; nevertheless, the 50-user occupancy of 11.2
requests versus 4 slots agrees with the direct server evidence of 3.82/4 busy slots
and 46 deferred requests. The extra load therefore became queue time.

For a **45 s latency SLO**, all 10-user completions met the target (maximum 42.8 s),
so observed goodput was about **0.34 req/s**. At 50 users, 45 s is approximately the
80th percentile, so goodput was only about **0.38 x 0.80 = 0.30 req/s**. These are
small-sample estimates, but they show that raw throughput rose while SLO-qualified
goodput fell.

I would first reduce the output-token budget (for example, 48/96 to 32/64 and require
concise answers). Decode keeps each slot occupied for most of these long requests,
so shorter outputs directly reduce service and queue time. I would not add more
threads because the thread sweep was already flat, nor switch to Q2 because it was
6.03x slower on this Vulkan setup; adding more parallel slots on the same small GPU
would also increase contention and shrink the context available per slot.
