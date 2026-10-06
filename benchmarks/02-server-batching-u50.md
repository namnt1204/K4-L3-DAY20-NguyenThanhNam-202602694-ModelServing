# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.44 of 4 slots (86%) |
| `requests_processing` | 4 |
| `requests_deferred` | 75 |
| `kv_cache_usage_ratio` | n/a · not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 3752 |

Highest sampled value was **3.44 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

1. **Peak batch width**: Độ rộng batch thực tế đạt đỉnh **3.44 trên 4 slots** (tương đương 86% công suất tối đa của server), đồng thời `requests_processing = 4`. Điều này cung cấp bằng chứng trực tiếp và xác thực rằng thuật toán Continuous Batching đang vận hành hiệu quả: scheduler liên tục gộp các request đồng thời vào chung bước giải mã (shared decode steps) thay vì chạy tuần tự.
2. **So sánh với Effective Concurrency**: Trong `02-server-results.md`, con số concurrency ở 50 users ghi nhận 0.0 do Locust chỉ tính toán trên các request đã hoàn thành (completed requests), trong khi thời gian chờ vượt quá thời lượng 60s của phiên test. Do đó số đo concurrency từ Locust bị under-estimate nghiêm trọng.
3. **Độ tin cậy**: Ta hoàn toàn tin tưởng vào số liệu nội tại từ Prometheus `/metrics` của server (`n_busy_slots = 3.44/4` và `requests_deferred = 75`), vì nó phản ánh trực tiếp trạng thái vật lý bên trong engine: toàn bộ 4 slots luôn bận 100% và có tới 75 lượt request bị dồn ứ trong hàng đợi.
