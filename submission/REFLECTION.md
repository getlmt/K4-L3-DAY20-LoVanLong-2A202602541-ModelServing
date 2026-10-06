# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Lò Văn Long
**MSSV:** 2A202602541
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 Home Single Language 22H2 (build 19045); lab chạy bằng PowerShell 7.6.6
- **CPU:** AMD Ryzen 7 4800H with Radeon Graphics (Zen 2)
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** SSE3, AVX, AVX2; không có AVX-512 (llama.cpp tự chọn `ggml-cpu-haswell.dll`)
- **RAM:** 15.4 GB (2 × 8 GB DDR4-3200, dual-channel)
- **Accelerator:** NVIDIA GeForce GTX 1650 4 GB (GDDR6). Có GPU, nhưng base track chủ động chạy CPU-only (xem setup story); GPU chỉ dùng ở bonus §6
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-cpu-x64.zip`
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi (`runtime_environment: local`), không dùng Colab/Kaggle.

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

Có ba chỗ phải xử lý.
1. `.\lab.ps1` lỗi cú pháp trên Windows PowerShell 5.1: file lưu UTF-8 không BOM, nên dấu
   "—" bị đọc thành `â€”` và ký tự `”` trong đó đóng chuỗi sớm. Tôi cài PowerShell 7 bằng
   winget và chạy `.\lab.ps1` trong `pwsh`.
2. Python mặc định dùng cp1252 nên crash khi in `✓`/`─`. Tôi đặt `PYTHONUTF8=1`.
3. Setup tự chọn bản CUDA vì máy có GTX 1650. Tôi tải bản CPU bằng
   `fetch-runtime.py --asset llama-b10488-bin-win-cpu-x64.zip` để base track đo trên CPU.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 3868 | 386 / 429 | 58.7 / 59.0 | 4057 / 4137 / 4137 | 17.0 |
| UD-Q2_K_XL | 2.24 | 3792 | 498 / 549 | 45.6 / 46.0 | 3395 / 3423 / 3423 | 21.9 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

Bản 2-bit nhỏ hơn 0.73 GB (-25%) và decode nhanh hơn 1.29x (21.9 so với 17.0 tok/s), nhưng
TTFT chậm hơn khoảng 29% (498 so với 386 ms). Tôi hỏi cùng 3 câu trên `serve` và
`serve.py --compare` (temperature 0): bản 2-bit giải thích sai TPOT và trả lời dài hơn
35-92% số token, nên cả 3 câu đều xong chậm hơn bản 4-bit. Kết luận: không đáng dùng, vì
máy có 15.4 GB RAM nên dung lượng không phải giới hạn.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.61 | 14000 | 19000 | 19000 | 8.3 | 0.0% |
| 50 | 0.70 | 30000 | 53000 | 56000 | 19.7 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.16×
- **P95 tăng:** 2.79×
- **Effective concurrency ở 50 users:** 19.7 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.93 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Server bão hoà ngay từ 10 user (effective concurrency 8.3, lớn hơn 4 slot).
- **Bằng chứng:** tốc độ decode tổng đứng yên ở khoảng 34 tok/s (busy 3.93/4), nên tải ×5
  chỉ cho RPS ×1.16 còn P95 ×2.79.
- **Phần tăng thêm là queue time:** min latency vẫn khoảng 5–6 s, trong khi
  `requests_deferred` lên tới 36–46.
- **Goodput:** với SLO P95 ≤ 20 s, ở 10 user goodput là 0.61 RPS; ở 50 user chỉ còn ≤ 0.35 RPS.
- **Knob đổi trước:** GPU offload (`-ngl 99`), vì decode bị chặn bởi bandwidth (DDR4 khoảng
  27 GB/s so với VRAM khoảng 192 GB/s).

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | chạy `localhost` trên laptop, không có k8s/Compose | stub |
| N17 Data pipeline | `TOY_DOCS` viết sẵn trong code, không có job nạp dữ liệu | stub |
| N18 Lakehouse | không có Delta/Iceberg, `TOY_DOCS` đứng thay | stub |
| N19 Vector + features | `retrieve()` dùng keyword overlap, không có vector index/embedding | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 4740.9 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM chiếm 100% như tôi đoán, nhưng khoảng 2.3 s (48%) trong đó là chờ kết nối: trên Windows
`localhost` thử `::1` trước, mà server chỉ listen `127.0.0.1`. Phần model thật gồm prefill
khoảng 1.0 s và decode khoảng 1.5 s. Để giảm 2x, tôi gọi `127.0.0.1` và giữ prompt ổn định để
dùng prefix cache. Chạy thử với `--base-url http://127.0.0.1:8080`: 4741 → 1810 ms.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** hạ `-t` từ 16 (dùng hết luồng logical) xuống 8 (= số core vật lý). Đo decode
`tg128` trên UD-Q4_K_XL, CPU-only, bằng `.\lab.ps1 tune`. Toàn bộ đường cong:
`-t 1` 8.5 · `-t 4` 17.3 · `-t 8` 17.5 · `-t 16` 16.0 · `-t 32` 14.3 tok/s.

```
before:  16.0 tok/s  (-t 16)
after:   17.5 tok/s  (-t 8)
speedup: 1.09×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

**Decode trên CPU bị chặn bởi memory bandwidth, không phải FLOPs.** Tôi đọc kích thước tensor
trong file GGUF. Mỗi token phải đọc lại khoảng 1.55 GB weights. Bảng `per_layer_token_embd`
(1.62 GB) không tính vào vì mỗi token chỉ tra 1 dòng. `token_embd` (0.28 GB) thì bị đọc toàn
bộ, vì Gemma dùng chung nó làm LM head.

Lấy 1.55 GB nhân tok/s ra lượng đọc thực tế: 1 thread 13.2 GB/s, 4 thread 26.9 GB/s, 8 thread
27.2 GB/s. Máy có 2 thanh DDR4-3200 dual-channel, trần lý thuyết là 51.2 GB/s; khoảng 27 GB/s
(53%) là mức đọc thực tế đạt được. Một core không tự kéo hết bandwidth: nó vừa phải giải nén
và nhân, vừa chỉ giữ được ít yêu cầu đọc RAM đang chờ cùng lúc. Nhưng 4 core đã đủ làm đầy 2
memory channel. Vì vậy knee nằm ở 4 thread, sớm hơn kỳ vọng "knee ở số core vật lý" của deck.
Thread thứ 5 đến 8 chỉ xếp hàng chờ cùng một nguồn dữ liệu, nên 4 → 8 thread chỉ thêm 1%.

**Vượt 8 thread thì chậm đi.** `-t 16` đặt 2 luồng SMT lên cùng một core: chung L1/L2, chung
đơn vị load/store, nên không thêm byte/s nào. Trong khi đó threadpool của llama.cpp phải
barrier-sync 16 thread sau mỗi op của graph (hàng trăm op mỗi token), và mỗi thread nhận phần
việc nhỏ hơn. Kết quả là 16.0 tok/s, chậm hơn 9%. `-t 32` vượt số luồng phần cứng: Windows
phải chia lượt chạy, và thread xong sớm phải chờ ở barrier cho thread đang bị tạm dừng. Kết
quả là 14.3 tok/s, chậm hơn 18%.

Mặc định của lab (`-t` = số core vật lý = 8) đã là tối ưu; bài học là đừng "dùng hết 16
luồng". Kiểm chứng chéo: bản 2-bit đọc ít hơn 1.46x byte mỗi token (1.07 so với 1.55 GB)
nhưng chỉ decode nhanh hơn 1.29x (khoảng 23 GB/s), vì kernel 2-bit tốn compute giải nén hơn.
Đây cũng là lý do TTFT (prefill, compute-bound) của bản 2-bit chậm hơn.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Gần một nửa latency của RAG pipeline không nằm ở model mà nằm ở chữ `localhost`. Trên Windows,
`localhost` phân giải ra `::1` trước, mà server chỉ listen IPv4, nên mỗi kết nối mới mất khoảng
2.25 s (đo riêng: 2241–2292 ms qua `localhost`, 187–192 ms qua `127.0.0.1`). Locust cũng gọi
`localhost`, nhưng vì giữ kết nối nên tối đa chỉ request đầu của mỗi user chịu độ trễ này.

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
- [ ] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [x] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Có dùng **Claude Code** (Claude của Anthropic, chạy trong VSCode) vào các việc sau:
- Chạy các lệnh lab theo hướng dẫn và chụp ảnh từng cửa sổ terminal thật bằng Windows API
  (`PrintWindow`).
- Chẩn đoán lỗi môi trường: `lab.ps1` không có BOM trên PowerShell 5.1, Python dùng cp1252,
  `localhost` phân giải ra IPv6 trên Windows.
- Đo thêm số liệu phụ: kích thước tensor GGUF, cấu hình RAM, băng thông VRAM.
- Soạn nháp các phần nhận xét trong `benchmarks/*.md` và REFLECTION.

Mọi con số đều do script của lab sinh ra trên máy tôi, không sửa tay. Tôi đã đọc lại, đối chiếu
số liệu và chịu trách nhiệm về nội dung.
