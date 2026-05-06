# 02 — llama-server Load Test Results

Server: llama-cpp-python (Python binding), single-threaded request handling.
Model: Llama-3.2-3B-Instruct-Q4_K_M.gguf
Settings: `n_threads=6`, `n_ctx=2048`, `n_gpu_layers=0` (CPU only).

## Locust Results

| Concurrency | Duration | Total Reqs | Failures | Avg (ms) | Min (ms) | Max (ms) | P50 (ms) | P95 (ms) | P99 (ms) | RPS |
|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|--:|
| 10 | 60s | 4 | 0 | 35758 | 15862 | 55842 | 42000 | 56000 | 56000 | 0.07 |
| 50 | 60s | 2 | 0 | 43587 | 36644 | 50531 | 51000 | 51000 | 51000 | 0.04 |

## Observations

- The Python server handles requests serially (no continuous batching), so concurrency >1 just adds queuing delay.
- At concurrency 10, only 4 requests completed in 60s; at concurrency 50, only 2.
- Average response time increased from 35.8s to 43.6s due to queuing.
- This demonstrates why native llama-server with `--parallel` (continuous batching) is essential for any multi-user scenario.
- No /metrics endpoint available (Python binding limitation) — native llama-server binary provides Prometheus metrics.

## Smoke Test

Single request latency: ~14925ms for a short chat completion (80 max tokens).
