# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 15 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.93 of 4 slots (98%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 4330 |

Highest sampled value was **3.93 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

`make metrics` chạy chồng thời gian với `load-50` (bắt đầu 4 s sau locust). Peak
`n_busy_slots_per_decode` là **3.93 / 4 slot (98%)**, và cả 15 mẫu đều nằm trong khoảng
3.87–3.93, với `requests_processing = 4` suốt lần chạy. Nghĩa là gần như mọi bước decode đều
gộp đủ 4 request: continuous batching đang hoạt động. Lợi ích đo được:
`tokens_predicted_total` tăng 2046 token trong 59.9 s, tức khoảng **34.2 tok/s tổng**, gấp
khoảng 2 lần một request đơn lẻ (16.8 tok/s ở smoke, 17.5 tok/s ở tune). Mỗi bước decode đọc
weights từ RAM một lần cho cả 4 sequence. Mức tăng chỉ khoảng 2x chứ không đạt 4x, vì ở batch
4 phần compute trên CPU bắt đầu lớn hơn hẳn.

**So với effective concurrency trong `02-server-results.md` (19.7):** hai số đo hai thứ khác
nhau. Busy slots chỉ đếm request đang được decode, tối đa bằng `--parallel 4`. Effective
concurrency (RPS × latency trung bình) đếm cả request đang xếp hàng. Gauge của server cho thấy
hàng đợi thật lớn hơn nhiều: `requests_deferred` dao động 36–46, cộng 4 request đang xử lý
là khoảng 40–50 request trong hệ thống, gần bằng toàn bộ 50 user.

Tôi tin gauge của server hơn. Con số 19.7 từ locust bị **thấp hơn thực tế** vì hai lý do:
(1) locust chỉ tính latency của 40 request đã hoàn thành, còn khoảng 46 request đang chờ lúc
dừng (chính là những request chờ lâu nhất) không được tính; (2) hàng đợi tăng liên tục suốt
60 s, nên hệ thống chưa đạt trạng thái ổn định mà Little's Law giả định.
