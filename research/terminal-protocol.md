# Terminal Protocol Research: Full Terminal Application

**Date:** 2026-04-23
**Status:** Research findings for ConnectedEdge project

## Executive Summary

Zephyr provides a comprehensive shell subsystem with multiple transport backends. For a "full terminal application" connecting a Linux device (Raspberry Pi) to an nRF board, the recommended approach is:

1. **Use Zephyr's built-in shell** with the appropriate transport backend (BLE NUS or UART)
2. **Map Linux terminal via `socat` or `screen`** to the serial/BLE interface
3. **Use MCUmgr's fs_mgmt group** for file transfers (logs, models) rather than the shell itself

The shell provides command execution, debugging, and interactive control. File transfers are better handled by the dedicated MCUmgr protocol, which can run over the same transport.

## 1. Existing Protocols: Zephyr Shell

### Zephyr Shell Architecture

Zephyr's shell is a modular command interpreter with multiple transport backends:

- **UART** (CONFIG_SHELL_BACKEND_SERIAL) - Default, serial console
- **BLE NUS** (CONFIG_SHELL_BACKEND_UART + CONFIG_BT_ZEPHYR_NUS) - Shell over Bluetooth Low Energy
- **MQTT** (CONFIG_SHELL_BACKEND_MQTT) - Shell over MQTT topics
- **Telnet** (CONFIG_SHELL_BACKEND_TELNET) - Network shell
- **SSH** (CONFIG_SHELL_BACKEND_SSH) - Secure network shell
- **RTT** (CONFIG_SHELL_BACKEND_RTT) - J-Link Real-Time Transfer
- **WebSocket** (CONFIG_SHELL_BACKEND_WEBSOCKET) - Web-based shell
- **RPMsg** (CONFIG_SHELL_BACKEND_RPMSG) - Inter-processor communication

### Shell Over BLE (NUS)

Zephyr supports shell over BLE using the Nordic UART Service (NUS). This was contributed via PR #69881 and merged into mainline Zephyr.

**Required Kconfig options:**
```ini
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y
CONFIG_SHELL=y
CONFIG_SHELL_BACKEND_UART=y
CONFIG_BT=y
CONFIG_BT_PERIPHERAL=y
CONFIG_BT_ZEPHYR_NUS=y
CONFIG_UART_BT=y
```

**Device tree configuration:**
```dts
/ {
    chosen {
        zephyr,shell-uart = &bt_nus_shell_uart;
    };
    bt_nus_shell_uart: bt_nus_shell_uart {
        compatible = "zephyr,nus-uart";
        rx-fifo-size = <1024>;
        tx-fifo-size = <1024>;
    };
};
```

**Throughput optimizations (optional):**
```ini
CONFIG_BT_RX_STACK_SIZE=2048
CONFIG_BT_L2CAP_TX_MTU=512
CONFIG_BT_BUF_ACL_RX_SIZE=502
CONFIG_BT_BUF_ACL_TX_SIZE=502
CONFIG_BT_CTLR_DATA_LENGTH_MAX=251
```

### Shell Over MQTT

Zephyr also provides a shell backend over MQTT. The shell subscribes to `<device_id>/sh/rx` for commands and publishes to `<device_id>/sh/tx` for output.

**Required Kconfig options:**
```ini
CONFIG_SHELL=y
CONFIG_SHELL_BACKEND_MQTT=y
CONFIG_NETWORKING=y
CONFIG_NET_IPV4=y
CONFIG_SHELL_MQTT_SERVER_ADDR="broker_address"
```

### Shell Command Groups

Zephyr provides extensive built-in command groups:

- `bt` - Bluetooth commands (requires CONFIG_BT_SHELL)
- `net` - Network commands (requires CONFIG_NET_SHELL)
- `kernel` - Kernel/thread management
- `device` - Device driver management
- `fs` - File system commands
- `shell` - Shell utilities

## 2. Interface: Mapping Linux Terminal to Device

### UART Interface

For UART-based shell access from a Raspberry Pi:

```bash
# Direct serial connection
screen /dev/ttyAMA0 115200

# Or using minicom
minicom -D /dev/ttyAMA0 -b 115200
```

### BLE NUS Interface

For BLE-based shell access, use `socat` with the BlueZ NUS tools:

```bash
# Install socat
apt install socat

# Connect to NUS service
# First, find the device and connect via gatttool or btool
# Then use socat to bridge the terminal to the BLE connection
```

**Alternative: Custom Python wrapper**

For a more polished experience, a Python wrapper can be built using `bleak` or `pybluez`:

```python
import asyncio
from bleak import BleakClient

async def shell_session(address):
    async with BleakClient(address) as client:
        # Connect to NUS service
        # Read/write characteristics
        # Bridge stdin/stdout
        pass

asyncio.run(shell_session("AA:BB:CC:DD:EE:FF"))
```

### SSH Interface

If the nRF board has network connectivity (WiFi/Ethernet), SSH provides the most familiar terminal experience:

```ini
CONFIG_SHELL=y
CONFIG_SHELL_BACKEND_SSH=y
CONFIG_SSH_SERVER=y
```

```bash
ssh root@nrf-board-ip
```

## 3. Capabilities: File Transfers

### Shell File System Commands

The shell provides basic file system operations via the `fs` command group:

- `fs mount` - Mount filesystem
- `fs ls` - List directory
- `fs cd` - Change directory
- `fs read` - Read file (hex/ASCII dump)
- `fs write` - Write hex data to file
- `fs rm` - Remove file
- `fs mkdir` - Create directory

However, these are limited for large file transfers (logs, ML models).

### MCUmgr File Management (fs_mgmt)

For robust file transfers, use Zephyr's MCUmgr protocol with the fs_mgmt group:

**Supported operations:**
- File download (read chunks from device)
- File upload (write chunks to device)
- File status (get file size)
- File hash/checksum (CRC32, SHA256)
- File close (cleanup open handles)

**Transport options:** MCUmgr can run over:
- Serial (UART)
- BLE (using SMP over GATT)
- Network (TCP/UDP)

**Configuration:**
```ini
CONFIG_MCUMGR=y
CONFIG_MCUMGR_GRP_FS=y
CONFIG_MCUMGR_TRANSPORT_SHELL=y  # or _BT, _SHELL, etc.
```

**Usage example (shell transport):**
```bash
# Upload file to device
mcumgr fs upload --path /data/model.tflite --file ./model.tflite

# Download file from device
mcumgr fs download --path /data/logs/app.log --file ./app.log
```

### Recommendation: Separate Channels

For the ConnectedEdge project, recommend:

1. **Shell over BLE/UART** for interactive terminal, debugging, and command execution
2. **MCUmgr over the same transport** for file transfers (logs, TFLite models)
3. **MQTT** for application-level communication and telemetry

This separation provides:
- Clean protocol boundaries
- Efficient file transfer with chunking and error handling
- Interactive debugging without file transfer overhead
- Compatibility with existing Zephyr tooling

## References

1. Zephyr Shell Documentation: https://docs.zephyrproject.org/latest/services/shell/index.html
2. Zephyr Shell Over BLE Presentation (Croxel, EOSS 2024): https://www.croxel.com/insights/presentation-eoss-shell-ble
3. Zephyr Shell Over BLE PR #69881: https://github.com/zephyrproject-rtos/zephyr/pull/69881
4. Zephyr MCUmgr File Management: https://docs.zephyrproject.org/latest/services/device_mgmt/smp_groups/smp_group_8.html
5. Zephyr MQTT Shell Backend: https://docs.zephyrproject.org/latest/services/shell/backends/mqtt.html
6. Zephyr Shell File System Sample: https://docs.zephyrproject.org/latest/samples/subsys/shell/fs/README.html
7. Nordic UART Service (NUS) Specification: https://infocenter.nordicsemi.com/topic/ug_nus/nus.html
8. Zephyr Network Shell: https://docs.zephyrproject.org/latest/services/connectivity/networking/api/net_shell.html

## Open Questions

1. What is the expected file size range for logs and ML models? This affects chunk size optimization.
2. Should the shell be available over both BLE and MQTT simultaneously, or is one preferred?
3. What security requirements exist? (BLE pairing, MQTT authentication, SSH keys)
4. Is the Raspberry Pi the only Linux device, or should the solution be device-agnostic?
