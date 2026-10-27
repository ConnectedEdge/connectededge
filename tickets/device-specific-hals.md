# Ticket: Device-Specific HALs
**Label**: `wayfinder:research`
**Status**: Resolved

## Question
We need to support various nRF boards with different sensor configurations. How should we design the Hardware Abstraction Layer (HAL) to map sensors across different boards?

We need to research:
1. **Zephyr's Devicetree**: How to use Zephyr's devicetree system to describe sensor configurations in a board-agnostic way.
2. **Sensor API**: How to use Zephyr's sensor API to abstract different sensor drivers behind a common interface.
3. **Configuration Strategy**: How to map logical sensor names (e.g., "temperature", "accelerometer") to physical devices across different boards.
4. **Dynamic Discovery**: Can the backend discover what sensors a device has at runtime, or must it be configured statically?

The output should be a HAL design specification with code examples.

## Resolution
Use Zephyr's devicetree overlays for board-specific sensor configuration, the Sensing Subsystem for sensor enumeration and data collection, and MQTT capability reporting at device startup. Devicetree aliases map logical sensor names (e.g., "temperature", "accelerometer") to physical devices, enabling board-agnostic application code. The device enumerates sensors at startup via `sensing_get_sensors()` and reports capabilities to the backend via MQTT. The backend then configures data collection based on available sensors.

**Research artifact**: `/research/device-specific-hals.md`
