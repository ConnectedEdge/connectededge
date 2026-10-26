# Ticket: The Terminal Protocol
**Label**: `wayfinder:research`

## Question
We want a "full terminal application" where a Linux device (e.g., Raspberry Pi) interacts with an nRF board.

We need to research:
1. **Existing Protocols**: Is Zephyr's built-in shell over BLE/UART sufficient, or do we need a custom wrapper?
2. **Interface**: How do we map the Linux terminal (SSH/etc) to the device's serial/BLE interface?
3. **Capabilities**: Can we support file transfers (for logs/models) through this terminal, or should that be a separate MQTT channel?
