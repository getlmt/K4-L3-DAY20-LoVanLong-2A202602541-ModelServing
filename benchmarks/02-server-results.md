# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 35 | 0.61 | 14000 | 19000 | 19000 | 8.3 | 0.0% |
| 50 | 40 | 0.70 | 30000 | 53000 | 56000 | 19.7 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.16x** (23% of linear) |
| P95 latency | **2.79x** |
| Effective concurrency at 50 users | 19.7 vs `--parallel 4` slots (occupancy/slot ratio 4.93) |

**Saturated.** Throughput delivered only 1.16x for 5x the offered load, and effective concurrency (19.7) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.16x while P95 moved 2.79x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

**Server đã bão hoà ngay từ 10 user.** Ở 10 user, effective concurrency đã là 8.3, lớn hơn
4 slot. P50 là 14 s, trong khi request ngắn nhanh nhất chỉ mất 6.3 s (thời gian phục vụ thật
khi không phải chờ slot). Từ 10 lên 50 user (tải ×5), RPS chỉ ×1.16 (0.61 → 0.70, 23% tuyến
tính) còn P95 ×2.79 (19 → 53 s). Ghi chú: ảnh `04-locust-10.png` ghi 36 request vì request
long-rag cuối cùng xong đúng lúc locust đang tắt, sau lần ghi CSV cuối; report này dùng CSV (35).

**Con số thuyết phục tôi nhất:** tốc độ decode tổng của server gần như cố định ở khoảng
**34 tok/s**. Trong lúc load-50, `tokens_predicted_total` tăng 2046 token trong 59.9 s
(`02-server-batching-u50.md`), mà 4 slot thì đều bận (`n_busy_slots_per_decode` 3.93/4). Đó là
trần của 4 slot trên 8 core. Khi đã chạm trần, thêm user không tăng throughput mà chỉ làm hàng
đợi dài thêm (`requests_deferred` 36–46).

**Phần P95 tăng thêm là queue time, không phải compute time.** Thời gian phục vụ một request
gần như không đổi giữa hai lần chạy: min latency 6.3 s ở 10 user và 5.2 s ở 50 user. Phần còn
lại của P95 = 53 s là thời gian chờ slot. Theo Little's Law, L = λ × W = 0.70 × 28.1 s ≈ 19.7
request trong hệ thống, trong khi chỉ có 4 request được decode cùng lúc. Gauge của server cho
thấy hàng đợi thật còn dài hơn (36–46), vì locust bỏ qua các request vẫn đang chờ khi dừng chạy.

**Goodput@SLO.** Tôi chọn SLO là P95 end-to-end ≤ 20 s cho câu trả lời chat ngắn.
- 10 user: P95 = 19 s, đạt SLO, nên goodput ≈ throughput = 0.61 RPS.
- 50 user: P95 = 53 s và ngay cả P50 đã là 30 s, nên không quá một nửa số request đạt SLO.
  Goodput chỉ ≤ 0.35 RPS, dù throughput tăng lên 0.70 RPS.

Theo Little's Law, muốn W ≤ 20 s với λ ≈ 0.65 RPS thì chỉ được khoảng 13 request trong hệ
thống. Nghĩa là server này chỉ giữ được SLO với khoảng 10–13 user đồng thời.

**Knob tôi đổi trước: GPU offload (bản llama.cpp CUDA, `-ngl 99`).** Nút cổ chai là tốc độ
decode, mà decode bị chặn bởi memory bandwidth: DDR4 dual-channel chỉ cho khoảng 27 GB/s thực
tế (`01-tuning-tg128.md`). GTX 1650 của máy dùng GDDR6, max memory clock 6001 MHz trên bus
128-bit, tức khoảng 192 GB/s. Tăng service rate μ là cách duy nhất vừa tăng trần throughput vừa
kéo P95 về trong SLO.

Tôi không chọn tăng `--parallel` trước. Từ 1 request đơn lẻ lên 4 slot, throughput mới chỉ ×2
(17 → 34 tok/s), vì CPU đã bắt đầu thiếu compute ở batch lớn. Thêm slot chủ yếu làm mỗi
request chậm hơn, và `ctx=2048` chia cho 8 slot chỉ còn 256 token mỗi slot, không đủ cho prompt
RAG dài. Nếu không dùng được GPU, knob thứ hai là admission control: giới hạn khoảng 12 request
đồng thời và trả lỗi 429 ngay cho phần vượt, để các request đã nhận vẫn nằm trong SLO.
