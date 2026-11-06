# Bluetooth Auto Toggle (Windhawk Mod)

[![Windhawk Mod](https://img.shields.io/badge/Windhawk-Mod-0078D6.svg)](https://windhawk.net/)
[![Platform](https://img.shields.io/badge/Platform-Windows%2010%20%7C%2011-blue.svg)](https://microsoft.com/windows)
[![Version](https://img.shields.io/badge/Version-1.2.0-green.svg)](#specifications)
[![Architecture](https://img.shields.io/badge/Architecture-x64-success.svg)](#specifications)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A [Windhawk](https://windhawk.net/) tool mod for Windows 10 and 11 that automatically cycles Bluetooth adapter power off and back on in milliseconds when Windhawk boots and a connected Bluetooth peripheral is detected.

---

## Overview and Problem Statement

On Windows 10 and 11 systems especially those utilizing Fast Startup Bluetooth adapters and paired wireless peripherals can encounter driver initialization and power-state synchronization issues following a system boot:

* **Peripheral Wakeup and Audio Pipeline Failures:** Wireless audio endpoints (such as headphones and earbuds) and HID peripherals (mice and keyboards) often display a "Connected" status in Windows while remaining completely unresponsive, failing to route audio, or suffering severe input latency until the Bluetooth radio is power-cycled.
* **Driver Stack Unresponsiveness:** Certain Bluetooth controller drivers fail to reinitialize audio sinks or profile channels cleanly upon cold boots without a manual radio restart via the Windows Action Center or Settings.
* **Redundant Cycling on Idle Systems:** Blindly power-cycling Bluetooth on every startup when no device is paired or actively connected introduces unnecessary radio transitions.

**Bluetooth Auto Toggle** addresses these issues systematically: when Windhawk launches at user logon, the mod observes connected Bluetooth devices through Advanced Query Syntax (AQS) queries. Once a connected device is detected, the mod allows a configurable settle interval for hardware drivers, cycles active Bluetooth radios off and back on within configurable milliseconds, and ensures uninterrupted device reconnection.

---

## Key Features

* **Startup & Manual Turn-On Cycle:** Automatically cycles Bluetooth off and back on at startup if active. If Bluetooth was OFF during boot, the mod listens for when you manually turn it ON later and instantly executes the fast toggle.
* **Device-Aware Activation:** Evaluates device connectivity before initiating cycling, ensuring that idle systems without connected peripherals are not cycled unnecessarily.
* **Configurable Post-Connection Settle Delay:** Introduces a configurable delay (default: 2,000 ms) after device detection to ensure the driver stack and profile layers have settled before executing the toggle.
* **Millisecond Precision Control:** Provides granular configuration over startup settle timing, discovery retry intervals, and radio off-to-on duration in milliseconds.
* **Isolated Tool Mod Architecture:** Executes inside an isolated, dedicated `windhawk.exe` background process using official Windhawk tool mod conventions, preventing any injection into `explorer.exe` or the Windows desktop shell.
* **Hardware Stack Cold-Boot Discovery:** Implements exponential-like discovery retry polling to handle scenarios where the Windows Bluetooth service (`bthserv`) or physical adapter initialization is delayed during startup.
* **State-Safe Radio Operation:** Inspects the active power state of each radio (`RadioState::On`) prior to cycling, ensuring that radios in Airplane mode or administratively disabled states remain untouched.
* **Persistent Session Deduplication:** Employs Windows Terminal Services (WTS) logon timestamps coupled with persistent Windhawk binary storage (`Wh_GetBinaryValue` / `Wh_SetBinaryValue`) to ensure the mod executes only once per logon session, surviving tool process restarts.
* **Safe Power Restoration:** Ensures radios are commanded back on with retries even if the mod is unloaded or interrupted mid-cycle, eliminating scenarios where Bluetooth is left disabled.
* **Live Configuration Testing:** Supports on-demand verification through the `forceToggleOnSettingsChange` setting, allowing immediate testing from the Windhawk configuration interface without requiring a system reboot.

---

## Installation and Configuration

### Prerequisites
1. Windows 10 (version 1809 or later) or Windows 11.
2. [Windhawk](https://windhawk.net/) (v1.4 or later recommended).

### Adding the Mod to Windhawk
1. Open the **Windhawk** application.
2. Navigate to **Settings** (gear icon) and verify **Developer mode** is enabled under the Developer tab.
3. In the **Developer** tab, select **New Mod**.
4. Replace the default template with the complete contents of [`bluetooth-auto-toggle.wh.cpp`](./bluetooth-auto-toggle.wh.cpp).
5. Click **Compile and inject** (or **Compile and test**) in the top-right toolbar. Windhawk will compile the source code via Clang and launch the tool process.

### Configuration Settings Reference

Settings can be customized directly within the mod details view in Windhawk:

| Setting | Default | Description |
| :--- | :---: | :--- |
| **Startup Bluetooth State** | `on` | Initial power state enforcement at startup: force Bluetooth ON (`on`, ensures Bluetooth is active at startup even if turned off before shutdown), force Bluetooth OFF (`off`, stays OFF and automatically executes the fast toggle when you manually turn it back on later), or leave unchanged (`unchanged`, retains Windows state). |
| **Activation Mode** | `startup` | Mode selection: on startup (`startup`, cycles if ON, or triggers on manual turn-on if OFF at boot), or only after a connected Bluetooth device is detected (`deviceConnected`). |
| **Delay After Device Found (ms)** | `2000` | Settle time in milliseconds to wait after device detection before initiating the power cycle. |
| **Device Connection Wait Timeout (seconds)** | `60` | Maximum time window to wait for a device to connect after boot (`0` waits indefinitely until a device is detected). |
| **Startup Settle Delay (ms)** | `2000` | Initial hold time after tool launch before polling hardware, ensuring kernel driver stabilization. |
| **Toggle Off-to-On Duration (ms)** | `500` | Duration in milliseconds the radio remains powered off before issuing the power-on command. |
| **Radio Detection Retries** | `10` | Maximum retry count for radio discovery if the Bluetooth adapter is not immediately reported by the system. |
| **Radio Retry Interval (ms)** | `500` | Interval in milliseconds between discovery attempts. |
| **Toggle Only Once Per Logon Session** | `true` | Prevents redundant execution if Windhawk restarts or settings are reloaded within the same Windows session. |
| **Re-toggle on Settings Save** | `false` | Bypasses session markers, startup delays, and device checks to perform an immediate diagnostic cycle when settings are saved. |

---

## Technical Architecture and Implementation

The mod is engineered to meet strict system-stability and driver-safety requirements:

1. **Process Isolation**
   - Configured via `@include windhawk.exe` to run exclusively as a companion tool process.
   - Decoupled from Windows Explorer and core OS UI components to eliminate shell instability risks.

2. **Asynchronous Non-Blocking Execution**
   - Mod initialization (`Wh_ModInit` / `WhTool_ModInit`) completes immediately, avoiding OS watchdog timeouts.
   - All polling, WinRT async invocations, and wait states execute on a dedicated background worker thread (`BluetoothWorkerThread`) initialized within a Multi-Threaded Apartment (MTA).

3. **Hardware State Transition Safety**
   - Instantaneous 0 ms radio toggling can cause Bluetooth firmware or hardware controllers to reject commands (`RadioAccessStatus::Denied`).
   - The configurable `toggleDelayMs` ensures physical radios enter full low-power state before receiving the power-on instruction.

4. **Exception Handling and Resiliency**
   - Direct WinRT API access (`Windows.Devices.Radios.Radio`, `Windows.Devices.Bluetooth.BluetoothDevice`, `Windows.Devices.Enumeration.DeviceInformation`) is wrapped in structured `try...catch` handlers.
   - Radio names and references are cached to avoid unhandled `winrt::hresult_error` exceptions during power-down transitions.

---

## Specifications

* **Target Process:** `windhawk.exe` (dedicated tool mod process)
* **Toolchain:** Clang / LLVM (MinGW-w64 target) via the Windhawk Compiler suite
* **Compiler Flags / Dependencies:** `-lole32 -loleaut32 -lruntimeobject -lshell32 -lwtsapi32`
* **Windows Runtime APIs:** `Windows.Devices.Radios`, `Windows.Devices.Bluetooth`, `Windows.Devices.Enumeration`
* **Resource Profile:** Zero idle CPU consumption; sleeps on OS event handles (`g_hStopEvent`, `g_hTriggerEvent`) when not cycling.

---

## License

This project is licensed under the [MIT License](LICENSE).
