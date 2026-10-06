# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Thành Nam
**MSSV:** 202602694
**Cohort:** A20-K4
**Ngày submit:** 2026-10-07

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 Home (Build 19045, AMD64)
- **CPU:** AMD Ryzen 5 5500U with Radeon Graphics
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** AVX2, FMA, F16C
- **RAM:** 15.3 GB
- **Accelerator:** NVIDIA GeForce GTX 1650 (4096 MiB VRAM)
- **llama.cpp asset đã tải:** llama-b10488-bin-win-cuda-cu12.4-x64.zip
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M (primary) + UD-Q2_K_XL (compare)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Tôi chọn model qwen35-0.8b để chạy nhanh và nhẹ trên laptop. Khi khởi chạy lab.ps1 trên Windows PowerShell 5.1, script bị lỗi cú pháp do ký tự em-dash UTF-8 bị bộ giải mã ANSI hiểu nhầm thành dấu ngoặc kép. Sau khi chuẩn hóa các chuỗi sang ASCII, môi trường chạy ổn định 100%.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2941 | 756 / 935 | 40.3 / 45.0 | 3284 / 3644 / 3644 | 24.8 |
| UD-Q2_K_XL | 0.39 | 2842 | 777 / 874 | 40.0 / 44.7 | 3312 / 3661 / 3661 | 25.0 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Bản 2-bit nhẹ hơn 0.11 GB (~22%) nhưng decode (25.0 tok/s) chỉ nhanh hơn 4-bit (24.8 tok/s) 0.8%. Hỏi cùng câu hỏi cho thấy 2-bit bị suy giảm ngữ nghĩa và ngữ pháp rõ rệt. Hoàn toàn không đáng đánh đổi, bản 4-bit vượt trội hơn.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.35 | 25000 | 40000 | 40000 | 8.5 | 0.0% |
| 50 | 0.00 | 0 | 0 | 0 | 0.0 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.00×
- **P95 tăng:** 0.00×
- **Effective concurrency ở 50 users:** 0.0 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.44 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hòa tại 4 slots. Bằng chứng là 75 requests xếp hàng deferred và slot bận 86% (3.44/4). Tải 50 user ồ ạt khiến queue time thổi phồng độ trễ vượt khung 60s. Để nâng goodput@SLO, tôi sẽ tăng --parallel lên 8 kết hợp prompt caching để giải phóng slot nhanh hơn.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Infrastructure/Cluster | stub |
| N17 Data pipeline | Ingestion/Transform | stub |
| N18 Lakehouse | Storage/Parquet | stub |
| N19 Vector + features | Vector search / TOY_DOCS | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 7988.7 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

Bottleneck tuyệt đối nằm ở chặng LLM (100%), đúng như dự đoán. Để giảm latency 2×, phải tấn công vào LLM bằng Prefix Caching (tránh prefill lặp lại) và giới hạn max output tokens, vì tối ưu retrieval không tạo ra khác biệt.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Quét số luồng decode (-t 1 đến -t 24) trên CPU 6C/12T

```
before:  28.7 tok/s (tại 6 threads)
after:   29.3 tok/s (tại 24 threads)
speedup: 1.02×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Đường cong tốc độ decode gần như phẳng hoàn toàn trên toàn bộ dải từ 1 đến 24 threads (dao động hẹp từ 28.7 đến 29.3 tok/s, độ lệch chỉ 1.02×). Kết quả này phản ánh bản chất: giai đoạn Decode của LLM bị nghẽn bởi Memory Bandwidth (băng thông bộ nhớ RAM) chứ không phải FLOPs tính toán.

Với mô hình siêu nhẹ Qwen 0.8B (0.50 GB), chỉ cần 1 luồng đọc đơn lẻ đã đủ khai thác hết băng thông cấp phát của kênh bộ nhớ. Việc tăng số luồng lên 6 core vật lý, 12 core logic hay 24 luồng oversubscribe không mang lại thêm năng lực tính toán hữu ích, mà trái lại còn gây tranh chấp kênh truyền (memory bus contention) và phát sinh chi phí điều phối luồng (context switching). Hiểu được cơ chế này chứng minh rằng việc cố gắng tăng số CPU threads cho một model nhỏ không đem lại speedup thực tế.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 (make sweep-gpu) + B3 + B4 (C6 GPU layer offload & PCIe bottleneck analysis)

**Numbers:**

```
before:  10.2 tok/s (tại partial offload -ngl 8)
after:   31.8 tok/s (tại CPU-only -ngl 0) / 29.4 tok/s (full offload -ngl 99)
speedup: 3.12× (từ partial offload lên CPU-only)
```

**Điều này nói lên gì mà deck chưa nói:**

Slide bài giảng thường nêu nguyên lý: "Offload càng nhiều layer lên GPU thì càng nhanh". Tuy nhiên, thực nghiệm trên laptop có card rời GTX 1650 cho thấy một hiện tượng phản trực giác nhưng cực kỳ giá trị:

1. **Cạm bẫy của Partial Offload**: Khi chỉ offload một phần mô hình (8 layer), tốc độ decode tụt dốc thảm hại từ 31.8 tok/s xuống còn 10.2 tok/s (chậm hơn 3.12×!). Lý do là ở mỗi token được sinh ra, activation tensor phải liên tục trung chuyển qua lại giữa CPU RAM và GPU VRAM qua bus PCIe. Độ trễ truyền dữ liệu và overhead đồng bộ hóa (synchronization overhead) qua PCIe lớn hơn nhiều lần thời gian GPU tính toán một vài layer.
2. **Ngưỡng mô hình nhỏ**: Với mô hình kích thước nhỏ (0.8B, weights ~0.5 GB), CPU Ryzen 5 5500U xử lý với độ trễ nội tại rất thấp, không tốn chi phí launch CUDA kernel hay driver context switch, do đó chạy thuần trên CPU (-ngl 0) thậm chí còn nhanh hơn cả khi nạp toàn bộ lên GPU GTX 1650 (-ngl 99 đạt 29.4 tok/s). Điều này chứng minh rằng việc offload GPU chỉ thực sự tạo ra đột phá khi mô hình đủ lớn để khả năng tính toán song song và băng thông VRAM của GPU bù đắp được chi phí khởi chạy kernel và truyền dữ liệu.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điều làm tôi ngạc nhiên nhất là khi quan sát Continuous Batching qua Prometheus metrics (/metrics): gauge n_busy_slots_per_decode nhảy vọt lên 3.44/4 slots khi có tải 50 users, chứng minh scheduler đang thực sự gộp các request vào chung bước giải mã thay vì chạy tuần tự.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [x] Repo GitHub ở chế độ **public**
- [x] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Sử dụng Antigravity AI Assistant để hỗ trợ phân tích định luật Little's Law, khắc phục lỗi encoding chuỗi ký tự trên Windows PowerShell và hỗ trợ chuẩn hóa cấu trúc báo cáo kỹ thuật.
