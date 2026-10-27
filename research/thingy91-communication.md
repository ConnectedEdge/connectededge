# Thingy:91 Communication Path Research

**Date:** 2026-04-24
**Status:** Research findings for ConnectedEdge project
**Ticket:** Thingy:91 Communication Path

## Executive Summary

The Nordic Thingy:91 integrates two processors: the **nRF9160 SiP** (cellular LTE-M/NB-IoT/GNSS) and the **nRF52840 SoC** (BLE/NFC). Both can operate concurrently. For the ConnectedEdge project, a **hybrid approach** is recommended:

- **Cellular (nRF9160)**: Primary data path for telemetry and ML inference results via MQTT over LTE-M/NB-IoT
- **BLE (nRF52840)**: Local configuration, provisioning, and terminal access via shell-over-BLE (NUS)

This aligns with the project's existing architecture which already includes a "direct NB-IoT path for cellular devices" and uses MQTT as the communication protocol.

## 1. Thingy:91 Hardware Architecture

The Thingy:91 is a battery-operated prototyping platform with two distinct processors:

### nRF9160 SiP (Cellular Processor)
- **Modem**: Multimode LTE-M/NB-IoT with +23 dBm output power
- **Bands**: 700-960 MHz + 1710-2200 MHz (global certification)
- **GNSS**: Integrated GPS/QZSS
- **Processor**: 64 MHz Arm Cortex-M33, 1 MB flash, 256 kB RAM
- **Security**: Arm CryptoCell 310, TrustZone
- **Power saving**: eDRX, PSM modes

### nRF52840 SoC (BLE Processor)
- **Wireless**: Bluetooth 5.3 LE, NFC passive tag
- **Processor**: 64 MHz Arm Cortex-M4F, 1 MB flash, 256 kB RAM
- **USB**: Built-in USB controller

### Key Architectural Fact
**LTE-M/NB-IoT can operate simultaneously with Bluetooth LE.** The two processors communicate internally, and each can run independent firmware. This enables true concurrent operation of both communication paths.

## 2. Communication Path Options

### Option A: Cellular Only (nRF9160)
- Direct MQTT over LTE-M/NB-IoT to broker
- Uses Zephyr's networking stack with modem library
- No gateway dependency
- **Pros**: Simple architecture, global coverage, direct-to-cloud
- **Cons**: Higher power consumption, requires SIM/eSIM and data plan, higher latency for local operations

### Option B: BLE Only (nRF52840 via Gateway)
- Device sends data via BLE to nearby gateway
- Gateway bridges to MQTT broker
- **Pros**: Lower power, no SIM required, good for dense deployments
- **Cons**: Requires gateway infrastructure, limited range (~100m), single point of failure

### Option C: Hybrid (Recommended)
- **Cellular for data**: MQTT over LTE-M/NB-IoT for telemetry and ML results
- **BLE for local operations**: Configuration, provisioning, terminal access
- **Pros**: Best of both worlds, aligns with existing architecture, flexible deployment
- **Cons**: More complex firmware, two radio stacks to maintain

## 3. Protocol Analysis

### MQTT over Cellular (LTE-M/NB-IoT)
Nordic provides official MQTT samples for Thingy:91 using the nRF9160:

```bash
# Build for Thingy:91 with TLS
west build -b thingy91/nrf9160/ns samples/net/mqtt -- -DOVERLAY_CONFIG=overlay-tls-nrf9160.conf
```

**Key characteristics:**
- Uses Zephyr's MQTT client library
- TLS 1.2 supported (modem TLS buffers limit incoming messages to 2 kB)
- Client ID defaults to IMEI number
- Supports QoS 0, 1, 2
- Automatic reconnection logic built into samples

**Power considerations:**
- PSM (Power Saving Mode): Device sleeps for extended periods, wakes periodically
- eDRX (extended Discontinuous Reception): Shorter sleep intervals, faster response
- Trade-off: Latency vs. battery life

### BLE Protocol Options

#### Nordic UART Service (NUS)
- Standard service for serial-like communication
- Well-documented, widely supported
- Good for terminal/shell access
- Throughput: ~50-100 kbps practical

#### Shell-over-BLE (Zephyr)
Zephyr provides shell backend over BLE NUS (merged in PR #69881):

```ini
CONFIG_SHELL=y
CONFIG_SHELL_BACKEND_UART=y
CONFIG_BT=y
CONFIG_BT_PERIPHERAL=y
CONFIG_BT_ZEPHYR_NUS=y
CONFIG_UART_BT=y
```

This enables a full interactive shell via BLE, ideal for:
- Device configuration
- Debugging
- Terminal access from Linux (via socat or Python wrapper)

#### MCUmgr over BLE
For file transfers (model updates, logs):
```ini
CONFIG_MCUMGR=y
CONFIG_MCUMGR_GRP_FS=y
CONFIG_MCUMGR_TRANSPORT_BT=y
```

## 4. Use Case Mapping

| Use Case | Path | Protocol | Rationale |
|----------|------|----------|-----------|
| Sensor telemetry | Cellular | MQTT over LTE-M/NB-IoT | Direct-to-cloud, no gateway dependency |
| ML inference results | Cellular | MQTT over LTE-M/NB-IoT | Low latency, reliable delivery |
| Model updates | Cellular | MCUmgr over MQTT | Large file transfer, chunked |
| Device configuration | BLE | Shell-over-BLE (NUS) | Interactive, local access |
| Provisioning | BLE | Custom GATT or NUS | One-time setup, no network needed |
| Terminal/debug access | BLE | Shell-over-BLE (NUS) | Interactive debugging |
| Firmware updates | Cellular | MCUmgr over MQTT | OTA updates |

## 5. Alignment with ConnectedEdge Architecture

The hybrid approach aligns perfectly with the project's existing decisions:

1. **Gateway Tiering Model**: Thingy:91 uses the "direct NB-IoT path" for cellular devices, bypassing the gateway tiers for data. BLE operations would use the Sniffer/Proxy tiers when a gateway is nearby.

2. **MQTT Protocol**: Both cellular and gateway paths use MQTT, maintaining protocol consistency.

3. **Terminal Protocol Decision**: The project already decided on "Zephyr's built-in shell over BLE (NUS)" for terminal access, which matches this recommendation.

4. **Model Slot Architecture**: MCUmgr over MQTT for model updates aligns with the decoupled memory map approach.

## 6. Implementation Considerations

### Firmware Architecture
- **nRF9160 firmware**: MQTT client, sensor data collection, ML inference (TFLite Micro)
- **nRF52840 firmware**: BLE stack, shell service, provisioning interface
- **Inter-processor communication**: Via internal UART or shared memory

### Power Management
- Use PSM for battery-operated deployments (minutes to hours between transmissions)
- Use eDRX for near-real-time applications (seconds between transmissions)
- BLE advertising intervals should be optimized for discovery vs. power trade-off

### Security
- **Cellular**: TLS 1.2 with CA certificates, MQTT authentication
- **BLE**: BLE pairing (LE Secure Connections), optional encryption
- **SIM**: eSIM with remote provisioning for global deployment

## 7. References

1. Nordic Thingy:91 Hardware User Guide: https://infocenter.nordicsemi.com/topic/ug_thingy91/UG/thingy91/intro/frontpage.html
2. Nordic Thingy:91 Product Brief: https://www.nordicsemi.com/-/media/Software-and-other-downloads/Product-Briefs/Nordic-Thingy-91-PB-v2.1.pdf
3. nRF Connect SDK MQTT Sample: https://docs.nordicsemi.com/bundle/ncs-2.4.4/page/nrf/samples/net/mqtt/doc/description.html
4. Zephyr Shell Over BLE PR #69881: https://github.com/zephyrproject-rtos/zephyr/pull/69881
5. Nordic Secure MQTT Sensor/Actuator Sample: https://docs.nordicsemi.com/bundle/ncs-3.0.2/page/zephyr/samples/net/secure_mqtt_sensor_actuator/README.html
6. ConnectedEdge Wayfinder Map: https://github.com/ConnectedEdge/connectededge/blob/main/WAYFINDER_MAP.md

## Open Questions

1. What is the expected deployment density? (Affects BLE gateway vs. cellular-only decision)
2. What is the target battery life? (Affects PSM/eDRX configuration)
3. Will devices be deployed globally? (Affects LTE band configuration and eSIM strategy)
4. What is the expected data volume per device per day? (Affects data plan and MQTT QoS selection)
