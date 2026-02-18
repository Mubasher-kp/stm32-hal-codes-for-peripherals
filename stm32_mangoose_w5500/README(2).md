# Smart Kannur Weather Station — STM32 Blackpill + W5500 + Mongoose

A wired Ethernet web server running on STM32F411CEU6 (Blackpill) with a W5500
Ethernet module, powered by the Mongoose Networking Library. Serves a live
weather dashboard over HTTP on a static IP — a direct port of the ESP32 WiFi
version to wired Ethernet.

![Platform](https://img.shields.io/badge/Platform-STM32F411CEU6-blue)
![Ethernet](https://img.shields.io/badge/Ethernet-W5500-green)
![Networking](https://img.shields.io/badge/Networking-Mongoose-orange)
![IDE](https://img.shields.io/badge/IDE-STM32CubeIDE-red)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

---

## Table of Contents

- [Overview](#overview)
- [Hardware Requirements](#hardware-requirements)
- [Wiring](#wiring)
- [Software Requirements](#software-requirements)
- [Project Structure](#project-structure)
- [CubeMX Configuration](#cubemx-configuration)
- [Mongoose Setup](#mongoose-setup)
- [Network Configuration](#network-configuration)
- [Building & Flashing](#building--flashing)
- [Usage](#usage)
- [API Endpoints](#api-endpoints)
- [Sensor Integration](#sensor-integration)
- [Troubleshooting](#troubleshooting)
- [Roadmap](#roadmap)

---

## Overview

This project implements a live weather monitoring web dashboard on an STM32
Blackpill microcontroller connected to a wired LAN via the W5500 SPI Ethernet
chip. The Mongoose Networking Library provides the full TCP/IP stack, HTTP
server, and JSON API — replacing the need for LwIP or any separate networking
middleware.

**Key Features:**
- Static IP assignment — accessible from anywhere on the LAN
- Live sensor dashboard (Temperature, Humidity, Pressure, Wind Speed, Wind Direction)
- REST API endpoints (`/api/latest`, `/data`) returning JSON
- CORS enabled — compatible with any central aggregator or frontend
- Fully self-contained HTML/CSS/JS served from STM32 Flash memory
- No RTOS required — bare-metal polling loop
- Drop-in replacement for the ESP32 WiFi version in wired network environments

---

## Hardware Requirements

| Component | Details |
|---|---|
| STM32F411CEU6 Blackpill | 100 MHz Cortex-M4, 512 KB Flash, 128 KB RAM |
| W5500 Ethernet Module | SPI, hardwired TCP/IP offload, 10/100 Mbps |
| RJ45 Patch Cable | Cat5e or better |
| Network Switch / Router | Any standard LAN switch |
| USB-TTL UART Adapter | For debug output (PA9 TX @ 115200 baud) |
| ST-Link V2 or Blackpill USB | For flashing via SWD or DFU |
| 3.3V Power Supply | Blackpill onboard 3.3V reg is sufficient for W5500 |

> ⚠️ **Do NOT use Bluepill (STM32F103C8T6)** for this project.
> Its 20 KB RAM is too limited for Mongoose's TCP/IP buffers + HTTP stack +
> sensor data. Use the Blackpill (STM32F411CEU6) with 128 KB RAM.

---

## Wiring

| W5500 Pin | STM32F411 Blackpill Pin | Notes |
|---|---|---|
| SCS (CS) | PA4 | GPIO Output, active LOW |
| SCLK | PA5 | SPI1 SCK |
| MISO | PA6 | SPI1 MISO |
| MOSI | PA7 | SPI1 MOSI |
| RST | PA3 | GPIO Output, active LOW |
| INT | PA2 (optional) | Interrupt — not used in polling mode |
| 3.3V | 3.3V | |
| GND | GND | |

> Connect the W5500 RJ45 port to any port on your LAN switch with a patch cable.

---

## Software Requirements

- [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html) v1.14+
- STM32CubeMX (bundled with CubeIDE)
- [Mongoose Networking Library](https://github.com/cesanta/mongoose) — single file (`mongoose.c` + `mongoose.h`)
- STM32F4 HAL Drivers (auto-included by CubeMX)
- Serial terminal (optional): `minicom`, `PuTTY`, or Arduino Serial Monitor @ 115200

---

## Project Structure

```
SmartKannur-STM32-W5500/
├── Core/
│   ├── Inc/
│   │   ├── main.h
│   │   ├── mongoose.h          ← Mongoose header (copy here)
│   │   └── mongoose_config.h   ← Your Mongoose config (create this)
│   ├── Src/
│   │   ├── main.c              ← Main application code
│   │   ├── mongoose.c          ← Mongoose source (copy here)
│   │   └── stm32f4xx_it.c
├── Drivers/
│   └── STM32F4xx_HAL_Driver/
├── SmartKannur_W5500.ioc       ← CubeMX project file
└── README.md
```

---

## CubeMX Configuration

Open `SmartKannur_W5500.ioc` in CubeMX or follow these settings manually:

### RCC
| Setting | Value |
|---|---|
| HSE | Crystal/Ceramic Resonator |
| PLL Source | HSE |
| HCLK | 100 MHz |

### SPI1 — W5500 Interface
| Setting | Value |
|---|---|
| Mode | Full-Duplex Master |
| Data Size | 8 Bits |
| First Bit | MSB First |
| CPOL | Low |
| CPHA | 1 Edge (Mode 0) |
| NSS | Software (manual CS) |
| Prescaler | /8 → ~12.5 MHz |
| Pins | PA5 (SCK), PA6 (MISO), PA7 (MOSI) |

### GPIO Outputs
| Pin | Label | Initial State |
|---|---|---|
| PA4 | W5500_CS | High |
| PA3 | W5500_RST | High |
| PA2 | W5500_INT (optional) | Input, pull-up |

### USART1 — Debug
| Setting | Value |
|---|---|
| Mode | Asynchronous |
| Baud Rate | 115200 |
| Word Length | 8 Bits |
| Stop Bits | 1 |
| Parity | None |
| TX Pin | PA9 |

### Project Manager
- Toolchain: **STM32CubeIDE**
- Generate peripheral initialization as separate .c/.h pairs: **Enabled**

---

## Mongoose Setup

### 1. Get the library
```bash
# Download mongoose.c and mongoose.h
wget https://raw.githubusercontent.com/cesanta/mongoose/master/mongoose.c
wget https://raw.githubusercontent.com/cesanta/mongoose/master/mongoose.h
```
Copy both files into `Core/Src/` and `Core/Inc/` respectively.

### 2. Create `Core/Inc/mongoose_config.h`
```c
#pragma once

#define MG_ARCH                 MG_ARCH_CUSTOM
#define MG_ENABLE_TCPIP         1
#define MG_ENABLE_DRIVER_W5500  1
#define MG_IO_SIZE              512
#define MG_ENABLE_CUSTOM_MILLIS 1
#define MG_ENABLE_LOG           1
#define MG_LOG_LEVEL            MG_LL_DEBUG
```

### 3. Add `mongoose.c` to the build
In STM32CubeIDE, right-click `Core/Src/mongoose.c` → **Add to build** (ensure
it is not excluded from compilation).

---

## Network Configuration

Edit the defines at the top of `main.c`:

```c
#define DEVICE_ID   "STM32_BLK_01"
#define STATIC_IP   MG_IPV4(172, 16, 111, 30)
#define SUBNET_MASK MG_IPV4(255, 255, 248, 0)
#define GATEWAY     MG_IPV4(172, 16, 104, 1)
```

> To use **DHCP** instead, set `.ip = 0` in the `mg_tcpip_if` struct in `main.c`.
> No other changes are needed.

---

## Building & Flashing

### Build
```
Project → Build All  (Ctrl+B)
```
Expected binary size: ~150–200 KB Flash, ~30–50 KB RAM (well within Blackpill limits).

### Flash via ST-Link (SWD)
```
Run → Debug  (F11)
```
Or use **STM32CubeProgrammer** with ST-Link V2.

### Flash via USB DFU (no ST-Link needed)
1. Hold **BOOT0** button, press **NRST**, release NRST, release BOOT0
2. Run:
```bash
dfu-util -a 0 -s 0x08000000:leave -D build/SmartKannur_W5500.bin
```

---

## Usage

1. Power the Blackpill and connect the W5500 RJ45 to your LAN switch
2. Open a browser on any device on the same network
3. Navigate to:
```
http://172.16.111.30
```
4. The Smart Kannur live weather dashboard will load
5. Sensor values refresh automatically every **5 seconds**

### Debug Output (UART1 @ 115200)
Connect a USB-TTL adapter to PA9 (TX). You will see:
```
=== Smart Kannur | STM32_BLK_01 ===
Static IP: 172.16.111.30 | GW: 172.16.104.1
Web Server started on port 80
```

---

## API Endpoints

| Endpoint | Method | Response |
|---|---|---|
| `/` | GET | Full HTML dashboard |
| `/api/latest` | GET | JSON — latest sensor reading |
| `/data` | GET | JSON — same as `/api/latest` |

### Sample `/api/latest` Response
```json
{
  "device": "STM32_BLK_01",
  "temperature": 28.4,
  "humidity": 72.1,
  "pressure": 1012.3,
  "windSpeed": 12.5,
  "windDirection": 215,
  "timestamp": 3600
}
```

---

## Sensor Integration

Replace the placeholder `update_sensors()` function in `main.c` with your real
sensor reads. The function is called every 5 seconds.

### BME280 (Temperature, Humidity, Pressure) via I2C
```c
// Example using STM32 HAL I2C + your BME280 driver
void update_sensors(void) {
    BME280_ReadAll(&hi2c1, &s_temperature, &s_humidity, &s_pressure);
    // wind sensor reads via UART or ADC below
    s_windSpeed     = anemometer_get_speed();
    s_windDirection = wind_vane_get_direction();
}
```

### Anemometer / Wind Vane
- **Pulse-count anemometer** → TIM input capture on PA0 (TIM2 CH1)
- **Wind vane (analog)** → ADC1 on PA1
- **NPK / soil sensor** → UART2 (RS-485 via MAX485 transceiver)

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Can't ping `172.16.111.30` | W5500 not initializing | Check SPI wiring, verify CS is PA4 and toggling |
| Browser shows "connection refused" | Mongoose not running / wrong port | Confirm `mg_http_listen` on port 80, check UART debug log |
| Garbled UART output | Wrong baud rate | Set terminal to 115200 8N1 |
| Dashboard loads but values stuck | `mg_mgr_poll()` being blocked | Ensure no `HAL_Delay()` in the main loop — use tick-based timing only |
| Build error: `MG_ARCH not defined` | `mongoose_config.h` missing or not found | Verify include path has `Core/Inc` |
| STM32 resets repeatedly | Stack overflow | Increase stack size in linker script or reduce `MG_IO_SIZE` |
| HTML not updating in browser | Browser cache | Hard refresh with `Ctrl+Shift+R` |

---

## Roadmap

- [ ] Replace dummy sensors with real BME280 (I2C) reads
- [ ] Add anemometer pulse-count via TIM2 input capture
- [ ] Add wind vane ADC read
- [ ] WebSocket push updates (replace 5s polling)
- [ ] MQTT publish to AWS IoT Core via W5500
- [ ] Multi-device aggregation into central Node.js / Python server
- [ ] OTA firmware update over HTTP
- [ ] Data logging to SD card via SPI2

---

## Related Projects

- [Smart Kannur ESP32 Version](../ESP32_WeatherStation/) — WiFi version of this same dashboard
- [Mongoose Networking Library](https://github.com/cesanta/mongoose)
- [ControllersTech STM32 + W5500 + Mongoose Tutorial](https://controllerstech.com/stm32-ethernet-using-mongoose-part-2/)

---

## License

MIT License — see [LICENSE](LICENSE) for details.
