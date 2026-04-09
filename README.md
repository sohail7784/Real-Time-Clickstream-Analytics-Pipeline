# Real-Time-Clickstream-Analytics-Pipeline
# Real-Time Clickstream Analytics Pipeline

> **Python → Kafka → ClickHouse → Grafana** — streams ~100 e-commerce events/sec through a Medallion Architecture (Bronze/Silver/Gold) and surfaces live business metrics on a dashboard. Fully Dockerized.

![Python](https://img.shields.io/badge/Python-3.9+-blue) ![Kafka](https://img.shields.io/badge/Kafka-7.5-black) ![ClickHouse](https://img.shields.io/badge/ClickHouse-latest-yellow) ![Grafana](https://img.shields.io/badge/Grafana-latest-orange) ![Docker](https://img.shields.io/badge/Docker-Compose-blue)

---

## What it does

A Python producer generates realistic e-commerce events (views, add-to-cart, checkouts) and publishes them to Kafka. ClickHouse consumes this stream in real-time, transforms it through materialized views, and pre-aggregates it into hourly metrics. Grafana queries the Gold layer and renders live charts — all sub-second.

**Questions this pipeline answers in real-time:**
- Which products generated the most revenue in the last hour?
- How many unique users checked out in the last 24 hours?
- What's the event breakdown by action type and device?

---

## Architecture

```
Extrac The ZIP Folder

Python Producer (~100 events/sec)
        │
        ▼
   ┌─────────┐
   │  Kafka  │  web_events topic · 6 partitions
   └────┬────┘
        │
        ▼
┌──────────────────────────────────┐
│           ClickHouse             │
│                                  │
│  BRONZE  kafka_web_events_queue  │  ← Kafka Engine, raw JSON
│              │ Materialized View │
│  SILVER  web_events_final        │  ← ReplacingMergeTree, deduped
│              │ Materialized View │
│  GOLD    web_events_hourly_*     │  ← AggregatingMergeTree, pre-calced
└──────────────┬───────────────────┘
               ▼
          Grafana :3000
```

---

## Quick Start

**Prerequisites:** Docker Desktop, Python 3.9+

```bash
git clone https://github.com/sohail7784/clickhouse-streaming-project.git
cd clickhouse-streaming-project

# 1. Boot all containers
docker compose up -d

# 2. Create Kafka topic
docker exec clickhouse-streaming-kafka-1 /usr/bin/kafka-topics \
  --bootstrap-server localhost:9092 --create --topic web_events \
  --partitions 6 --replication-factor 1

# 3. Apply ClickHouse schemas
docker exec -i clickhouse-server clickhouse-client --multiquery < clickhouse/01-tables.sql
docker exec -i clickhouse-server clickhouse-client --multiquery < clickhouse/02-materialized-views.sql
docker exec -i clickhouse-server clickhouse-client --multiquery < clickhouse/03-aggregations.sql

# 4. Start the producer
pip install -r producers/requirements.txt
python producers/producer.py
```

You should see: `🚀 Starting Stream Generation... Press Ctrl+C to stop.`

---

## Verify Data is Flowing

```sql
-- Connect: docker exec -it clickhouse-server clickhouse-client

SELECT count(), max(event_time) FROM default.web_events_final;

SELECT product_id, round(sum(amount), 2) AS revenue
FROM default.web_events_final
GROUP BY product_id ORDER BY revenue DESC LIMIT 5;
```

---

## Grafana Dashboard

1. Open [http://localhost:3000](http://localhost:3000) → login `admin / admin`
2. **Dashboards → Import** → upload `dashboards/grafana-dashboard.json`

Live charts for hourly revenue, event counts, and top products will appear immediately.

---

## Services

| Service | URL |
|---|---|
| Grafana | http://localhost:3000 |
| AKHQ (Kafka UI) | http://localhost:8080 |
| ClickHouse HTTP | http://localhost:8123 |

---

## Teardown
docker compose down -v


**Mohammed Sohail** — [GitHub](https://github.com/sohail7784) · [Portfolio](https://mohammed-sohail.vercel.app)
