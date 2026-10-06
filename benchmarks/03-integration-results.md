# 03 - Integrate: RAG pipeline run

Host Windows-AMD64 · llama.cpp 10488 ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 8483.8 | 8484.0 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 5637.5 | 5637.6 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 9844.7 | 9844.8 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **7988.7** · total **7988.8**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **Goodput** is more useful than raw throughput because it specifically accounts for **SLOs** (Service Level Objectives).

While raw throughput ignores SLOs, Goodput counts only the requests per second that met the targets. This means Goodput measures how many requests actually satisfied the system's performance requirements, whereas raw throughput would count request

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory.

By storing the KV cache in non-contiguous pages, it removes the internal fragmentation that would otherwise waste most of the GPU's memory capacity.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound** and **decode is memory-bound**.

This is because the context states that:
*   **Prefill** is compute-bound (requires significant CPU/GPU time for the computation).
*   **Decode** is memory-bound (requires significant memory bandwidth for the output).

By splitting these operations into separate pools (prefill and decode), the sys


## Which N16-N19 pieces are real

1. **Trạng thái triển khai**:
   - N16 (Cloud/IaC): **stub**
   - N17 (Data pipeline): **stub**
   - N18 (Lakehouse): **stub**
   - N19 (Vector index & embeddings): **stub** (sử dụng keyword overlap trên TOY_DOCS)
   - N20 (Serving runtime): **real** (llama-server OpenAI-compatible trên cổng 8080)

2. **Phân tích Dominant Stage**:
   - Chặng **llm** chiếm tới 7988.7 ms (gần như 100% tổng thời gian 7988.8 ms), hoàn toàn khớp với kỳ vọng vì các thao tác retrieval trên bộ dữ liệu nhỏ chỉ mất 0.1 ms.
   - **Tấn công để giảm latency 2×**: Theo định luật Amdahl, tối ưu hóa chặng embed hoặc retrieve không mang lại ý nghĩa vì tỷ trọng của chúng chỉ chiếm 0.001%. Muốn giảm độ trễ 2×, ta bắt buộc phải tấn công vào chặng **llm** bằng các kỹ thuật:
     - **Prompt Caching / Prefix Caching**: Tái sử dụng KV cache của phần context và system prompt dài để triệt tiêu thời gian prefill (TTFT).
     - **Speculative Decoding / Giới hạn max output tokens**: Tăng tốc độ sinh token hoặc cắt giảm số token giải mã không cần thiết.
     - **Batching & GPU Offloading**: Tối ưu hóa số layer nạp vào GPU VRAM để tăng tốc độ decode.
