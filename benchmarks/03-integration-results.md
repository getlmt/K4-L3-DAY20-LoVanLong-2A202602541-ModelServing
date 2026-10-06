# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 5235.3 | 5235.4 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 4484.7 | 4484.8 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 4502.8 | 4502.9 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **4740.9** · total **4741.0**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

| Day | Piece | Real hay stub? |
|:--|:--|:--|
| N16 Cloud/IaC | chạy `localhost` trên laptop, không có k8s/Compose | stub |
| N17 Data pipeline | corpus là list `TOY_DOCS` viết sẵn trong code, không có job nạp dữ liệu | stub |
| N18 Lakehouse | không có Delta/Iceberg, `TOY_DOCS` đứng thay | stub |
| N19 Vector + features | `retrieve()` dùng keyword overlap, không có vector index và embedding server (embed = 0.0 ms) | stub |
| N20 Serving | `llama-server` Gemma 4 E2B UD-Q4_K_XL, CPU, `--parallel 4` | **real** |

**Stage chiếm nhiều nhất là `llm` (100%), đúng như tôi đoán, nhưng lý do thì không như tôi
nghĩ.** Server tự báo mỗi câu chỉ tốn prefill + decode khoảng 2.2–2.9 s (ví dụ câu 1: prefill
149 tok/1180 ms + decode 30 tok/1749 ms), trong khi client đo `llm` = 4.5–5.2 s. Cả 3 câu đều
dư khoảng **2.3 s nằm ngoài model**. Nguyên nhân: pipeline gọi `http://localhost:8080`;
trên Windows `localhost` phân giải ra `::1` (IPv6) trước, nhưng server chỉ listen
`127.0.0.1`, nên mỗi kết nối mới phải chờ IPv6 thất bại rồi mới chuyển sang IPv4. Tôi đo riêng:
`GET /health` qua `localhost` mất 2241–2292 ms, qua `127.0.0.1` mất 187–192 ms, qua một
`httpx.Client` giữ kết nối thì từ request thứ 2 chỉ còn 0.5 ms.

Tách trung bình 4741 ms của stage `llm`: khoảng 2.3 s (48%) là kết nối, 1.0 s (21%) là
prefill, 1.5 s (31%) là decode.

**Nếu phải giảm latency 2 lần, tôi tấn công phía client trước, chưa cần đụng tới model.**
1. Gọi `127.0.0.1` hoặc dùng một `httpx.Client` giữ kết nối (keep-alive), để bỏ khoảng 2.1 s
   mỗi request.
2. Giữ system prompt và context giống hệt nhau từng byte để server dùng lại prefix đã cache.

Tôi chạy thử `.\lab.ps1 pipeline --base-url http://127.0.0.1:8080` ngay sau lần chạy trên:
mean `llm` giảm từ **4741 xuống 1810 ms (2.6x)**. Lần đó prefill chỉ còn 5 token, khoảng
120 ms, vì server dùng lại KV cache của đúng các prompt đó (prefix caching). Phần còn lại gần
như toàn là decode (khoảng 1.5 s cho 24–30 token ở khoảng 17 tok/s). Muốn giảm tiếp thì phải
tăng tốc decode (GPU offload, xem `02-server-results.md`) hoặc giới hạn số token output. Với
retrieval thật (N19 vector index), stage `embed`/`retrieve` sẽ không còn là 0 ms; khi đó nên
đo lại xem chúng có vượt decode không.
