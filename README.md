# sql-inference-server

High-performance C++ inference server for T5-based Text-to-SQL with thread pooling, request queuing, and p99 latency tracking.

> Model exported from [Text-to-SQL fine-tuning project](https://github.com/ryanwang-trt/Text-to-Sql) — T5-small fine-tuned on the Spider benchmark dataset.

---

## Overview

This project takes the fine-tuned T5-small model from my Text-to-SQL project and serves it through a production-style C++ inference server. The goal is not just to run the model — it's to handle concurrent requests efficiently, measure real latency under load, and demonstrate systems-level thinking around AI inference.

**What this is NOT:** a Python FastAPI wrapper. Every layer of the serving stack is written in C++.

---

## Architecture (Planned)

```
HTTP Request
     │
     ▼
┌─────────────────┐
│   Drogon HTTP   │  ← C++ HTTP server
│     Server      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Request Queue  │  ← Bounded queue with backpressure
│  (std::queue)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Thread Pool   │  ← Configurable worker threads
│  (N workers)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  ONNX Runtime   │  ← T5 encoder + decoder (C++ API)
│  T5-small model │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Metrics Tracker │  ← Per-request latency, p50/p95/p99
└─────────────────┘
         │
         ▼
    HTTP Response
```

---

## Project Structure

```
sql-inference-server/
├── src/
│   ├── main.cpp          ← entry point, server init
│   ├── server.cpp        ← HTTP routing (Drogon)
│   ├── inference.cpp     ← ONNX Runtime wrapper
│   ├── thread_pool.cpp   ← thread pool implementation
│   ├── queue.cpp         ← request queue with backpressure
│   └── metrics.cpp       ← latency tracking, p50/p95/p99
├── include/
│   ├── inference.h
│   ├── thread_pool.h
│   ├── queue.h
│   └── metrics.h
├── models/
│   └── t5-sql.onnx       ← exported from Python (see scripts/)
├── scripts/
│   └── export_onnx.py    ← exports PyTorch model to ONNX
├── tests/
│   └── load_test.sh      ← wrk load test script
├── CMakeLists.txt
└── README.md
```

---

## Build Plan — Phase by Phase

### Phase 1: ONNX Export + Single-threaded Inference
**Goal:** Get the model running in C++ at all.

- [ ] Export T5-small to ONNX via `scripts/export_onnx.py`
- [ ] Set up CMake with ONNX Runtime dependency
- [ ] Load model and run a single inference in C++
- [ ] Verify output matches Python baseline

**Key challenge:** T5 exports as two separate ONNX graphs (encoder + decoder). Managing the encoder hidden states and decoder autoregressive loop in C++ is non-trivial.

---

### Phase 2: HTTP Server + Request Handling
**Goal:** Accept real HTTP requests.

- [ ] Integrate Drogon HTTP server
- [ ] `POST /predict` endpoint — accepts JSON `{ "question": "...", "db_id": "..." }`
- [ ] Basic request/response pipeline (single-threaded)
- [ ] Error handling and input validation

---

### Phase 3: Concurrency
**Goal:** Handle multiple simultaneous requests without falling over.

- [ ] Implement thread pool with configurable `N` workers
- [ ] Bounded request queue — reject requests when queue is full (503)
- [ ] Thread-safe ONNX Runtime session management
- [ ] Benchmark: measure throughput vs thread count

**Key design decision:** One ONNX session per thread vs shared session — will test both and document the tradeoff.

---

### Phase 4: Latency Metrics
**Goal:** Know exactly how the server performs under load.

- [ ] Per-request timer using `std::chrono::high_resolution_clock`
- [ ] Rolling window: track last N requests
- [ ] Compute p50, p95, p99 latency
- [ ] `GET /metrics` endpoint — returns JSON with current stats
- [ ] Load test with `wrk` — document results at different concurrency levels

---

## API (Planned)

### `POST /predict`
```json
// Request
{
  "question": "how many employees are in each department",
  "db_id": "company"
}

// Response
{
  "sql": "SELECT department, COUNT(*) FROM employees GROUP BY department",
  "latency_ms": 42.3
}
```

### `GET /metrics`
```json
{
  "requests_total": 1024,
  "requests_in_flight": 3,
  "latency_p50_ms": 38.1,
  "latency_p95_ms": 71.4,
  "latency_p99_ms": 103.2,
  "queue_depth": 0
}
```

---

## Stack

| Component | Technology |
|-----------|-----------|
| HTTP Server | Drogon |
| Model Runtime | ONNX Runtime (C++ API) |
| Build System | CMake |
| Load Testing | wrk |
| Model Export | PyTorch → ONNX (Python) |

---

## Results (To Be Filled)

| Workers | Requests/sec | p50 (ms) | p95 (ms) | p99 (ms) |
|---------|-------------|----------|----------|----------|
| 1       | —           | —        | —        | —        |
| 2       | —           | —        | —        | —        |
| 4       | —           | —        | —        | —        |
| 8       | —           | —        | —        | —        |

---

## Why C++?

Python is the standard for ML inference, but C++ gives direct control over threading, memory, and latency. This project is about understanding what happens *below* the FastAPI layer — how requests get queued, how threads share a model, and where the latency actually comes from.

---

## Related

- [Text-to-SQL fine-tuning](https://github.com/ryanwang-trt/Text-to-Sql) — the model this server runs
