# Device-Specific HALs: Sensor Abstraction for ConnectedEdge

**Research Date:** July 28, 2025
**Status:** Complete

## Executive Summary

Zephyr provides a mature devicetree-based hardware abstraction that maps perfectly to ConnectedEdge's requirement of supporting multiple nRF boards with different sensor configurations. The recommended architecture uses **devicetree overlays for board-specific configuration**, the **Sensing Subsystem for sensor enumeration and data collection**, and **MQTT capability reporting at device startup** to inform the backend of available sensors.

The HAL design enables the same application firmware to run across all board variants, with hardware differences resolved at build time through devicetree overlays. The backend discovers device capabilities dynamically via MQTT, eliminating the need for static device configuration.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    ConnectedEdge Backend                │
│  (discovers sensors via MQTT, configures collection)    │
└────────────────────────────┬────────────────────────────┘
                             │ MQTT
┌────────────────────────────┴────────────────────────────┐
│              ConnectedEdge Edge Firmware                │
│  ┌──────────────────────────────────────────────────┐   │
│  │           Sensing Subsystem (HAL)                 │   │
│  │  - Enumerate sensors via sensing_get_sensors()    │   │
│  │  - Report capabilities to backend via MQTT        │   │
│  │  - Collect data based on backend configuration    │   │
│  └──────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────┐   │
│  │           Zephyr Sensor Drivers                   │   │
│  │  (BME280, LIS3DH, ICM42688, etc.)                │   │
│  └──────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────┐   │
│  │           Devicetree (Board-Specific)             │   │
│  │  - Defines physical sensors and their properties  │   │
│  │  - Aliases map logical names to physical devices  │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## 1. Zephyr Devicetree for Sensor Configuration

### How It Works

Zephyr's devicetree system provides a hierarchical description of hardware that is processed at build time. The system generates C macros and device instances from the devicetree, enabling board-agnostic application code.

**Key components:**
- **Board DTS files** (`BOARD.dts`): Describe the specific board's hardware
- **SoC DTSI files** (`SOC.dtsi`): Describe the SoC's peripherals and capabilities
- **Overlays** (`.overlay`): Modify or extend the base devicetree for specific configurations
- **Bindings** (`.yaml`): Define the structure of device nodes for validation and macro generation

### Building for Different Boards

```bash
# Build for nRF52840 DK with default sensors
west build -b nrf52840dk_nrf52840 connectededge-firmware

# Build for custom board with overlay
west build -b nrf52840dk_nrf52840 connectededge-firmware \
    -- -DDTC_OVERLAY_FILE=boards/nrf52840dk-nrf52840-custom.overlay

# Build for nRF5340 with different sensor configuration
west build -b nrf5340dk_nrf5340 connectededge-firmware
```

## 2. Sensor API: Low-Level vs. High-Level

Zephyr provides two sensor access APIs. For ConnectedEdge, the **Sensing Subsystem** (high-level) is recommended.

### Low-Level Sensor API

Direct access to sensor drivers. Simple but requires manual device management.

```c
#include <zephyr/drivers/sensor.h>

const struct device *const temp_dev = DEVICE_DT_GET(DT_ALIAS(temp0));

if (!device_is_ready(temp_dev)) {
    LOG_ERR("Temperature sensor not ready");
    return -ENODEV;
}

struct sensor_value temp;
sensor_sample_fetch_chan(temp_dev, SENSOR_CHAN_AMBIENT_TEMP);
sensor_channel_get(temp_dev, SENSOR_CHAN_AMBIENT_TEMP, &temp);

LOG_INF("Temperature: %d.%06d C", temp.val1, temp.val2);
```

**Limitations:**
- No built-in enumeration of available sensors
- Manual device discovery via devicetree macros
- No arbitration when multiple components need the same sensor
- No built-in sampling rate management

### High-Level Sensing Subsystem (Recommended)

Provides enumeration, arbitration, scheduling, and multiple client support.

```c
#include <zephyr/sensing/sensing.h>

// Enumerate all sensors
int num_sensors;
const struct sensing_sensor_info **sensors;
sensing_get_sensors(&num_sensors, &sensors);

LOG_INF("Found %d sensors", num_sensors);

for (int i = 0; i < num_sensors; i++) {
    LOG_INF("Sensor %d: type=%d, name=%s",
            i, sensors[i]->type, sensors[i]->name);
}
```

**Advantages for ConnectedEdge:**
- Dynamic enumeration of available sensors
- Multiple clients can open the same sensor (firmware + backend requests)
- Built-in interval arbitration
- HID-standard sensor type definitions
- Decoupled from transport (works with MQTT, BLE, etc.)

## 3. Configuration Strategy: Logical-to-Physical Mapping

### Devicetree Aliases

The key to board-agnostic code is using **devicetree aliases** to map logical sensor names to physical devices. Each board defines its own aliases in its devicetree or overlay.

**Example: nRF52840 DK with BME280 and LIS3DH**

```dts
// boards/nrf52840dk-nrf52840.overlay
/ {
    aliases {
        temperature = &bme280;
        humidity = &bme280;
        pressure = &bme280;
        accelerometer = &lis3dh;
    };
};

&i2c0 {
    bme280: bme280@76 {
        compatible = "bosch,bme280";
        reg = <0x76>;
        status = "okay";
    };

    lis3dh: lis3dh@19 {
        compatible = "st,lis3dh";
        reg = <0x19>;
        status = "okay";
    };
};
```

**Example: Custom board with ICM42688 and SHT31**

```dts
// boards/custom-board.overlay
/ {
    aliases {
        temperature = &sht31;
        humidity = &sht31;
        accelerometer = &icm42688;
        gyroscope = &icm42688;
    };
};

&i2c1 {
    sht31: sht31@44 {
        compatible = "sensirion,sht31";
        reg = <0x44>;
        status = "okay";
    };

    icm42688: icm42688@6f {
        compatible = "invensense,icm42688";
        reg = <0x6f>;
        status = "okay";
    };
};
```

### Application Code (Board-Agnostic)

```c
#include <zephyr/sensing/sensing.h>
#include <zephyr/sensing/sensing_sensor_types.h>

// Application never knows which physical sensor is being used
// It only requests by logical type

void init_sensors(void) {
    int num_sensors;
    const struct sensing_sensor_info **sensors;
    
    sensing_get_sensors(&num_sensors, &sensors);
    
    for (int i = 0; i < num_sensors; i++) {
        switch (sensors[i]->type) {
            case SENSING_SENSOR_TYPE_MOTION_ACCELEROMETER_3D:
                LOG_INF("Accelerometer available: %s", sensors[i]->name);
                break;
            case SENSING_SENSOR_TYPE_LIGHT_AMBIENTLIGHT:
                LOG_INF("Light sensor available: %s", sensors[i]->name);
                break;
            // ... other types
        }
    }
}
```

## 4. Dynamic Discovery: Runtime vs. Static

### Finding: Build-Time Enumeration, Runtime Reporting

Zephyr's devicetree is processed at **build time**, not runtime. However, the Sensing Subsystem provides `sensing_get_sensors()` which enumerates all configured sensors at runtime. This is effectively **dynamic discovery within the constraints of the build configuration**.

**Key insight for ConnectedEdge:** The device reports its capabilities to the backend via MQTT at startup. The backend then knows exactly what sensors are available and can configure data collection accordingly.

### Capability Reporting via MQTT

```c
#include <zephyr/sensing/sensing.h>
#include "mqtt_client.h"

void report_capabilities(void) {
    int num_sensors;
    const struct sensing_sensor_info **sensors;
    
    sensing_get_sensors(&num_sensors, &sensors);
    
    char topic[64];
    char payload[512];
    
    snprintf(topic, sizeof(topic), "connectededge/%s/capabilities", device_id);
    
    // Build JSON payload with sensor information
    snprintf(payload, sizeof(payload), "{\"sensors\": [");
    
    for (int i = 0; i < num_sensors; i++) {
        if (i > 0) strcat(payload, ", ");
        
        char sensor_json[128];
        snprintf(sensor_json, sizeof(sensor_json),
                "{\"type\": \"%s\", \"name\": \"%s\", \"index\": %d}",
                sensing_type_to_string(sensors[i]->type),
                sensors[i]->name,
                sensors[i]->index);
        
        strcat(payload, sensor_json);
    }
    
    strcat(payload, "] }");
    
    mqtt_publish(topic, payload, 1);
}

const char *sensing_type_to_string(int type) {
    switch (type) {
        case SENSING_SENSOR_TYPE_MOTION_ACCELEROMETER_3D:
            return "accelerometer";
        case SENSING_SENSOR_TYPE_MOTION_GYROMETER_3D:
            return "gyroscope";
        case SENSING_SENSOR_TYPE_LIGHT_AMBIENTLIGHT:
            return "light";
        default:
            return "unknown";
    }
}
```

### Backend Configuration Flow

```
Device boots → Enumerates sensors → Reports capabilities via MQTT
     ↓
Backend receives capabilities → Stores in device registry
     ↓
User selects sensors to monitor in UI → Backend sends config to device
     ↓
Device receives config → Opens requested sensors → Begins collection
```

## 5. HAL Design Specification

### Layer 1: Devicetree (Hardware Description)

Each board variant has a devicetree overlay that:
1. Defines all physical sensors and their bus addresses
2. Creates aliases mapping logical names to physical devices
3. Sets sensor-specific properties (sample rate, scale, etc.)

**Standard alias naming convention:**
- `temperature` - Ambient temperature sensor
- `humidity` - Humidity sensor
- `pressure` - Barometric pressure sensor
- `accelerometer` - 3-axis accelerometer
- `gyroscope` - 3-axis gyroscope
- `magnetometer` - 3-axis magnetometer
- `light` - Ambient light sensor
- `proximity` - Proximity sensor

### Layer 2: Sensing Subsystem (Sensor Abstraction)

The firmware uses the Sensing Subsystem to:
1. Enumerate available sensors at startup
2. Open sensors by type/index
3. Configure sampling intervals
4. Receive data via callbacks

```c
#include <zephyr/sensing/sensing.h>

static void sensor_data_callback(sensing_sensor_handle_t handle,
                                 const void *buf, void *context) {
    const struct sensing_sensor_info *info = sensing_get_sensor_info(handle);
    
    // Process sensor data based on type
    switch (info->type) {
        case SENSING_SENSOR_TYPE_MOTION_ACCELEROMETER_3D:
            handle_accelerometer_data(buf);
            break;
        // ... other types
    }
}

static struct sensing_callback_list callbacks = {
    .data_event = sensor_data_callback,
    .context = NULL,
};

void open_sensor_by_type(int type, int index, uint32_t interval_us) {
    int num_sensors;
    const struct sensing_sensor_info **sensors;
    
    sensing_get_sensors(&num_sensors, &sensors);
    
    for (int i = 0; i < num_sensors; i++) {
        if (sensors[i]->type == type && sensors[i]->index == index) {
            sensing_sensor_handle_t handle;
            sensing_open_sensor(sensors[i], &callbacks, &handle);
            
            struct sensing_sensor_config config = {
                .attribute = SENSING_SENSOR_ATTRIBUTE_INTERVAL,
                .interval = interval_us,
            };
            sensing_set_config(handle, &config, 1);
            
            return;
        }
    }
}
```

### Layer 3: MQTT Integration (Backend Communication)

```c
// Report sensor data to backend
void publish_sensor_data(const char *sensor_type, const void *data, size_t len) {
    char topic[128];
    snprintf(topic, sizeof(topic), "connectededge/%s/data/%s",
             device_id, sensor_type);
    
    mqtt_publish(topic, data, len);
}

// Handle backend configuration
void handle_backend_config(const char *payload) {
    // Parse JSON config
    // For each requested sensor:
    //   open_sensor_by_type(type, index, interval)
}
```

## 6. Supported Sensors and Boards

### Common nRF Board Sensor Configurations

| Board | Temperature | Humidity | Pressure | Accelerometer | Gyroscope |
|-------|-------------|----------|----------|---------------|-----------|
| nRF52840 DK | Internal (NCT75) | - | - | LIS3DH | - |
| nRF5340 DK | Internal (NCT75) | - | - | ICM42688 | ICM42688 |
| Custom IoT Board | BME280 | BME280 | BME280 | LIS3DH | - |
| Industrial Sensor | SHT31 | SHT31 | BMP280 | ADXL355 | - |

### Sensor Driver Availability

Zephyr has drivers for 100+ sensors. Key drivers for ConnectedEdge:

- **Environmental:** BME280, SHT31, BMP280, HTU21D
- **Motion:** LIS3DH, ICM42688, ADXL355, MPU6050
- **Light:** TSL2591, BH1750
- **Gas:** SGP30, CCS811

## 7. Implementation Roadmap

### Phase 1: Basic HAL (Week 1-2)
- [ ] Implement devicetree overlays for nRF52840 DK and nRF5340 DK
- [ ] Implement sensor enumeration via Sensing Subsystem
- [ ] Implement MQTT capability reporting
- [ ] Test with BME280 and LIS3DH on nRF52840 DK

### Phase 2: Backend Integration (Week 3-4)
- [ ] Backend receives and stores device capabilities
- [ ] Backend sends sensor configuration to devices
- [ ] Device opens/closes sensors based on backend config
- [ ] End-to-end test: sensor data flows from device to backend

### Phase 3: Additional Boards (Week 5-6)
- [ ] Create devicetree overlays for additional board variants
- [ ] Test with different sensor combinations
- [ ] Document board-specific build instructions

## 8. References

### Zephyr Documentation
- [Sensors API](https://docs.zephyrproject.org/latest/hardware/peripherals/sensor/index.html)
- [Sensing Subsystem](https://docs.zephyrproject.org/latest/services/sensing/index.html)
- [Devicetree HOWTOs](https://docs.zephyrproject.org/latest/build/dts/howtos.html)
- [Devicetree Input/Output Files](https://docs.zephyrproject.org/latest/build/dts/intro-input-output.html)

### Zephyr API References
- [sensor.h](https://docs.zephyrproject.org/latest/doxygen/html/sensor_8h.html)
- [sensing.h](https://docs.zephyrproject.org/apidoc/latest/group__sensing__api.html)

### Zephyr Samples
- [BME280 Sensor Sample](https://docs.zephyrproject.org/latest/samples/sensor/bme280/README.html)
- [Sensing Subsystem Sample](https://github.com/zephyrproject-rtos/zephyr/tree/main/samples/subsys/sensing)

### Related GitHub Issues
- [Sensor Subsystem: client facing API](https://github.com/zephyrproject-rtos/zephyr/issues/48149)
- [Sensing Subsystem Upstream PR](https://github.com/zephyrproject-rtos/zephyr/pull/55389)

## 9. Open Questions

1. **Sensor calibration:** Should calibration data be stored on the device or in the backend? Recommendation: Backend, with device reporting raw values.

2. **Multi-sensor fusion:** Should the device perform fusion (e.g., IMU + magnetometer for orientation) or send raw data to the backend? Recommendation: Raw data to backend for flexibility, with optional on-device fusion for latency-sensitive use cases.

3. **Power management:** Should sensors be powered down when not in use? Recommendation: Yes, use Sensing Subsystem's client arbitration to automatically power down sensors when no clients are requesting data.

4. **Error handling:** How should sensor failures be reported to the backend? Recommendation: Publish to `connectededge/{device_id}/errors/{sensor_type}` topic with error code and timestamp.
