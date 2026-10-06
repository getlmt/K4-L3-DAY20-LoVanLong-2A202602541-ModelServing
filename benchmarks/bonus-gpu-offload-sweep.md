# Bonus - GPU offload sweep

Host `Windows-AMD64` · backend(s) `nvidia_cuda, vulkan` ·
llama.cpp `b10488` · `threads=8` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 16.0 | 1.00x | 23% |
| 8 | 17.7 | 1.11x | 26% |
| 16 | 26.1 | 1.63x | 38% |
| 24 | 40.5 | 2.53x | 59% |
| 32 | 53.1 | 3.32x | 77% |
| 99 | 68.7 | 4.30x | 100% |

Best: `-ngl 99` at 68.7 tok/s
-- 4.30x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

Chạy bằng bản prebuilt `llama-b10488-bin-win-cuda-12.4-x64.zip` trên GTX 1650 4 GB (GDDR6).
Base track vẫn là bản CPU.

**Full offload là tốt nhất: `-ngl 99` đạt 68.7 tok/s, nhanh gấp 4.30x so với `-ngl 0`.**
Đường cong không có đỉnh ở giữa, nghĩa là VRAM không hết. Log nạp model ghi
`CUDA0 model buffer = 1481.89 MiB`, KV 36 MiB, compute 117.52 MiB, tổng khoảng 1.6 GB trên
3.3 GB VRAM còn trống. File 3 GB vừa card 4 GB vì `per_layer_token_embd` (1.6 GB, log ghi
`CPU_Mapped 1804 MiB`) nằm lại RAM như một bảng tra: mỗi token chỉ đọc 1 dòng.

**Vì sao 4.3x:** decode bị chặn bởi memory bandwidth. Mỗi token đọc khoảng 1.55 GB weights
(`01-tuning-tg128.md`). Khi chạy hoàn toàn trên GPU, GDDR6 đạt khoảng 113 GB/s (59% trần
192 GB/s). Khi chạy hoàn toàn trên CPU, DDR4 chỉ đạt khoảng 25 GB/s (49% trần 51.2 GB/s).
Tỉ lệ băng thông khoảng 4.6x, gần với tốc độ đo được 4.3x.

**Partial offload là bài toán Amdahl.** Thời gian mỗi token ≈ byte trên CPU ÷ 25 GB/s +
byte trên GPU ÷ 113 GB/s. Mô hình này khớp đo thực trong khoảng ±7% từ `-ngl 24` trở lên. Ở
`-ngl 8`, dù 37% số byte đã lên GPU, CPU vẫn phải đọc 0.98 GB, nên chỉ được 1.11x. Ở
`-ngl 8–16`, tốc độ đo được còn thấp hơn mô hình 13–27%: chia graph giữa hai thiết bị có chi
phí riêng (đồng bộ, copy activations, đánh thức threadpool CPU) mà tôi chưa đo tách được.

**Điều tôi không ngờ: `-ngl` đếm cả output layer, và output layer được offload trước.** Tôi
đo thêm bằng llama-bench (cùng `-t 8 -p 0 -n 128 -r 2`):

| -ngl | tg128 (tok/s) | Log nạp model |
|:--|--:|:--|
| 35 | 60.79 ± 1.12 | `offloading output layer to GPU` + 34 repeating layers, `offloaded 35/36` |
| 36 | 68.82 ± 0.04 | `offloaded 36/36` |

Model có đúng 35 layer, nhưng `-ngl 35` chưa phải full offload: layer 0 và KV của nó vẫn ở
CPU (log ghi `CPU KV buffer 2 MiB`). Chỉ khoảng 51 MiB weights ở lại CPU mà mất 13% tốc độ, vì
mỗi token vẫn phải đi qua CPU một lần. Ban đầu tôi đoán ngược lại (LM head được chuyển cuối
cùng); log đã cho thấy tôi sai.

Ghi chú: `-ngl 0` trên bản CUDA (16.0 tok/s) chậm hơn bản CPU ở `make tune` (17.5 tok/s)
khoảng 9%. Tôi chưa kiểm tra nguyên nhân.
