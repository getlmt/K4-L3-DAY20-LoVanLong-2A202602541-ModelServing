# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **8 physical · 16 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 8.5 | 48% |
| 4 | 17.3 | 99% |
| 8 | 17.5 | 100% |
| 16 | 16.0 | 91% |
| 32 | 14.3 | 82% |

**Best**: `-t 8` at 17.5 tok/s
**Slowest tested**: `-t 1` at 8.5 tok/s (2.07x spread)
**Against the physical-core default** (`-t 8`, 17.5 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Your explanation

**Knee nằm ở 4 thread, sớm hơn số core vật lý.** Từ 1 lên 4 thread, tốc độ tăng gấp đôi
(8.5 → 17.3 tok/s), nhưng từ 4 lên 8 thread (= số core vật lý) chỉ thêm 1% (17.5 tok/s).
Mặc định của lab (`-t 8`) đã là điểm tốt nhất (1.00x), nhưng 4 core sau gần như không đóng góp.

**Vì sao: decode chạm trần memory bandwidth.** Mỗi token mới phải đọc lại weights từ RAM.
Tôi đọc kích thước tensor trong file GGUF: trừ `per_layer_token_embd` (1.62 GB, mỗi token chỉ
tra 1 dòng), mỗi bước decode phải đọc khoảng **1.55 GB**. Trong đó có `token_embd` 0.28 GB,
vì Gemma dùng chung nó làm LM head nên bị đọc toàn bộ. Máy có 2 thanh DDR4-3200 chạy
dual-channel, nên trần lý thuyết là 2 × 8 B × 3200 MT/s = **51.2 GB/s**.

| -t | tg128 (tok/s) | Lượng đọc thực tế (1.55 GB × tok/s) |
|--:|--:|--:|
| 1 | 8.5 | 13.2 GB/s |
| 4 | 17.3 | 26.9 GB/s |
| 8 | 17.5 | 27.2 GB/s |
| 16 | 16.0 | 24.9 GB/s |
| 32 | 14.3 | 22.2 GB/s |

Một thread chỉ kéo được khoảng 13 GB/s: một core vừa phải tự giải nén và nhân, vừa chỉ giữ
được một số giới hạn yêu cầu đọc RAM đang chờ cùng lúc (memory-level parallelism). Từ 4 thread
trở đi, tổng lượng đọc chững lại ở khoảng 27 GB/s (khoảng 53% trần lý thuyết, mức thường gặp
của DDR4 trên laptop). Lúc này bộ nhớ đã là nút cổ chai: thêm core chỉ thêm thread xếp hàng
chờ cùng 2 memory channel.

**Vượt số core vật lý thì chậm đi.** `-t 16` dùng cả luồng SMT: hai luồng logical dùng chung
một core (chung L1/L2, chung bộ load/store), nên không tạo thêm bandwidth. Trong khi đó
threadpool của llama.cpp phải đồng bộ (barrier) nhiều thread hơn sau mỗi op của graph, nên
chậm đi 9%. `-t 32` (gấp đôi số luồng phần cứng) bị oversubscribe: hệ điều hành phải chia
lượt chạy, và thread đã xong việc phải đứng chờ ở barrier cho thread đang bị tạm dừng, nên
chậm đi 18%.

**Kiểm chứng chéo với bench.** Bản 2-bit đọc ít hơn 1.46x số byte mỗi bước (1.07 so với
1.55 GB) nhưng chỉ decode nhanh hơn 1.29x, tức chỉ kéo được khoảng 23 GB/s. Kernel 2-bit tốn
nhiều compute giải nén hơn nên không còn thuần bandwidth-bound. Đây cũng là lý do TTFT
(prefill, vốn compute-bound) của bản 2-bit chậm hơn.
