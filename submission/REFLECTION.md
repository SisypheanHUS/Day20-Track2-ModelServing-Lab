# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Dinh Thai Tuan
**MSSV:** 2A202600360
**Ngày submit:** 2026-05-07

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

- **OS:** Windows 11 Pro (10.0.26200)
- **CPU:** Intel Core i7-10750H @ 2.60GHz
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** AVX2
- **RAM:** 31.8 GB
- **Accelerator:** NVIDIA Quadro T2000 Max-Q 4GB VRAM
- **llama.cpp backend đã chọn:** CPU (CUDA runtime DLLs unavailable — CUDA Toolkit v13.0 installed nhưng thiếu runtime binaries, nên fall back sang CPU)
- **Recommended model tier:** Llama-3.2-3B-Instruct (Q4_K_M)

**Setup story** (≤ 80 chữ): Máy có CUDA Toolkit v13.0 nhưng thiếu runtime DLLs (cudart64, cublas64) nên llama-cpp-python CUDA wheel không load được. Fall back sang CPU prebuilt wheel. Cũng cần bật PYTHONIOENCODING=utf-8 vì Windows cp1252 không render Unicode box-drawing characters. Repo dùng Q3_K_L thay Q2_K vì bartowski repo không có Q2_K cho Llama-3.2-3B.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| Llama-3.2-3B-Instruct-Q4_K_M.gguf | 2559 | 348 / 463 | 166.2 / 184.3 | 10784 / 11973 / 12005 | 6.0 |
| Llama-3.2-3B-Instruct-Q3_K_L.gguf | 692 | 563 / 639 | 150.0 / 186.0 | 9959 / 12288 / 12362 | 6.7 |

**Một quan sát** (≤ 50 chữ): Q3_K_L load nhanh hơn 3.7x (692 vs 2559ms) và decode nhanh hơn ~11% (6.7 vs 6.0 tok/s) nhờ file nhỏ hơn → ít memory bandwidth. Tuy nhiên TTFT cao hơn 62%. Trên CPU-only, smaller quant thắng ở throughput nhưng prefill chậm hơn do lower precision arithmetic.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 0.07 | 42000 | 56000 | 56000 | 0 |
| 50 | 0.04 | 51000 | 51000 | 51000 | 0 |

**KV-cache observation**: Dùng llama-cpp-python server (Python binding), không có /metrics endpoint (đó là feature của llama-server native binary). Server xử lý serial — chỉ 1 request tại 1 thời điểm, nên concurrency 50 thực tế giống concurrency 1 với 49 requests xếp hàng. RPS giảm từ 0.07 xuống 0.04 khi tăng users vì queuing delay tăng và 60s run time chỉ đủ cho 2 requests hoàn thành. Đây là minh họa rõ ràng tại sao continuous batching (như trong llama-server native với `--parallel`) quan trọng cho production serving.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub: localhost only
- **N17 (Data pipeline):** stub: in-memory dict
- **N18 (Lakehouse):** stub: in-memory TOY_DOCS
- **N19 (Vector + Feature Store):** stub: TOY_DOCS (keyword overlap scoring)

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: N/A (stub dùng keyword overlap, không có embedding)
- retrieve: 0.1 ms
- llama-server: 8371 – 21308 ms (trung bình ~15760 ms)

**Reflection** (≤ 60 chữ): Bottleneck hoàn toàn nằm ở LLM inference — retrieve chỉ 0.1ms (toy keyword matching), còn llama-server chiếm >99.99% pipeline time. Khớp kỳ vọng: trên CPU-only với 3B model, decode ~6 tok/s nên 200 max_tokens cần ~33s. Trong production, embedding + vector search sẽ thêm 10-50ms nhưng vẫn negligible so với LLM.

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** Chuyển từ Q4_K_M (2.0 GB) sang Q3_K_L (1.6 GB) — quantization nhỏ hơn trên CPU-only system.

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before (Q4_K_M): TPOT P50 = 166.2ms, decode = 6.0 tok/s, load = 2559ms
after  (Q3_K_L): TPOT P50 = 150.0ms, decode = 6.7 tok/s, load = 692ms
speedup: ~1.12× decode, ~3.7× load time
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

Trên i7-10750H CPU-only (không GPU offload), bottleneck là memory bandwidth — mỗi token decode cần đọc toàn bộ model weights từ RAM qua CPU cache. Q3_K_L nhỏ hơn ~20% so với Q4_K_M, nghĩa là mỗi decode step đọc ít bytes hơn → nhanh hơn ~12%. Load time giảm 3.7x vì file nhỏ hơn mmap nhanh hơn.

Điều thú vị: TTFT lại *tăng* ở Q3_K_L (563ms vs 348ms P50). Prefill là compute-bound (matrix multiply toàn bộ prompt tokens cùng lúc), không bandwidth-bound như decode. Q3_K_L cần dequantization phức tạp hơn Q4_K_M (3-bit packing kém aligned hơn 4-bit), nên prefill chậm hơn dù decode nhanh hơn. Đây là tradeoff mà deck §1 đề cập: quantization nhỏ hơn không phải lúc nào cũng nhanh hơn — phụ thuộc phase nào đang chạy.

---

## 6. (Optional) Điều ngạc nhiên nhất

Trên CPU-only, Q3_K_L decode nhanh hơn Q4_K_M (~12%) nhưng prefill chậm hơn (~62%). Nếu chỉ nhìn TPOT thì smaller quant có vẻ "miễn phí" — nhưng TTFT tăng đáng kể. Với workload long-context (RAG), TTFT penalty này sẽ rất rõ.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` (CSV từ locust) đã commit
- [ ] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep) — không làm bonus sweep riêng, dùng so sánh Q4_K_M vs Q3_K_L từ Track 01
- [ ] Ít nhất 6 screenshots trong `submission/screenshots/` — chạy qua CLI, không có screenshots
- [ ] `make verify` exit 0 (chạy ngay trước khi push)
- [ ] Repo trên GitHub ở chế độ **public**
- [ ] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
