### Hi, I'm Salim 👋

Backend engineer in İstanbul (6 years). I build high-traffic platforms and marketplaces with **Laravel, Node.js/TypeScript and Go**, with queues, caches and search at their core, and I integrate LLM agents into products.

The projects below focus on the part of backend work that is hard to get right: messages that are **neither lost nor applied twice** when processes, brokers and databases fail. Each one has tests, CI and Kubernetes manifests, and a **chaos test** that kills containers with `SIGKILL` under load and then checks every message.

**To review any of them you only need Docker and `make`:** clone the repo and run `make demo`. It starts the whole stack, walks through the important behaviours (outages, retries, dead letters, idempotency, …) and checks each one, printing ✓ or failing loudly. `make test` runs the test suite. The ports do not overlap, so you can keep several projects running side by side.

| Project | What it shows, measured |
|---|---|
| [**spring-kafka-order-saga**](https://github.com/msalimcengiz/spring-kafka-order-saga) · Java 25 · Spring Boot 4 · Kafka | Order saga across 3 services: transactional outbox, inbox, compensation, DLT replay. **830–910 complete sagas/s, p95 488 ms at 500 orders/s. 15.5k orders with 28 instances and the broker killed: 0 money or stock drift.** |
| [**go-event-ingestor**](https://github.com/msalimcengiz/go-event-ingestor) · Go · Redis Streams · PostgreSQL | Batched `COPY` writes, poison-batch bisection, safe trimming, backpressure. **50.8k events/s (2.8× direct inserts, 99.9% fewer commits). 646k events, 14 kills: 0 lost, 0 duplicated.** |
| [**hono-mail-queue**](https://github.com/msalimcengiz/hono-mail-queue) · TypeScript · RabbitMQ | Priority classes (password resets at 9 ms p95 behind a 20k campaign), TTL retry ladder, delivery limit + DLQ replay, outbox. **13.7k emails, workers, API and broker killed: 0 lost.** |
| [**fastapi-celery-imports**](https://github.com/msalimcengiz/fastapi-celery-imports) · Python · FastAPI · Celery | Chunked CSV imports with chords, `acks_late`, quorum queues, signed webhooks. **~60k rows/s (2M rows in 33 s). 1.1M rows with workers and broker killed: 42 chunks re-ran, none applied twice.** |
| [**laravel-marketplace-search**](https://github.com/msalimcengiz/laravel-marketplace-search) · Laravel 13 · Elasticsearch 9 | Turkish search (missing Turkish characters, typos, synonyms), facets, zero-downtime reindex, Horizon, Postgres fallback behind a circuit breaker. **Category pages 20× faster than tuned Postgres FTS; 200k products indexed in 17 s; 0 errors across a live reindex.** |
| [**ads-insights-agent**](https://github.com/msalimcengiz/ads-insights-agent) · TypeScript · Claude API | Analyst agent with tenant-scoped SQL tools, human-approved writes, a Postgres job queue (leases, fencing, 429/529 retry), prompt caching, evals. **Queue: 860 runs/s, 0 claimed twice.** |

All numbers come from scripts in each repository, run on a laptop. Methods, setups and raw results: **[PERFORMANCE.md](PERFORMANCE.md)**.

**Stack:** PHP · TypeScript · Go · Java · Python · PostgreSQL · Redis · Elasticsearch · Kafka · RabbitMQ · Docker · Kubernetes · AWS

📫 [LinkedIn](https://linkedin.com/in/muhammetsalimcengiz) · msalimcengiz@gmail.com
