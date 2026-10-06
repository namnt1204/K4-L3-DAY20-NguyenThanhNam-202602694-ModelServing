# Bonus - GPU offload sweep

Host Windows-AMD64 · backend(s) 
vidia_cuda ·
llama.cpp 10488 · 	hreads=6 · metric 	g128

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 31.8 | 1.00x | 100% |
| 8 | 10.2 | 0.32x | 32% |
| 16 | 12.8 | 0.40x | 40% |
| 24 | 23.3 | 0.73x | 73% |
| 32 | 29.3 | 0.92x | 92% |
| 99 | 29.4 | 0.93x | 93% |

Best: -ngl 0 at 31.8 tok/s
-- 1.00x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

1. **Hiện tượng quan sát**: 
   - Đỉnh thông lượng decode tốt nhất đạt được ở **CPU-only (-ngl 0 với 31.8 tok/s)**. 
   - Khi bắt đầu partial offload lên GPU GTX 1650, tốc độ tụt dốc thảm hại ở -ngl 8 xuống chỉ còn **10.2 tok/s** (suy giảm hơn 3 lần, chỉ đạt 32% hiệu năng CPU).
   - Tốc độ tăng dần trở lại khi offload nhiều layer hơn: -ngl 16 (12.8 tok/s), -ngl 24 (23.3 tok/s), -ngl 32 (29.3 tok/s), và đạt 29.4 tok/s khi full offload (-ngl 99).

2. **Cơ chế vật lý (PCIe Host-to-Device Bottleneck)**:
   - Yếu tố cạn kiệt trước ở đây không phải là VRAM (GTX 1650 có 4 GB VRAM, thừa sức chứa trọn vẹn model Qwen 0.8B chỉ ~0.5 GB), mà là **băng thông và độ trễ giao tiếp Host-to-Device qua bus PCIe (PCIe transfer penalty)**.
   - Khi chia tách các layer giữa CPU và GPU (partial offload), ở mỗi token giải mã, tensor activation trung gian phải liên tục được copy qua lại giữa CPU RAM và GPU VRAM qua bus PCIe. Độ trễ trễ truyền dữ liệu (data transfer latency) và chi phí đồng bộ hóa (synchronization barrier overhead) hoàn toàn áp đảo thời gian tính toán thực tế của các layer đó.
   - Khi offload toàn bộ (-ngl 99), việc giao tiếp qua PCIe giữa các layer bị triệt tiêu, đưa tốc độ hồi phục về 29.4 tok/s. Tuy nhiên, đối với một mô hình siêu nhỏ (0.8B), CPU Ryzen 5 5500U đọc weights từ RAM có độ trễ cực thấp và không phải chịu overhead khởi tạo CUDA kernel (kernel launch overhead) hay driver latency, do đó CPU-only thậm chí còn đạt tốc độ cao hơn một chút so với GPU GTX 1650 laptop.
