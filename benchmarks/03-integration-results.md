# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 10715.0 | 10715.1 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.2 | 6724.6 | 6724.9 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 8311.0 | 8311.2 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **8583.5** · total **8583.7**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **goodput** is more useful than raw throughput because it specifically filters for requests that met the **TTFT** (Time-to-Fault) and **TPOT** (Time-to-Pollout) targets.

Raw throughput ignores SLOs (Service Level Objectives), whereas goodput counts only the requests per second that met these targets. This distinction highlights that goodput is designed to account fo

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** by storing the Key-Value (KV) cache in non-contiguous pages. This design removes the wasted space that would otherwise be consumed by the internal fragmentation of contiguous memory blocks, allowing the GPU to utilize more available memory for computation.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bound**, as the context explicitly states: "Disaggregated serving splits prefill and decode onto separate pools because prefill is compute-bound and decode is memory-bandwidth-bound."

In this scenario, the model's compute cost is high (prefill), while the memory bandwidth cost is high (decode). By separating t


## Which N16-N19 pieces are real

- **N16 (cluster): stubbed/not integrated.** This run used one local llama.cpp
  process rather than the N16 cluster.
- **N17 (data pipeline): stubbed.** The corpus was the hard-coded `TOY_DOCS` list;
  no N17 ingestion or transformation pipeline supplied it.
- **N18 (lakehouse): stubbed.** Documents stayed in the in-memory Python list, not
  a lakehouse table or other durable N18 store.
- **N19 (vector index): stubbed.** Retrieval used keyword overlap, not embeddings
  or a vector index.

The dominant stage was **LLM inference: 8583.5 ms, effectively 100% of the 8583.7
ms mean total**. This was expected for an in-memory six-document corpus: embedding
was disabled (0.0 ms) and keyword retrieval averaged only 0.1 ms. To halve pipeline
latency I would therefore attack the LLM stage, first by requesting concise answers
and reducing `max_tokens` from 200 to about 64. The server timings show 62-110
decoded tokens per answer and 3124-5518 ms spent in decode, so shortening generation
can remove seconds; even eliminating retrieval entirely would save only 0.1 ms.
