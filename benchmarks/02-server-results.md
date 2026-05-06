# 02 — Server Benchmark Results

Comparison of `llama-server` performance and internal Prometheus metrics under different concurrency levels.

| Concurrency | Avg Req/s | Avg Tokens/sec | Max KV Cache % | P50 Latency (ms) | P95 Latency (ms) |
|-------------|-----------|----------------|----------------|------------------|------------------|
| 10          | 1.27      | 109.4          | 0.0%           | 5500             | 9000             |
| 50          | 0.88      | 75.9           | 0.0%           | 16000            | 29000            |
