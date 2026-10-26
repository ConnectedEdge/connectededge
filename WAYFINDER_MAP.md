# ConnectedEdge Map

## Destination
A functional MVP vertical slice: `Sensor Data (nRF)` $\rightarrow$ `Tiered Gateway` $\rightarrow$ `Backend` $\rightarrow$ `ML Training` $\rightarrow$ `Model-Only Update (The Last Mile)`.

## Notes
- **Domain**: Embedded ML / IoT Platform.
- **Stack**: Zephyr RTOS, MQTT, WebSockets, TFLite Micro.
- **Key Goal**: Solve the "Last Mile" (intuitive deployment of models back to edge).

## Decisions so far
<!-- Index of closed tickets -->

## Not yet specified
- **Backend Storage Strategy**: Efficiency of high-frequency time-series data storage.
- **Front-End Labeling UI**: UX for time-series data labeling.
- **Device-Specific HALs**: Sensor mapping for various nRF boards.

## Out of scope
- Commercial scaling/multi-tenancy (MVP focus).
- Custom ML math engines (using existing TFLite Micro).
