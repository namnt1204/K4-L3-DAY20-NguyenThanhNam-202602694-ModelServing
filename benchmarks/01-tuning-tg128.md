# 01 - Tune: thread-count sweep

Model Qwen3.5-0.8B-Q4_K_M.gguf · host Windows-AMD64 · llama.cpp 10488
CPU: **6 physical · 12 logical** cores · 
gl=99 · metric 	g128

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 28.8 | 98% |
| 3 | 28.7 | 98% |
| 6 | 28.7 | 98% |
| 12 | 28.9 | 99% |
| 24 | 29.3 | 100% |

**Best**: -t 24 at 29.3 tok/s
**Slowest tested**: -t 3 at 28.7 tok/s (1.02x spread)
**Against the physical-core default** (-t 6, 28.7 tok/s): 1.02x

Use this in your run:

`ash
LAB_N_THREADS=24 make bench
`

## Your explanation

1. **Vị trí Knee**: Đường cong đo được gần như phẳng hoàn toàn trên toàn dải từ 1 đến 24 threads (tốc độ dao động hẹp từ 28.7 đến 29.3 tok/s, độ lệch chỉ 1.02×). Điểm bão hòa (knee) xuất hiện ngay từ 1 thread và chạm ngưỡng ổn định quanh 6 physical cores của CPU AMD Ryzen 5 5500U.
2. **Cơ chế vật lý**: 
   - Giai đoạn Decode bị giới hạn bởi **Memory Bandwidth** (băng thông bộ nhớ) chứ không phải FLOPs, vì mỗi token sinh ra bắt buộc phải nạp lại weights từ bộ nhớ. Với mô hình siêu nhẹ Qwen 0.8B (~0.5 GB), một luồng đọc duy nhất đã nhanh chóng tận dụng hết khả năng cấp phát dữ liệu của kênh RAM.
   - Khi tăng số thread lên mức logical (12) và oversubscribe (24), các thread thừa không giúp tăng tốc mà phải tranh chấp chung kênh truyền bộ nhớ (memory bus contention), đồng thời gây overhead điều phối luồng và chuyển đổi ngữ cảnh (context switching). Do đó, thông lượng decode đi ngang và không có speedup thực tế đáng kể giữa các cấu hình luồng.
