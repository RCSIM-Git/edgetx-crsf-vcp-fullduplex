# 📡 EdgeTX — Bidirectional CRSF USB-VCP & Real-Time Telemetry Mirror for GCS

[![License: GPL v2](https://img.shields.io/badge/License-GPL_v2-blue.svg)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.html)
[![EdgeTX Version](https://img.shields.io/badge/EdgeTX-2.10-orange.svg)](https://github.com/EdgeTX/edgetx)
[![Status](https://img.shields.io/badge/Status-Functional_%26_Bench_Tested-success.svg)](#-supported-radios--binaries)

This fork of **EdgeTX** unlocks **true bidirectional communication** over a single standard USB-C cable between your radio transmitter and a PC Ground Control Station (GCS) or Telemetry Dashboard.

It merges the development work of EdgeTX PR **#7630** (*CRSF Trainer over USB-VCP*) with our custom **Full-Duplex Telemetry Mirroring Engine (`telemetrySetMirrorCb`)**, turning supported radios into ultra-low-latency bidirectional CRSF transceivers.

---

## 🎯 The Problem & Our Solution

### Before:
* **Simplex Only:** Standard USB joystick modes or baseline PR #7630 allow sending control channels from PC to the radio, but **no telemetry returns to the PC**.
* **Hardware Clutter:** Receiving telemetry on PC required external ExpressLRS/Crossfire receivers, FTDI USB dongles, or second radio modules connected via serial adapters.

### With this Fork:
* 🏎️ **Control In (PC $\rightarrow$ Radio):** Transmit up to 16 CRSF channels at 100–250 Hz directly from your sim wheel, gamepad, or autonomous GCS planner.
* 🛰️ **Telemetry Out (Radio $\rightarrow$ PC GCS):** Live telemetry from your model (Battery, Link Statistics/RSSI/LQ/SNR, GPS coordinates/speed, IMU/Attitude, Variometer) is mirrored in real time through the same USB Virtual COM Port back to your PC software.
* 🔌 **Zero Dongles:** Exactly one standard USB-C cable plugged into the radio's top USB port handles everything.

---

## 🏗️ Architecture & Data Flow

```text
[ RC Vehicle / Drone / Rover ]
      │  ▲
      │  │  Over-The-Air RF (ExpressLRS / Crossfire)
      ▼  │
[ RadioMaster Transmitter (Internal / External RF) ]
      │  ▲
      │  │  EdgeTX Mixer & Mirroring Core
      ▼  │
[ USB-VCP Full-Duplex CDC Driver (115200 - 420000 bps) ]
      │  ▲
      │  │  Standard USB-C Cable
      ▼  │
[ PC Ground Control Station (RCSIM-GCS, Mission Planner, Telemetry Logger) ]
  • Channels 1–16 (100–250 Hz)    ──> Sent to Radio
  • Live Telemetry (0x14, 0x08, 0x02, 0x86) <── Received by GCS
```

---

## 📦 Supported Radios & Binaries

| Radio Model | Target Architecture | Flash Optimization | Status | Notes |
|:---|:---|:---:|:---:|:---|
| **RadioMaster MT12** | Surface (Cars/Boats) | Yes (Stripped PXX/Heli) | ✅ Tested & Verified | Surface radio (cars/boats) |
| **RadioMaster Pocket** | Handheld (Air/Surface) | Yes (512 KB target) | ✅ Tested & Verified | 512KB Flash optimized |
| **RadioMaster Boxer** | Air / Full-size | Standard | ⚠️ Experimental | Full Flash build |
| **RadioMaster TX16S / MKII** | Color LCD Touch | Standard | ⚠️ Experimental | Color LCD target |
| **RadioMaster TX12 MKII** | Compact | Yes (512 KB target) | ⚠️ Experimental | 512KB Flash optimized |
| **RadioMaster TX12 (V1)** | Compact | Yes (512 KB target) | ⚠️ Experimental | 512KB Flash optimized |
| **RadioMaster Zorro** | Gamepad style | Yes (512 KB target) | ⚠️ Experimental | 512KB Flash optimized |

---

## ⚙️ How It Works (C++ Implementation)

The modification patches EdgeTX serial routing (`radio/src/serial.cpp`):
1. **Bidirectional Port Initialization:** Sets `params.direction = ETX_Dir_TX_RX` on the USB-VCP serial driver.
2. **Hooking the Telemetry Stream:** Registers a direct callback into `telemetrySetMirrorCb(ctx, sendByte)` when CRSF Trainer over VCP is active:
   ```cpp
   if (drv && drv->setReceiveCb) {
       crsfTrainerStart(ctx, drv);
       telemetrySetMirrorCb(ctx, sendByte); // Mirrors live CRSF telemetry out via USB-VCP
   } else {
       crsfTrainerStop();
       telemetrySetMirrorCb(nullptr, nullptr);
   }
   ```
3. **Flash-Constrained Targets (512 KB):** Implements aggressive LTO and disables unused legacy protocols (`PXX1`, `PXX2`, `HELI`, `GHOST`) so the firmware fits cleanly into 512KB STM32 chips without stripping core EdgeTX capabilities.

---

## 🛠️ Step 1: Flashing Firmware to the Radio

The safest and most reliable flashing method is using the built-in EdgeTX Bootloader:

1. **Access the SD Card:**
   - Turn on your radio, connect it to the PC via USB-C, and choose **USB Storage (SD)**.
   - Alternatively, remove the MicroSD card from the radio and insert it into a PC card reader.
2. **Copy the Firmware:**
   - Navigate to the `FIRMWARE/` folder on the MicroSD card root.
   - Copy the appropriate `.bin` file matching your radio model into the `FIRMWARE/` folder.
3. **Eject & Enter Bootloader:**
   - Safely eject the SD card / disconnect the USB cable.
   - Power off the radio.
   - Hold both horizontal trim buttons inward (towards the power button) and press the **Power** button to boot into the **EdgeTX Bootloader**.
4. **Flash:**
   - Select **Write Firmware** on the radio screen.
   - Select the copied `.bin` file and confirm by holding Enter.
   - Once the progress bar reaches 100%, select **Exit** to reboot into normal EdgeTX mode.

---

## ⚙️ Step 2: EdgeTX Menu Configuration

Configure your radio model to accept trainer data from the USB Virtual COM Port:

### 1. System Setup (`SYS`)
* **Radio Setup (`SYS` -> `Radio Setup`):**
  - Set `USB Mode` to **VCP** (or **Ask**, and select *VCP / Serial* whenever you plug in the USB cable).
* **Hardware Configuration (`SYS` -> `Hardware`):**
  - Scroll to the `USB-VCP` (or `VCP`) port setting and change it to **CRSF Trainer**.

### 2. Model Setup (`MDL`)
* **Trainer Mode (`MDL` -> `Model Setup`):**
  - Scroll down to the **Trainer** section:
    - `Mode`: **CRSF**
    - `Channels`: **CH1 - CH16**
* **Trainer Function Assignment (`MDL` -> `Special Functions`):**
  - Add a new Special Function:
    - `Switch`: `ON` (or assign a toggle switch such as `SA` / `SB` to activate trainer control)
    - `Action`: **Trainer**
    - `Value`: **Sticks** (or **Axis**)
    - `Enable`: Checked (`ON`)

---

## 🧪 Step 3: Verification & Diagnostics

To verify bidirectional communication with standalone python scripts:

```bash
python crsf_vcp_duplex_test.py --port COM5 --baud 115200
```

**Expected terminal output:**
```text
======================================================================
  CRSF USB-VCP Full-Duplex Test
======================================================================
Opened COM5 (115200 bps).
[TX] 100 Hz Control loop active (Sending 16 RC channels)...
   <-- [TELEMETRY] LINK STATS: RSSI: -42 dBm | LQ: 100% | SNR: 12 dB
   <-- [TELEMETRY] BATTERY: 8.35 V | 1.2 A | 450 mAh
   <-- [TELEMETRY] GPS: 52.2297°N, 21.0122°E | Speed: 24.1 km/h | Sats: 16
```

---

## 🏎️ Step 4: Integration with RCSIM-GCS

1. Launch **RCSIM-GCS** on your PC.
2. Go to the **Cockpit / Connection** panel:
   - Select your radio's Virtual COM Port (e.g., `COM5` - STMicroelectronics Virtual COM Port).
   - Baudrate: `115200 bps`.
   - Protocol: `CRSF Direct (USB-VCP)`.
3. Map your sim-racing steering wheel, pedals (throttle/brake), and handbrake to the desired channels in the **Input Configuration** tab.
4. Armed driving with real-time FFB telemetry feedback is now active!

---

## 🛠️ Building From Source

See [`BUILD_INSTRUCTIONS.md`](BUILD_INSTRUCTIONS.md) for complete CMake flags, cross-compiler requirements (`arm-none-eabi-gcc`), and reproducible build commands for all radio targets.

---

## 📜 Upstream Attribution & License

* **Base Project:** [EdgeTX](https://github.com/EdgeTX/edgetx)
* **Base Pull Request:** PR [#7630](https://github.com/EdgeTX/edgetx/pull/7630) by BelixRogner (`feat/crsf-trainer-over-usb-vcp`, commit `e5784ee5`)
* **Complete Corresponding Source:** Published under [RCSIM-Git/edgetx-crsf-vcp-fullduplex](https://github.com/RCSIM-Git/edgetx-crsf-vcp-fullduplex).
* **License:** Distributed under the [GNU General Public License v2.0 (GPL-2.0)](https://www.gnu.org/licenses/old-licenses/gpl-2.0.html).
* **Disclaimer:** Unofficial community builds developed for the RCSIM project. Distributed without warranty of any kind.

