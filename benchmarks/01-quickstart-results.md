# 01 - Measure: latency baseline

Model Qwen3.5 0.8B · host Windows-AMD64 · llama.cpp 10488
Settings: 	hreads=6 
gl=99 ctx=2048
max_tokens=64 · warm-up discarded
Completed requests: Q4_K_M 10/10 · UD-Q2_K_XL 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2941 | 756 / 935 | 40.3 / 45.0 | 3284 / 3644 / 3644 | 24.8 |
| UD-Q2_K_XL | 0.39 | 2842 | 777 / 874 | 40.0 / 44.7 | 3312 / 3661 / 3661 | 25.0 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. decode tok/s = 1000 / TPOT_p50.
- UD-Q2_K_XL and Q4_K_M decode within 2% of each other here, for 0.11 GB difference on disk.

## Your observation

1. **Dung lượng**: Bản 2-bit (UD-Q2_K_XL) có kích thước 0.39 GB, nhẹ hơn bản 4-bit (Q4_K_M, 0.50 GB) là 0.11 GB (~115 MB, giảm ~22% dung lượng lưu trữ trên đĩa).
2. **Tốc độ**: Tốc độ Decode của bản 2-bit đạt 25.0 tok/s (TPOT P50 là 40.0 ms), chỉ nhanh hơn bản 4-bit đạt 24.8 tok/s (TPOT P50 là 40.3 ms) khoảng 0.2 tok/s (~0.8%), mức cải thiện gần như không đáng kể; trong khi TTFT P50 của 2-bit (777 ms) thậm chí chậm hơn 4-bit (756 ms) do chi phí tính toán giải lượng tử hóa (dequantization overhead).
3. **Đánh đổi**: Hoàn toàn **không đáng dùng 2-bit**. Với model nhỏ 0.8B, kích thước weights vốn đã nhẹ nên memory bandwidth không phải nút thắt lớn; việc nén xuống 2-bit làm giảm sút nghiêm trọng chất lượng câu trả lời và tính mạch lạc ngữ pháp mà tốc độ thực tế gần như giữ nguyên, do đó bản 4-bit (Q4_K_M) tối ưu và hữu dụng hơn rất nhiều.
