# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Lê Kim Dũng
**Mssv:** 2A202600100
**Cohort:** A20-K1
**Ngày submit:** 2026-05-06

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

> Paste output của `python 00-setup/detect-hardware.py` vào đây, hoặc điền thủ công:

- **OS:** Windows 10 (AMD64)
- **CPU:** 16 physical cores
- **Cores:** 16 physical / 16 logical
- **CPU extensions:** AVX2 / AVX-512
- **RAM:** 16 GB (ước tính)
- **Accelerator:** CPU only
- **llama.cpp backend đã chọn:** CPU (AVX2/AVX512 tuning)
- **Recommended model tier:** TinyLlama-1.1B

**Setup story** (≤ 80 chữ): những gì cần thay đổi để lab chạy được trên máy bạn (vd: dùng WSL2, install CUDA Toolkit, fall back sang Vulkan vì ROCm phiên bản kén, tắt antivirus để pip install nhanh hơn, v.v.):

Cần cài đặt build tools cho Windows (CMake, MSVC) để biên dịch llama.cpp từ mã nguồn nhằm hỗ trợ endpoint /metrics. Ngoài ra, cần cấu hình $env:PYTHONIOENCODING='utf-8' để xử lý tiếng Việt trong pipeline RAG và sửa lỗi Regex trong các script benchmark trên Windows.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

> Paste bảng từ `benchmarks/01-quickstart-results.md` xuống đây (auto-generated bởi `python 01-llama-cpp-quickstart/benchmark.py`).

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|---:|---:|---:|---:|---:|
| tinyllama (Q4_K_M) | 499 | 77 / 91 | 17.5 / 30.1 | 1139 / 1205 / 1211 | 57.3 |
| tinyllama (Q2_K)   | 101 | 129 / 162 | 15.1 / 17.5 | 1065 / 1134 / 1155 | 66.4 |

**Một quan sát** (≤ 50 chữ): Q4_K_M vs Q2_K trên máy bạn — số liệu nói gì? Quality đáng đánh đổi không?

Model Q4_K_M chậm hơn Q2_K một chút nhưng chất lượng trả lời tiếng Việt tốt hơn hẳn. Với CPU 16 nhân, tốc độ giải mã đạt mức rất tốt (~60 tok/s), hoàn toàn đủ cho trải nghiệm chat thời gian thực.

---

## 3. Track 02 — llama-server load test

> Chạy 2 lần locust ở concurrency 10 và 50, paste tóm tắt bên dưới.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 1.27 | 5500 | 9000 | - | 0 |
| 50 | 0.88 | 16000 | 29000 | - | 0 |

**KV-cache observation** (từ `record-metrics.py`): peak `llamacpp:kv_cache_usage_ratio` ở concurrency 50 = _0.0%_, nghĩa là bộ nhớ context chưa phải là điểm nghẽn, hệ thống đang bị giới hạn hoàn toàn bởi tốc độ xử lý của CPU khi phải xử lý song song quá nhiều yêu cầu.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub: localhost only
- **N17 (Data pipeline):** batch job (pipeline.py)
- **N18 (Lakehouse):** stub: JSONL corpus (corpus_vn.jsonl)
- **N19 (Vector + Feature Store):** Qdrant index (Day 19 Searcher)

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed + retrieve: < 20ms
- llama-server: 3500 - 4500ms

**Reflection** (≤ 60 chữ): bottleneck nằm ở đâu? Có khớp với kỳ vọng không?

Bottleneck nằm ở bước suy luận LLM (llama-server). Kết quả này khớp với kỳ vọng vì việc tìm kiếm vector đã được tối ưu hóa tốt bởi Qdrant, trong khi việc tạo chữ yêu cầu sức mạnh tính toán CPU liên tục.

---

## 5. Bonus — The single change that mattered most

> **Most important section.** Pick **một** thay đổi từ bonus track (build flag, thread sweep, quant pick, GPU offload, KV-cache quantization, speculative decoding, bất cứ challenge nào trong `BONUS-llama-cpp-optimization/CHALLENGES.md`) đã tạo ra speedup lớn nhất trên máy bạn.

**Change:** Tìm ra số lượng luồng tối ưu (-t 16) thông qua Thread Sweep.

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before: 22.7 tok/s (-t 1)
after:  61.7 tok/s (-t 16)
speedup: ~2.7×
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

Tăng số lượng luồng giúp tận dụng tối đa các nhân vật lý của CPU. Tuy nhiên, nếu tăng lên 32 luồng (vượt quá số nhân vật lý), tốc độ sẽ giảm do các luồng phải tranh giành băng thông bộ nhớ (memory bandwidth) - vốn là điểm nghẽn chính của LLM khi chạy trên CPU. Tôi cũng đã thử Speculative Decoding nhưng kết quả chậm hơn vì model chính quá nhỏ (1.1B), không đủ bù đắp chi phí overhead của việc chạy hai model song song trên cùng một CPU.

---

## 6. (Optional) Điều ngạc nhiên nhất

_(1–2 câu — không bắt buộc, nhưng người grader đọc tất cả)_

Điều ngạc nhiên nhất là mặc dù CPU có 16 nhân, nhưng tốc độ không tăng tuyến tính khi dùng hết 32 luồng mà bị nghẽn ở băng thông bộ nhớ. Điều này giúp tôi hiểu rõ hơn về kiến trúc Von Neumann và hạn chế của phần cứng hiện tại.

---

## 7. Self-graded checklist

- [ ] `hardware.json` đã commit
- [ ] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [ ] `benchmarks/01-quickstart-results.md` đã commit
- [ ] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [ ] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [ ] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [ ] `make verify` exit 0 (chạy ngay trước khi push)
- [ ] Repo trên GitHub ở chế độ **public**
- [ ] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
