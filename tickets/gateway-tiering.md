# Ticket: The Gateway Tiering Model
**Label**: `wayfinder:grilling`

## Question
We have identified three levels of gateways: **Sniffer**, **Proxy**, and **Orchestrator**. We need to define the precise functional boundary and hardware requirements for each.

1. **Sniffer**: Does it just listen to BLE Beacons (like Ruuvitag) and forward to MQTT, or does it handle basic aggregation?
2. **Proxy**: Does it handle bi-directional traffic only, or does it maintain a "Device Twin" state?
3. **Orchestrator**: What specifically makes it an orchestrator? (e.g., Local OTA storage, scheduling, local model verification).

The goal is to ensure the Backend knows which "tier" it is talking to so it can request the right operations.
