# Bluetooth Auto Toggle (Windhawk Mod)

[![Windhawk Mod](https://img.shields.io/badge/Windhawk-Mod-0078D6.svg)](https://windhawk.net/)
[![Platform](https://img.shields.io/badge/Platform-Windows%2010%20%7C%2011-blue.svg)](https://microsoft.com/windows)
[![Version](https://img.shields.io/badge/Version-1.1.0-green.svg)](#version-history)
[![Architecture](https://img.shields.io/badge/Architecture-x64-success.svg)](#technical-details)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A [Windhawk](https://windhawk.net/) tool mod for Windows 10 and 11 that automatically cycles Bluetooth **OFF** and back **ON** (in milliseconds) whenever Windhawk boots with your system and a connected Bluetooth device is detected.

---

## 💡 The Problem It Solves

Many Windows users experience recurring Bluetooth adapter or peripheral glitches following a cold boot or system startup:
* **Headphones / Earbuds Not Auto-Connecting:** Wireless audio devices (AirPods, Sony, Bose, etc.) fail to handshake cleanly on startup until Bluetooth is manually turned off and on in the Action Center.
* **Peripheral Sleep Glitches:** Bluetooth mice, keyboards, and game controllers remain in an unresponsive sleep state when Windows boots with Fast Startup enabled.
* **Unnecessary Cycling When Idle:** Blindly toggling Bluetooth on boot when no device is paired or in use wastes time and hardware power.

**Bluetooth Auto Toggle** intelligently solves this: as soon as Windhawk starts up on boot, it monitors for any connected Bluetooth peripheral (headset, mouse, keyboard, gamepad, etc.). The moment a device is found, it waits a configurable few seconds settle time, smoothly cycles the Bluetooth radio off and back on in milliseconds, and allows all your devices to establish a fresh, pristine connection.

---

## ✨ Key Features

* 🎯 **Device-Aware Activation (New in v1.1.0):** Only triggers the off/on cycle if a connected Bluetooth device is detected (or when one connects), avoiding blind toggles when no devices are in use.
* ⏱️ **Configurable Post-Connection Delay:** Waits a configurable "within seconds" duration (default: 2 seconds) after a device is detected before executing the toggle to ensure the connection has settled.
* ⚡ **Millisecond Toggle Precision:** Full control over startup settle timing and off-to-on toggle duration in milliseconds.
* 🛡️ **Isolated Tool Mod Architecture:** Runs in an isolated `windhawk.exe` background tool process without injecting into or hooking `explorer.exe` or any critical system shell components.
* 🔄 **Cold Boot Hardware Retry:** Automatically retries radio discovery if the Windows Bluetooth stack (`bthserv`) or adapter driver is still initializing during early boot.
* 📶 **Multi-Adapter Support:** Automatically identifies and safely cycles all available Bluetooth radios on the machine.
* 🔒 **Session-Safe Execution:** Uses a named session mutex to ensure Bluetooth only cycles once per logon session, preventing repeated toggling if Windhawk is restarted.
* 🧪 **Live Testing via Settings:** Includes a dedicated setting to trigger a test toggle right from the Windhawk Settings UI without needing to restart Windows.
* 🛑 **Instant & Safe Unload:** Uses event-driven synchronization (`g_hStopEvent`) instead of blocking sleep calls, guaranteeing instant cancellation on mod reload or shutdown.

---

## 🚀 How to Use in Windhawk

### Step 1: Enable Developer Mode in Windhawk
1. Launch the **Windhawk** application on your PC.
2. Open **Settings** (gear icon) in the navigation menu.
3. Check the box for **"Developer mode"** (or open the **Developer** tab).

### Step 2: Add and Compile the Mod
1. In the **Developer** tab, click **"New mod"**.
2. Replace the editor's default template code with the full contents of [`bluetooth-auto-toggle.wh.cpp`](./bluetooth-auto-toggle.wh.cpp).
3. Click the **"Compile and inject"** (or **"Compile and test"**) button in the top-right corner.
4. Windhawk will compile the mod using Clang and execute it in a dedicated background tool process (`windhawk.exe`).

### Step 3: Configure Settings
Once compiled, switch to the mod's **Settings** tab in Windhawk to customize timings to your hardware:

| Setting | Default | Description |
| :--- | :---: | :--- |
| **Only When Device Connected** | `true` | When enabled, Bluetooth is only cycled after a connected Bluetooth device is found. If disabled, Bluetooth cycles on startup regardless of connected devices. |
| **Delay After Device Found (ms)** | `2000` | Settle duration in milliseconds to wait after a connected device is detected before executing the off-and-on cycle. |
| **Device Connection Wait Timeout (seconds)** | `60` | Maximum seconds to wait for a Bluetooth device to connect after startup (`0` for indefinite wait until a device connects). |
| **Startup Settle Delay (ms)** | `2000` | Milliseconds to wait after Windhawk starts before querying Bluetooth (gives hardware drivers time to settle on cold boot). |
| **Toggle Off-to-On Duration (ms)** | `500` | Duration the radio remains powered OFF before being turned back ON. |
| **Radio Detection Retries** | `10` | Maximum discovery attempts if the Bluetooth radio is not immediately enumerated. |
| **Radio Retry Interval (ms)** | `500` | Milliseconds between radio discovery retries. |
| **Toggle Only Once Per Logon Session** | `true` | Prevents re-cycling Bluetooth if Windhawk UI is restarted during the same Windows session. |
| **Re-toggle on Settings Save** | `false` | When enabled, saving settings in Windhawk immediately executes a test toggle. |

---

## 🛠️ Technical Architecture & Design Safeguards

This mod was engineered to eliminate common pitfalls associated with system-level radio manipulation:

1. **Dedicated Process Isolation (Windhawk Tool Mod)**
   - Utilizes Windhawk's official tool mod pattern (`@include windhawk.exe`).
   - Does not hook or inject into `explorer.exe`, eliminating any possibility of shell crashes or Taskbar instability.
2. **Asynchronous Non-Blocking Initialization**
   - Windhawk's `Wh_ModInit` returns `TRUE` immediately.
   - All async discovery, waits, and state transitions run inside a dedicated background worker thread (`CreateThread`), preventing watchdog timeout terminations.
3. **Hardware State Transition Settle Time**
   - Instantaneous 0 ms toggling can cause Bluetooth firmware to drop commands or return `AccessDenied`.
   - The configurable `toggleDelayMs` ensures the physical radio fully enters power-down before receiving the power-up command.
4. **Modern Windows Runtime (C++/WinRT) Projections**
   - Interacts directly with `Windows.Devices.Radios.Radio`, `Windows.Devices.Bluetooth.BluetoothDevice`, and `Windows.Devices.Enumeration.DeviceInformation`.
   - Hosted in a Multi-Threaded Apartment (`winrt::init_apartment(apartment_type::multi_threaded)`).
   - All API calls are protected by structured `winrt::hresult_error` handlers.

---

## 📦 Working with GitHub

### Hosting in Your Own Repository
```bash
git init
git add .
git commit -m "feat: release v1.1.0 with device-aware toggle and millisecond control"
git branch -M main
git remote add origin https://github.com/<your-username>/bluetooth-automatic-toggle.git
git push -u origin main
```

### Contributing to the Official Windhawk Catalog
To make this mod available for one-click installation to all Windhawk users worldwide:
1. Fork the official Windhawk repository: [ramensoftware/windhawk-mods](https://github.com/ramensoftware/windhawk-mods).
2. Clone your fork locally:
   ```bash
   git clone https://github.com/<your-username>/windhawk-mods.git
   cd windhawk-mods
   ```
3. Copy [`bluetooth-auto-toggle.wh.cpp`](./bluetooth-auto-toggle.wh.cpp) into the `mods/` directory:
   ```bash
   cp "d:\#Anti Gravity\Windhalk\Bluetooth-Automatic-Toogle\bluetooth-auto-toggle.wh.cpp" mods/
   ```
4. Commit, push, and submit a Pull Request:
   ```bash
   git checkout -b add-bluetooth-auto-toggle
   git add mods/bluetooth-auto-toggle.wh.cpp
   git commit -m "Add Bluetooth Auto Toggle mod v1.1.0"
   git push origin add-bluetooth-auto-toggle
   ```

---

## 📋 Technical Details
* **Target Process:** `windhawk.exe` (dedicated standalone tool process)
* **Compiler:** Clang / LLVM (MinGW-w64) via Windhawk Compiler toolchain
* **Required Libraries:** `-lole32 -loleaut32 -lruntimeobject -lshell32`
* **Windows API:** Modern Windows Runtime (`Windows.Devices.Radios`, `Windows.Devices.Bluetooth`, `Windows.Devices.Enumeration`)
* **Resource Impact:** Zero idle CPU consumption; shuts down or sleeps on event handles when idle.

---

## 📄 License
This project is licensed under the [MIT License](LICENSE).
