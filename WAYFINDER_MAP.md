# ConnectedEdge Map

## Destination
A functional MVP vertical slice: `Sensor Data (nRF)` $ightarrow$ `Tiered Gateway` $ightarrow$ `Backend` $ightarrow$ `ML Training` $ightarrow$ `Model-Only Update (The Last Mile)`.

## Notes
- **Domain**: Embedded ML / IoT Platform.
- **Stack**: Zephyr RTOS, MQTT, WebSockets, TFLite Micro.
- **Key Goal**: Solve the "Last Mile" (intuitive deployment of models back to edge).

## Decisions so far
- [The Gateway Tiering Model](https://github.com/ConnectedEdge/connectededge/issues/2): Three tiers (Sniffer, Proxy, Orchestrator) based on software capabilities, with a direct NB-IoT path for cellular devices.
- [The 'Model Slot' Architecture](https://github.com/ConnectedEdge/connectededge/issues/3): Decoupled memory map using a dedicated Model Slot and Config Slot for safe, hot-swappable model updates.
- [The ML Workflow Wrapper](https://github.com/ConnectedEdge/connectededge/issues/5): No-code pipeline using a Generic Model Runner and Semantic Tensor Mapping for device-agnostic deployment.

## Not yet specified
- **Backend Storage Strategy**: Efficiency of high-frequency time-series data storage.
- **Front-End Labeling UI**: UX for time-series data labeling.
- **Device-Specific HALs**: Sensor mapping for various nRF boards.

## Out of scope
- Commercial scaling/multi-tenancy (MVP focus).
- Custom ML math engines (using existing TFLite Micro).
