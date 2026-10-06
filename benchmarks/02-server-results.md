# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=6` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 20 | 0.35 | 25000 | 40000 | 40000 | 8.5 | 0.0% |
| 50 | 0 | 0.00 | 0 | 0 | 0 | 0.0 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.00x** (0% of linear) |
| P95 latency | **0.00x** |
| Effective concurrency at 50 users | 0.0 vs `--parallel 4` slots (occupancy/slot ratio 0.00) |

**Saturated.** Throughput stopped scaling (0.00x delivered for 5x offered, 0% of linear) even though effective concurrency (0.0) sits below 4 slots. Something other than decode-slot count is the limit -- look at memory bandwidth, or context/KV pressure.

P95 grew no faster than throughput (0.00x vs 0.00x), so this server still has headroom at 50 users.

> **Small sample.** Only 0 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

1. **Điểm bão hòa & Bằng chứng**: Server bão hòa hoàn toàn ở mức 50 users (và thực tế đã vượt ngưỡng bão hòa từ mức 10 users khi concurrency hiệu dụng đã là 8.5 > 4 slots). Con số thuyết phục nhất là **`requests_deferred = 75`** kết hợp với **Throughput đạt 0.00× (0% tuyến tính)**: khi tăng tải 5×, hàng đợi nổ tung khiến độ trễ vượt quá ngưỡng thời gian chạy 60s của Locust, dẫn đến 0 request kịp hoàn thành.
2. **Queue Time vs Compute Time**: Độ trễ tăng đột biến ở 50 users hoàn toàn là **Queue Time** (thời gian nghẽn hàng đợi). Máy chủ vẫn xử lý hết công suất (`n_busy_slots_per_decode = 3.44/4` và `requests_processing = 4`), nhưng do chỉ có 4 slots, 46 request còn lại phải chờ đợi phía sau khiến tổng thời gian chờ thổi phồng lên trên 60 giây.
3. **Knob thay đổi để nâng Goodput@SLO**: Knob đầu tiên tôi sẽ thay đổi là **tăng `--parallel`** (từ 4 lên 6 hoặc 8 slots) hoặc áp dụng **Prompt Caching / giảm `ctx-size`**. Lý do: Nút thắt nghẽn nghiêm trọng nhất ở đây là thiếu slots xử lý đồng thời dẫn tới tích tụ hàng đợi. Việc mở rộng slots kết hợp giảm thời gian giữ slot của mỗi request sẽ trực tiếp giải phóng `requests_deferred`, triệt tiêu Queue Time và đưa các request trở về dưới ngưỡng SLO độ trễ cho phép.
