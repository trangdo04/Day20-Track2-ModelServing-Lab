# REFLECTION — Lab 20 (Personal Report)

> Báo cáo này chỉ phản ánh kết quả trên máy cá nhân của tôi. Không so sánh với bạn khác lớp.

---

**Họ tên:** Đỗ Thị Thùy Trang
**Cohort:** A20-K1
**Ngày submit:** 2026-05-06

---

## 1. Hardware & backend

- **OS:** Windows 10
- **CPU:** AMD Ryzen 5 5600H
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** AVX2, FMA
- **RAM:** 15.4 GB
- **Accelerator:** NVIDIA GeForce RTX 3050 Laptop GPU (4 GB)
- **Chọn backend:** CPU-only (Ollama standalone)
- **Model dùng:** qwen2.5:1.5b (CPU inference)

**Setup story:**
Ban đầu mình thử cài `llama-cpp-python` trên Windows, nhưng gặp lỗi build và thiếu toolchain/CMake/Visual Studio. Sau đó mình chuyển sang giải pháp ổn định hơn: dùng Ollama standalone CPU để hoàn thành bài lab.

---

## 2. Track 01 — Quickstart benchmarks

> Kết quả thu thập từ benchmark CPU trên model Qwen2.5-1.5B.

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|---:|---:|---:|---:|---:|
| qwen2.5-1.5b-instruct-q4_k_m.gguf | 2964 | 386 / 469 | 84.1 / 152.3 | 5622 / 6980 / 7186 | 11.9 |
| qwen2.5-1.5b-instruct-q2_k.gguf | 731 | 526 / 675 | 72.7 / 77.2 | 5154 / 5346 / 5398 | 13.8 |

**Nhận xét:**
- Q2_K có throughput cao hơn, nhưng TTFT và độ ổn định đầu tiên thấp hơn.
- Q4_K_M chậm hơn khoảng 15% nhưng cho chất lượng output ổn định hơn.
- Với môi trường CPU-only, việc chọn quantization là thỏa hiệp giữa tốc độ và độ chính xác.

---

## 3. Track 02 — llama-server load test

> Chạy load test với concurrency 10 và 50.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 1.44 | 5100 | 7800 | 9600 | 0 |
| 50 | 1.53 | 9400 | 23000 | 25000 | 0 |

**Quan sát KV-cache:**
- `llamacpp:n_busy_slots_per_decode` peak ~3.67
- `llamacpp:requests_deferred` = 0.0

Dù latency tăng mạnh khi concurrency lên 50, hệ thống vẫn không deferred request nào. Điều này chứng tỏ cấu hình slot và bộ nhớ đủ để tiếp nhận request đồng thời mà không gây lỗi.

---

## 4. Track 03 — Milestone integration

* N16 (Cloud/IaC): stub chạy local
* N17 (Data pipeline): stub in-memory dict
* N18 (Lakehouse): stub SQLite
* N19 (Vector + Feature Store): stub TOY_DOCS

**Tóm tắt tốc độ:**
- retrieve dữ liệu: ~0.0–0.1 ms
- inference llama-server: ~2916–6532 ms
- tổng pipeline: ~2916–6532 ms

**Reflection:**
Inference chiếm phần lớn độ trễ. Khi dữ liệu truy xuất gần như tức thì do stub, tổng thời gian hầu như phụ thuộc vào llama-server. Đây là kịch bản điển hình khi tích hợp LLM vào pipeline.

---

## 5. Bonus — The single change that mattered most

**Change:**
Tuning số lượng thread để phù hợp với CPU và memory bandwidth.

**Before vs after:**
```
before:
  -t 1  → 6.49 tok/s
  -t 6  → 16.07 tok/s

after:
  -t 12 → 17.21 tok/s
speedup: ~2.65×
```

**Vì sao hiệu quả:**
Decode LLM bị giới hạn nhiều bởi bộ nhớ và cache, không chỉ compute. Khi tăng thread đến gần số core/logic phù hợp, CPU tận dụng tốt hơn bộ nhớ và cache. Nếu oversubscribe quá nhiều thread thì performance giảm do tranh chấp cache và context switching.

