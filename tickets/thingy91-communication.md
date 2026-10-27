# Ticket: Thingy:91 Communication Architecture
**Label**: `wayfinder:research`
**Issue**: https://github.com/ConnectedEdge/connectededge/issues/6
**Status**: Resolved

## Question
The Nordic Thingy:91 has two communication paths: BLE (via nRF52840) and cellular (via nRF9160). We need to decide how Thingy:91 devices communicate with the ConnectedEdge platform.

We need to research and decide:
1. **Communication Path**: Should Thingy:91 use BLE commands (via a gateway) or direct cellular communication via the nRF91 chip?
2. **Hybrid Approach**: Could we use both — BLE for local/gateway communication and cellular for direct-to-cloud?
3. **Protocol**: What protocol should be used over each path? (MQTT over cellular, proprietary BLE protocol?)
4. **Use Cases**: When should each path be used? (e.g., BLE for configuration, cellular for data?)
5. **Remote Configuration**: How should the platform send configuration changes to remote Thingy:91 devices via cellular? (MQTT topics, MCUmgr over MQTT, custom protocol?)

The output should be a communication architecture decision with justification.

## Resolution
**Hybrid approach using both communication paths**:

- **Cellular (nRF9160)**: Primary data path via MQTT over LTE-M/NB-IoT for telemetry, ML inference results, and remote configuration. MCUmgr over MQTT handles model updates and firmware updates to remote devices.
- **BLE (nRF52840)**: Local operations via shell-over-BLE (NUS) for interactive configuration, provisioning, and terminal access when near a gateway.

This aligns with the project's existing "direct NB-IoT path" architecture and the prior Terminal Protocol decision. Remote configuration over cellular is handled by MCUmgr over MQTT, enabling the platform to push configuration changes, model updates, and firmware updates to any Thingy:91 device regardless of proximity to a BLE gateway.

**Research artifact**: `/research/thingy91-communication.md`
