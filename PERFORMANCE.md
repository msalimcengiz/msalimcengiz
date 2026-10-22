# Performance and correctness report

Measured numbers for the six portfolio projects. Every number below comes from a script in the project's repository, so you can reproduce it with Docker and `make`.

- `make bench` measures speed with no faults injected.
- `make chaos` kills processes and brokers with `SIGKILL` under load, then checks every message.

Raw results are in each repository's `bench/results/` or `chaos/results/`.

**Machine:** Apple M4 laptop. A single Docker VM (Colima, 4 vCPU, 6–8 GiB) runs the whole stack of each project: API, workers, broker, database and load generator. They share the same four CPUs, so these are numbers from a laptop, not a tuned cluster. They are lower bounds that show where each design's limits are.

## Summary

| project | throughput | latency | correctness under failure |
|---|---|---|---|
| [spring-kafka-order-saga](https://github.com/msalimcengiz/spring-kafka-order-saga) | **830–910 complete sagas/s** across 3 services; API accepts 4,300 orders/s | **p95 488 ms** end to end at 500 orders/s | 15,516 orders while 28 service instances and the Kafka broker were killed: **0 money or stock drift** |
| [go-event-ingestor](https://github.com/msalimcengiz/go-event-ingestor) | **50,800 events/s**, 2.8× direct inserts | p95 45 ms | 646,280 events while 14 containers were killed: **0 lost, 0 duplicated** |
| [hono-mail-queue](https://github.com/msalimcengiz/hono-mail-queue) | **940 emails/s** bulk first pass | password reset **p95 9 ms** behind a 20,000-email campaign | 13,733 emails while workers, the API and RabbitMQ were killed: **0 lost** |
| [fastapi-celery-imports](https://github.com/msalimcengiz/fastapi-celery-imports) | **58,000–61,000 rows/s** (2M rows) | 2M rows in 33–35 s | 1.1M rows while workers and RabbitMQ were killed: 42 chunks re-ran, **none applied twice** |
| [laravel-marketplace-search](https://github.com/msalimcengiz/laravel-marketplace-search) | **11,500 docs/s** index build; 83 searches/s with facets | category pages **20× faster** than Postgres FTS (10 ms vs 215 ms p50) | 2,739 searches during a full reindex and alias swap: **0 errors** |
| [ads-insights-agent](https://github.com/msalimcengiz/ads-insights-agent) | **710–860 runs/s** through the job queue | 1.2–1.4 ms queue overhead per run | 10,000 runs per batch: **0 claimed twice, 0 retried** |

## spring-kafka-order-saga

Java 25, Spring Boot 4, Kafka 4.2, Postgres 17. Each order is a saga across order, payment and inventory services:

- 4 Kafka messages
- 5 database transactions
- a transactional outbox on every write

**Benchmark** (`make bench`): balances and stock are topped up first, so every order completes.

| run | orders | accepted/s | completed sagas/s | saga p50 | saga p95 | saga p99 | not completed |
|---|---:|---:|---:|---:|---:|---:|---:|
| 64 clients, closed loop | 258,686 | 4,310 | **833** (913 in a second run) | – | – | – | 0 |
| 100 orders/s | 6,001 | 100 | 100 | 253 ms | 380 ms | 429 ms | 0 |
| 500 orders/s | 29,978 | 500 | 498 | **320 ms** | **488 ms** | 827 ms | 0 |

Saga latency runs from order created to `COMPLETED`, through two outbox relays and four Kafka hops. In the closed-loop run the API accepts orders about 5× faster than the sagas finish. The surplus waits in the outbox and in Kafka, so that run reports capacity, not latency.

**Chaos** (`make chaos`): 20 clients for 90 s. During the run, 28 service instances were killed with `SIGKILL`, the Kafka broker was killed once, and 5% of handler attempts failed on purpose.

| orders | completed | declined | out of stock, refunded | sagas unfinished | money drift | stock drift | dead letters |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 15,516 | 664 | 13,328 | 1,524 | **0** | **0** | **0** | **0** |

## go-event-ingestor

Go, Redis Streams, Postgres. The API appends events to a Redis Stream, and workers write them to Postgres in batches with `COPY`.

**Benchmark** (`make bench`): 100 clients for 30 s, 10 events per request. It compares the same binary with `WRITE_MODE=direct` (one `INSERT` per event).

| mode | events/s | p50 | p95 | p99 | Postgres commits per event |
|---|---:|---:|---:|---:|---:|
| direct | 18,175 | 54.5 ms | 63.4 ms | 73.2 ms | 0.98 |
| buffered | **50,811** | 13.3 ms | 45.1 ms | 66.3 ms | **0.001** |

**Chaos** (`make chaos`): 50 clients for 60 s. Both workers were killed every 6 s and the API every 18 s. Clients retry with the same event ids.

| events stored | acknowledged but lost | duplicate rows |
|---:|---:|---:|
| 646,280 | **0** | **0** |

## hono-mail-queue

TypeScript, Hono, RabbitMQ (quorum queues), Postgres. Email has priority classes, a TTL retry ladder, a delivery limit with a dead-letter queue, and a transactional outbox.

**Benchmark** (`make bench`): a 20,000-email bulk campaign plus 300 password resets at about 30/s. 10% of SMTP calls fail temporarily.

| class | emails | first-try latency p50 | p95 | p99 |
|---|---:|---:|---:|---:|
| transactional | 300 | 3 ms | **9 ms** | 12 ms |
| bulk | 20,000 | 8.4 s | 16.8 s | 17.6 s |

Bulk first-pass throughput was 943 emails/s. Password resets were not slowed down by the 20,000 bulk emails queued ahead of them.

**Chaos** (`make chaos`): 60 s of load. During the run:

- 12 worker containers were killed.
- The API and RabbitMQ were each killed once.
- 5% of SMTP calls failed.

| emails accepted | sent | failed | not finished | sent but missing from the SMTP sink |
|---:|---:|---:|---:|---:|
| 13,733 | 13,733 | **0** | **0** | **0** |

## fastapi-celery-imports

Python 3.13, FastAPI, Celery, RabbitMQ, Postgres. Catalog CSVs are split into 1,000-row chunks. Each chunk is validated and upserted in one transaction, and a chord finishes the import.

**Benchmark** (`make bench`): 20 files of 100,000 rows (1% invalid), uploaded back to back.

| import workers × processes | rows | valid rows stored | invalid rows reported by line | seconds | rows/s |
|---|---:|---:|---:|---:|---:|
| 1 × 4 | 2,000,000 | 1,980,000 | 20,000 | 34.5 | 57,971 |
| 2 × 4 | 2,000,000 | 1,980,000 | 20,000 | 34.1 | 58,651 |
| 4 × 4 | 2,000,000 | 1,980,000 | 20,000 | 32.8 | **60,976** |

More workers barely help here because all containers share the VM's 4 CPUs. One worker container already keeps them and Postgres busy.

**Chaos** (`make chaos`): 55 imports of 20,000 rows, one per second. During the run:

- 18 worker containers were killed.
- RabbitMQ was killed once.
- 2% of chunk attempts failed on purpose.

| rows | imports unfinished | chunks that ran more than once | chunks applied twice or lost |
|---:|---:|---:|---:|
| 1,100,000 | **0** | 42 | **0** |

## laravel-marketplace-search

Laravel 13 on Octane, Elasticsearch 9, Postgres 17, Horizon. Turkish product search over a 200,000-product catalog.

**Engine comparison** (300 queries per scenario, four facets per query):

| scenario | Elasticsearch p50 | Postgres FTS p50 |
|---|---:|---:|
| category page (facets over large sets) | **10.8 ms** | 215.6 ms |
| exact-keyword text | 35.5 ms | 9.0 ms |

Postgres is faster for exact keywords. It returns **0 results** for what users actually type: "ayakkabi" for "ayakkabı", "televizon", "smartphone". Elasticsearch returns 3,500–17,000 results for those queries.

**HTTP and reindex** (`make bench`):

| run | requests | req/s | p95 | errors |
|---|---:|---:|---:|---:|
| 32 clients, mixed search with facets | 2,506 | 83 | 498 ms | 0 |
| 50/s during a full reindex + alias swap | 2,739 | 50 | 479 ms | **0** |

**Index build:** 200,000 documents in 17.3 s (11,500 docs/s), or 27.6 s while serving search traffic. Throughput is bound by Elasticsearch CPU on the shared 4-vCPU VM.

## ads-insights-agent

TypeScript, Claude API, Postgres. Runs are queued and executed by workers that claim jobs with `FOR UPDATE SKIP LOCKED` and leases.

**Benchmark** (`make bench`) uses the scripted demo model, so it measures everything except Claude. It covers the API, the job queue, the agent loop and real SQL tool calls. 10,000 runs per batch:

| workers × concurrency | runs/s | failed | claimed twice or retried |
|---|---:|---:|---:|
| 1 × 4 | 712 | 0 | 0 |
| 2 × 4 | **859** | 0 | 0 |
| 4 × 4 | 801 | 0 | 0 |

A real Claude run takes seconds, so 1.2–1.4 ms of queue overhead per run is negligible.

This report has no numbers for answer quality or Claude latency. Measuring those needs an API key; the repository has a 12-case eval suite for that (`make eval`).
