# Ticket: The Terminal Protocol
**Label**: `wayfinder:research`
**Status**: Resolved

## Question
We want a "full terminal application" where a Linux device (e.g., Raspberry Pi) interacts with an nRF board.

We need to research:
1. **Existing Protocols**: Is Zephyr's built-in shell over BLE/UART sufficient, or do we need a custom wrapper?
2. **Interface**: How do we map the Linux terminal (SSH/etc) to the device's serial/BLE interface?
3. **Capabilities**: Can we support file transfers (for logs/models) through this terminal, or should that be a separate MQTT channel?

## Resolution
Zephyr's built-in shell over BLE (NUS) is sufficient — no custom wrapper protocol needed. For the MVP, use direct serial (USB/UART) with `picocom` or `screen` on the Raspberry Pi. For file transfers (logs, models), use MCUmgr SMP File Management (Group 8) over MQTT as a separate channel, not through the terminal shell. The shell is line-oriented and not designed for bulk data transfer; MCUmgr provides chunked, stateless, reliable file operations with offset tracking.

**Research artifact**: `/research/terminal-protocol.md`
