# Backend Storage Strategy for High-Frequency Time-Series Sensor Data

**Research Ticket:** Backend Storage Strategy  
**Project:** ConnectedEdge  
**Date:** 2026-05-26  
**Status:** Research Complete

## Executive Summary

For the ConnectedEdge project, we recommend **TimescaleDB** (PostgreSQL extension) as the primary time-series storage for the MVP, with a tiered retention architecture. This choice balances write throughput, query flexibility for ML training, operational simplicity, and long-term data retention economics.

**Key Recommendation:**
- **Primary Store:** TimescaleDB with hypertables and continuous aggregates
- **Write Pattern:** Batch writes via MQTT bridge with buffering
- **Query Pattern:** Pre-computed continuous aggregates for UI, raw data exports for ML training
- **Retention:** 7 days raw → 90 days hourly aggregates → 2 years daily aggregates

## Database Options Analysis

### Option 1: TimescaleDB (PostgreSQL Extension)

**Architecture:** PostgreSQL extension that adds hypertables (automatic time partitioning), continuous aggregates (materialized views that auto-refresh), and columnar compression.

**Strengths:**
- Full SQL compatibility with PostgreSQL ecosystem
- Excellent for ML training queries (JOINs, window functions, lateral joins)
- Continuous aggregates enable pre-computed downsampling
- 10-20x compression for time-series data
- Mature tooling and community support
- Single schema serves both real-time alerting and ML training exports

**Weaknesses:**
- Lower raw write throughput than specialized TSDBs (~500K points/sec)
- Higher operational complexity than managed services
- Requires PostgreSQL expertise

**Best For:** Teams needing rich query capabilities, ML integration, and SQL joins with metadata.

### Option 2: InfluxDB (v3 with Apache Arrow/Parquet)

**Architecture:** Purpose-built TSDB using TSM trees (v1/v2) or Apache Arrow/Parquet (v3). Own query language (InfluxQL) plus SQL via FlightSQL.

**Strengths:**
- High write throughput (1M+ points/sec)
- Strong IoT ecosystem (Telegraf integration)
- Built-in retention policies
- Cloud-managed options available
- Good for high-cardinality data

**Weaknesses:**
- Limited JOIN capabilities for ML training queries
- Version fragmentation (1.x, 2.x, 3.0)
- Less SQL-compatible than TimescaleDB
- Flux query language has steep learning curve

**Best For:** Pure metrics/monitoring workloads with less need for complex analytical queries.

### Option 3: QuestDB

**Architecture:** Java-based TSDB with column-oriented storage, optimized for high-speed ingestion and SQL queries with time-series extensions.

**Strengths:**
- Highest write throughput (4M+ points/sec)
- Excellent for high-frequency data
- SQL with time-series extensions (SAMPLE BY)
- Open-source with simple deployment

**Weaknesses:**
- Smaller ecosystem and community
- Less mature tooling
- Fewer JOIN/window function capabilities
- Limited ML-specific features

**Best For:** High-frequency financial tick data or extreme ingestion workloads.

### Option 4: Traditional Databases with Extensions

**PostgreSQL (plain):** Without TimescaleDB, requires manual partitioning and lacks time-series optimizations. Not recommended for high-frequency data.

**MongoDB:** Document store with time-series collections (v5.0+). Better for schema-less data but lacks time-series query optimizations.

### Option 5: Object Stores (S3/MinIO)

**Architecture:** Store data as Parquet/CSV files in object storage, query with Athena/Presto or local processing.

**Strengths:**
- Extremely cheap for long-term retention
- Scales virtually infinitely
- Good for batch ML training

**Weaknesses:**
- Not suitable for real-time queries
- No built-in downsampling or retention policies
- Higher latency for ad-hoc queries

**Best For:** Cold storage tier in a hybrid architecture.

## Recommendation: TimescaleDB

For ConnectedEdge's specific needs, TimescaleDB is the optimal choice because:

1. **ML Training Integration:** The ML training pipeline needs to query historical data with complex filters, JOINs with device metadata, and window functions. TimescaleDB's full PostgreSQL compatibility enables this without ETL steps.

2. **Single Schema for Multiple Consumers:** Both the UI (real-time dashboards) and ML training (historical analysis) can query the same schema. Continuous aggregates serve the UI, while raw data exports feed ML training.

3. **Operational Simplicity:** For an MVP, managing one PostgreSQL instance with TimescaleDB is simpler than managing multiple specialized databases.

4. **Cost-Effective Retention:** Columnar compression (10-20x) makes long-term retention affordable without requiring a separate object storage tier for the MVP.

## Architecture Design

### Write Patterns

**Challenge:** Handling bursty, high-frequency writes from multiple edge devices.

**Solution:**
1. **MQTT Bridge with Buffering:** The MQTT broker (Mosquitto/EMQX) receives sensor data from devices. A bridge service buffers and batches writes to TimescaleDB.
2. **Batch Size:** 10,000 lines or 10MB per batch (whichever comes first)
3. **Write Ordering:** Sort by timestamp (ascending) and tags (lexicographic) before writing
4. **Compression:** Use gzip compression for write requests (up to 5x speed improvement)
5. **Timestamp Precision:** Use millisecond precision (sufficient for sensor data, better performance than nanoseconds)

```
Edge Devices → MQTT Broker → Bridge Service (buffer/batch) → TimescaleDB
```

### Query Patterns

**UI Dashboard Queries:**
- Range queries with time filtering
- Aggregations (mean, min, max, count) over time windows
- Latest value per device
- Use continuous aggregates for sub-second response times

```sql
-- Real-time dashboard query using continuous aggregate
SELECT bucket, device_id, AVG(temperature) as avg_temp
FROM sensor_hourly_stats
WHERE bucket >= NOW() - INTERVAL '24 hours'
GROUP BY bucket, device_id
ORDER BY bucket DESC;
```

**ML Training Queries:**
- Export historical data for model training
- Feature extraction with window functions
- JOIN with device metadata for context
- Long time-range queries

```sql
-- ML training data export
SELECT sr.time, sr.device_id, sr.temperature, sr.humidity,
       d.device_type, d.location
FROM sensor_readings sr
JOIN devices d ON sr.device_id = d.id
WHERE sr.time BETWEEN '2026-01-01' AND '2026-03-01'
  AND d.device_type = 'temperature_sensor'
ORDER BY sr.device_id, sr.time;
```

### Retention & Downsampling Strategy

**Tiered Retention:**

| Tier | Retention | Resolution | Storage |
|------|-----------|------------|---------|
| Raw Data | 7 days | Full (e.g., 1Hz) | Rowstore (fast writes) |
| Hourly Aggregates | 90 days | 1 hour | Continuous aggregates |
| Daily Aggregates | 2 years | 1 day | Continuous aggregates |

**Implementation:**

```sql
-- Create hypertable with automatic partitioning
CREATE TABLE sensor_readings (
    time TIMESTAMPTZ NOT NULL,
    device_id TEXT NOT NULL,
    metric TEXT NOT NULL,
    value DOUBLE PRECISION NOT NULL,
    unit TEXT
);
SELECT create_hypertable('sensor_readings', 'time', chunk_time_interval => INTERVAL '1 day');

-- Enable compression for data older than 7 days
ALTER TABLE sensor_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_id, metric',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('sensor_readings', INTERVAL '7 days');

-- Create continuous aggregate for hourly stats
CREATE MATERIALIZED VIEW sensor_hourly_stats
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    device_id,
    metric,
    AVG(value) AS avg_value,
    MIN(value) AS min_value,
    MAX(value) AS max_value,
    COUNT(*) AS sample_count
FROM sensor_readings
GROUP BY bucket, device_id, metric;

-- Auto-refresh hourly aggregates every hour
SELECT add_continuous_aggregate_policy('sensor_hourly_stats',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- Retention policies
SELECT add_retention_policy('sensor_readings', INTERVAL '7 days');
SELECT add_retention_policy('sensor_hourly_stats', INTERVAL '90 days');
```

## Tradeoffs

### TimescaleDB vs. InfluxDB

| Factor | TimescaleDB | InfluxDB |
|--------|-------------|----------|
| Write Throughput | ~500K pts/s | ~1M+ pts/s |
| Query Flexibility | Full SQL + window functions | Limited JOINs |
| ML Integration | Excellent (psycopg2, pandas) | Good (Python client) |
| Operational Complexity | Higher (PostgreSQL) | Lower (managed options) |
| Cost | Open-source + PostgreSQL | Open-source or paid cloud |
| Ecosystem | Mature PostgreSQL ecosystem | Growing IoT ecosystem |

### TimescaleDB vs. QuestDB

| Factor | TimescaleDB | QuestDB |
|--------|-------------|---------|
| Write Throughput | ~500K pts/s | ~4M+ pts/s |
| SQL Compatibility | Full PostgreSQL | SQL with extensions |
| Community Size | Large | Small |
| ML Integration | Excellent | Limited |
| Operational Maturity | High | Growing |

## Future Considerations

If the project scales beyond MVP requirements:

1. **Hybrid Architecture:** Add object storage (S3/MinIO) for cold storage beyond 2 years
2. **Read Replicas:** For high query load, add read replicas for dashboard queries
3. **Sharding:** For extreme scale, consider TimescaleDB's distributed hypertables
4. **Specialized Stores:** If vibration waveform data is needed, add a purpose-built store (e.g., ReductStore) alongside TimescaleDB

## References

1. TimescaleDB Documentation - IoT Recipes: https://docs.timescale.com/use-timescale/latest/cookbook/
2. TimescaleDB Predictive Maintenance Architecture: https://www.tigerdata.com/learn/predictive-maintenance-database
3. InfluxDB Write Optimization: https://docs.influxdata.com/influxdb3/cloud-serverless/write-data/best-practices/optimize-writes/
4. InfluxDB High Velocity Data: https://www.influxdata.com/blog/optimizing-influxdb-performance-for-high-velocity-data/
5. QuestDB vs InfluxDB Comparison: https://questdb.com/blog/replace-influxdb/
6. Time-Series Database Comparison (MinhVo): https://minhvo.is-a.dev/blogs/database-time-series-influxdb-timescaledb-and-questdb
7. TPCTC 2024 - Benchmarking Time Series Databases: https://dlnext.acm.org/doi/10.1007/978-3-031-93858-0_4
