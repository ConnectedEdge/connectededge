# ConnectedEdge Map

## Destination
A functional MVP vertical slice: `Sensor Data (nRF)` $
ightarrow$ `Tiered Gateway` $
ightarrow$ `Backend` $
ightarrow$ `ML Training` $
ightarrow$ `Model-Only Update (The Last Mile)`.

## Notes
- **Domain**: Embedded ML / IoT Platform.
- **Stack**: Zephyr RTOS, MQTT, WebSockets, TFLite Micro.
- **Key Goal**: Solve the "Last Mile" (intuitive deployment of models back to edge).

## Decisions so far
- [The Gateway Tiering Model](https://github.com/ConnectedEdge/connectededge/issues/2): Three tiers (Sniffer, Proxy, Orchestrator) based on software capabilities, with a direct NB-IoT path for cellular devices.
- [The 'Model Slot' Architecture](https://github.com/ConnectedEdge/connectededge/issues/3): Decoupled memory map using a dedicated Model Slot and Config Slot for safe, hot-swappable model updates.
- [The ML Workflow Wrapper](https://github.com/ConnectedEdge/connectededge/issues/5): No-code pipeline using a Generic Model Runner and Semantic Tensor Mapping for device-agnostic deployment.
- [The Terminal Protocol](tickets/terminal-protocol.md): Zephyr's built-in shell over BLE (NUS) is sufficient; use MCUmgr SMP File Management over MQTT for file transfers.
- [Backend Storage Strategy](tickets/backend-storage-strategy.md): TimescaleDB with hypertables and continuous aggregates; tiered retention (7 days raw → 90 days hourly → 2 years daily).
- [Front-End Labeling UI](tickets/frontend-labeling-ui.md): Brush selection for time-range labeling, multi-class labels, versioned dataset snapshots, overlay + confusion matrix for review.
- [Device-Specific HALs](tickets/device-specific-hals.md): Devicetree overlays for board-specific config, Sensing Subsystem for enumeration, MQTT capability reporting at startup.
- [Device-Specific HALs](research/device-specific-hals.md): Use Zephyr devicetree overlays for board-specific sensor configuration with logical-to-physical mapping via aliases; Sensing Subsystem for runtime enumeration; MQTT capability reporting at device startup for dynamic backend discovery.

## Not yet specified

<!-- Fog of war: decisions that are suspected but not yet sharp enough to ticket -->

## Out of scope
- Commercial scaling/multi-tenancy (MVP focus).
- Custom ML math engines (using existing TFLite Micro).
