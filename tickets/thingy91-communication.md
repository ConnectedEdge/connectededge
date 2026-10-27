# Ticket: Thingy:91 Communication Architecture
**Label**: `wayfinder:research`

## Question
The Nordic Thingy:91 has two communication paths: BLE (via nRF52840) and cellular (via nRF9160). We need to decide how Thingy:91 devices communicate with the ConnectedEdge platform.

We need to research and decide:
1. **Communication Path**: Should Thingy:91 use BLE commands (via a gateway) or direct cellular communication via the nRF91 chip?
2. **Hybrid Approach**: Could we use both — BLE for local/gateway communication and cellular for direct-to-cloud?
3. **Protocol**: What protocol should be used over each path? (MQTT over cellular, proprietary BLE protocol?)
4. **Use Cases**: When should each path be used? (e.g., BLE for configuration, cellular for data?)

The output should be a communication architecture decision with justification.
