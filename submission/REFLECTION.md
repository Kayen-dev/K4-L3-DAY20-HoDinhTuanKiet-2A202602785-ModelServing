# Reflection — Day 20 Lab (Personal Report)

**Họ Tên:** Hồ Đình Tuấn Kiệt
**MSSV:** 2A202602785
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

- **OS:** Microsoft Windows 11 Home Single Language 64-bit, build 26200
- **CPU:** Intel Core i5-8265U @ 1.60 GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** AVX2
- **RAM:** 19.9 GB
- **Accelerator:** NVIDIA GeForce MX110, 2048 MiB; llama.cpp Vulkan offload (`ngl=99`)
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-vulkan-x64.zip`
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** `Q4_K_M` primary + `UD-Q2_K_XL` compare

**Chạy ở đâu:** Laptop cá nhân, chạy local; không dùng Colab/Kaggle.

**Setup story** (≤ 80 chữ): Tôi chọn Qwen3.5 0.8B để rút ngắn thời gian tải và
load test. Hugging Face tải quá chậm nên tôi tải thủ công hai file GGUF. PowerShell
5.1 gặp lỗi mã hóa với `lab.ps1`, vì vậy tôi gọi trực tiếp các Python script tương
ứng. Runtime prebuilt Vulkan chạy được, không cần compile.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 7108 | 758 / 867 | 50.3 / 56.2 | 3927 / 4078 / 4078 | 19.9 |
| UD-Q2_K_XL | 0.39 | 5522 | 1036 / 1119 | 298.9 / 302.9 | 19886 / 20179 / 20179 | 3.3 |

**Quan sát** (≤ 60 chữ): Q2 nhỏ hơn 22% nhưng không nhanh hơn: decode chậm hơn
`6.03×` và TTFT cũng xấu hơn. Với cùng câu hỏi, Q4 bám chủ đề và có cấu trúc; Q2
nhầm TTFT/TPOT thành khái niệm training, lặp ý và chạm giới hạn token. Tiết kiệm
0.11 GB không đáng đổi lấy tốc độ và chất lượng này.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.34 | 25000 | 43000 | 43000 | 8.4 | 0.0% |
| 50 | 0.38 | 34000 | 53000 | 53000 | 11.2 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** `1.09×` (22% mức tuyến tính)
- **P95 tăng:** `1.23×`
- **Effective concurrency ở 50 users:** `11.2` so với `--parallel=4` slots
- **Peak `llamacpp:n_busy_slots_per_decode`:** `3.82 / 4` slots (95%)

**Saturation reading** (≤ 80 chữ): Server bão hòa trong khoảng 10–50 users: tải
tăng `5×` nhưng RPS chỉ tăng `1.09×`, trong khi P95 tăng `1.23×`. `3.82/4` busy
slots và 46 request deferred chứng minh phần latency tăng là queue time. Với SLO
45 giây, goodput giảm khoảng `0.34 → 0.30 req/s`. Tôi sẽ giảm output budget
48/96 xuống 32/64 token trước để giải phóng slot sớm hơn.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Local single-process thay cho cluster | Stub |
| N17 Data pipeline | `TOY_DOCS`, không có ingestion pipeline | Stub |
| N18 Lakehouse | Python list trong bộ nhớ, không có lakehouse | Stub |
| N19 Vector + features | Keyword overlap, không có vector index | Stub |
| N20 Serving | `llama-server` local, OpenAI-compatible API | Real |

**Latency split** (mean của 3 query):

- embed: `0.0 ms`
- retrieve: `0.1 ms`
- llm: `8583.5 ms`
- total: `8583.7 ms`
- **stage chiếm nhiều nhất:** `llm` (gần 100% total)

**Reflection** (≤ 60 chữ): Kết quả khớp kỳ vọng vì corpus chỉ có sáu document
trong RAM, còn LLM chiếm gần toàn bộ latency. Muốn giảm pipeline `2×`, tôi sẽ yêu
cầu câu trả lời ngắn và giảm `max_tokens` từ 200 xuống khoảng 64. Decode hiện sinh
62–110 token và tốn 3124–5518 ms; tối ưu retrieval chỉ tiết kiệm 0.1 ms.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

**Change:** Dùng `Q4_K_M` thay cho `UD-Q2_K_XL` trên runtime Vulkan.

```text
before:  3.3 tok/s  (UD-Q2_K_XL)
after:   19.9 tok/s (Q4_K_M)
speedup: 6.03×
```

**Tại sao nó work:** Q2 giảm kích thước model từ 0.50 xuống 0.39 GB, nhưng ít bit
chỉ giúp khi decode bị giới hạn chủ yếu bởi bandwidth. Trên MX110 2 GB với Vulkan
offload, format Q2 tạo thêm chi phí unpack/dequantize và không ánh xạ hiệu quả bằng
Q4 vào kernel hiện tại. Phần tính toán thêm lớn hơn lợi ích của việc đọc ít hơn 22%
trọng số, nên TPOT P50 tăng từ 50.3 lên 298.9 ms.

Kết quả này trái với trực giác “ít bit luôn nhanh hơn”, nhưng được xác nhận bởi cả
decode rate (`3.3` so với `19.9 tok/s`) và E2E P50 (`19886` so với `3927 ms`). Tăng
CPU threads không sửa được bottleneck: sweep `-t 1..16` gần như phẳng ở
`20.5–21.1 tok/s` vì `ngl=99` đã chuyển phần decode chính sang GPU.

---

## 6. Bonus  *(optional)*

Không thực hiện bonus; báo cáo tập trung vào base track.

---

## 7. Điều làm tôi ngạc nhiên nhất  *(optional)*

Q2 nhỏ hơn nhưng chậm hơn Q4 tới `6.03×`. Kết quả này cho thấy số bit không tự động
quyết định tốc độ; hiệu quả kernel dequantization và bottleneck phần cứng mới là yếu
tố quyết định trên runtime cụ thể.

---

## 8. Self-check trước khi push

- [x] `hardware.json` và `models/active.json` có số liệu local
- [x] Đủ report benchmark, tuning, serving, batching và integration
- [x] Không còn placeholder bắt buộc trong `benchmarks/*.md`
- [x] Sáu ảnh đại diện cho năm checkpoint; mục 3 tách thành `03a-serve.png` và `03b-smoke.png`
- [x] Không commit model weights, runtime, `.venv` hay `.env`
- [x] Tên repo đúng `K4-L3-DAY20-HoDinhTuanKiet-2A202602785-ModelServing`
- [ ] Paste URL public vào VinUni LMS — thao tác thủ công sau khi push

### Screenshot mapping

1. `01-hardware-probe.png`
2. `02-bench.png`
3. `03a-serve.png` + `03b-smoke.png`
4. `04-locust-10.png`
5. `05-locust-50.png`

---

## 9. Khai báo sử dụng AI

Tôi dùng **OpenAI Codex** để điều phối các lệnh lab, debug lỗi tải Hugging Face và
mã hóa PowerShell, kiểm tra tính nhất quán giữa artifact, đồng thời hỗ trợ diễn đạt
phần phân tích từ số liệu đo trên máy của tôi. AI không tạo số liệu hoặc screenshot
giả; các bảng, CSV, metrics và ảnh terminal đều đến từ những lần chạy thật trong
repo này.
