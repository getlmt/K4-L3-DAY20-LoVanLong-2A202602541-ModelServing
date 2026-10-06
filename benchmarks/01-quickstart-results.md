# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3868 | 386 / 429 | 58.7 / 59.0 | 4057 / 4137 / 4137 | 17.0 |
| UD-Q2_K_XL | 2.24 | 3792 | 498 / 549 | 45.6 / 46.0 | 3395 / 3423 / 3423 | 21.9 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.29x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

Số liệu trên là lần chạy `.\lab.ps1 bench` đầu tiên (Ryzen 7 4800H, CPU-only, `-t 8`,
`ngl=0`), ngay sau khi tải model nên weights có thể đã nằm sẵn trong page cache.

**Tốc độ và kích thước.** `UD-Q2_K_XL` nhỏ hơn 0.73 GB (2.24 so với 2.97 GB, khoảng -25%)
và decode nhanh hơn **1.29x** (TPOT P50 45.6 so với 58.7 ms, tức 21.9 so với 17.0 tok/s).
Điều này khớp với việc decode bị chặn bởi memory bandwidth: mỗi token phải đọc lại weights,
ít byte hơn thì mỗi bước decode nhanh hơn. Ngược lại, **TTFT của 2-bit chậm hơn khoảng 29%**
(P50 498 so với 386 ms): prefill bị chặn bởi compute, và kernel 2-bit tốn nhiều lệnh giải nén
(dequantize) hơn cho mỗi trọng số, nên phần byte tiết kiệm được không bù lại.

**Chất lượng.** Tôi hỏi cùng 3 câu trên bản 4-bit (`.\lab.ps1 serve`, :8080) và bản 2-bit
(`serve.py --compare --port 8090`), `temperature=0`, `max_tokens=200`:

| Câu hỏi | UD-Q4_K_XL | UD-Q2_K_XL |
|:--|:--|:--|
| Phân biệt TTFT và TPOT (2 câu) | Đúng · 62 tok · 4.19 s | **Sai**: gọi TPOT là "Token Processing Time" · 84 tok · 4.48 s |
| 3 bút giá $2, 12 bút giá bao nhiêu | Đúng $8 · 49 tok · 3.40 s | Đúng $8 nhưng dài gần gấp đôi · 94 tok · 5.09 s |
| Giải thích continuous batching (tiếng Việt) | Sai một phần (gọi là kỹ thuật huấn luyện) · 75 tok · 4.87 s | Sai hơn và dài hơn · 102 tok · 5.30 s |

Bản 2-bit sinh nhiều hơn 35-92% số token cho cùng câu hỏi, nên dù mỗi token nhanh hơn 1.29x,
**cả 3 câu trả lời đều xong chậm hơn** bản 4-bit. Trong bench, E2E của 2-bit có vẻ nhanh hơn
(3395 so với 4057 ms) chỉ vì hầu hết request bị cắt ở `max_tokens=64`.

**Kết luận: không đáng dùng 2-bit trên máy này.** RAM 15.4 GB không phải giới hạn, nên 0.73 GB
tiết kiệm được không mua thêm được gì; đổi lại là TTFT tệ hơn, câu trả lời sai nhiều hơn và dài
hơn. 2-bit chỉ đáng cân nhắc khi RAM là ràng buộc cứng (máy 4-8 GB) và chấp nhận chất lượng thấp hơn.
