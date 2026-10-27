# Ticket: Backend Storage Strategy
**Label**: `wayfinder:research`

## Question
We need to store high-frequency time-series sensor data efficiently. What storage technology and architecture should we use?

We need to research:
1. **Database Options**: Time-series databases (InfluxDB, TimescaleDB, QuestDB) vs. traditional databases with time-series extensions vs. object stores.
2. **Write Patterns**: How to handle bursty, high-frequency writes from multiple devices.
3. **Query Patterns**: What queries will the ML training pipeline and UI need? (aggregations, range queries, downsampling).
4. **Retention & Downsampling**: How to manage data growth over time.

The output should be a recommendation with tradeoffs.
