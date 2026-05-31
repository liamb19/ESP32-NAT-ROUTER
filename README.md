# ESP32 NAT Router & Wi-Fi Range Extender

A low-power, embedded networking solution that transforms an Adafruit HUZZAH32 ESP32 Feather into a functional, standalone Wi-Fi repeater. This project leverages a lightweight Network Address Translation (NAT) engine to securely proxy data packets and extend a localized 2.4GHz wireless footprint using a single radio.

---

## Tech Stack & Hardware

- **Development Board:** Adafruit HUZZAH32 ESP32 Feather (Silicon Labs CP2104 USB-to-UART Bridge)
- **Memory Infrastructure:** 4MB Quad-SPI Flash
- **Flashing Engine:** `esptool.py` (Espressif System Image Tool) automated via Windows PowerShell
- **Environment & Diagnostics:** Native Espressif Firmware / Low-level Register Debugging via Arduino IDE Serial Monitor

---

## System Architecture & The Single-Radio Challenge

Standard commercial Wi-Fi repeaters utilize dual-band architectures or multiple dedicated radios to transmit and receive data packages simultaneously. Because the standard ESP32 houses only a **single 2.4GHz radio transceiver**, it cannot transmit to a client and receive from an upstream gateway at the exact same physical instant.

To circumvent this hardware constraint, the firmware implements aggressive **time-slicing**. The radio rapidly alternates roles at the microsecond level: acting as a **Station (STA)** to fetch data from the upstream home router, and an **Access Point (AP)** to broadcast to local client devices. 


---

## Installation & Deployment Guide

Because this firmware configuration relies on custom partition tables, flashing via standard graphical user interfaces (GUIs) can override system offsets, causing data collisions. Follow this verified terminal deployment pipeline instead.

### 1. Hardware & Driver Configuration
Before flashing, your operating system must establish a valid serial link with the board's UART bridge.
1. Download and install the official **Silicon Labs CP210x Virtual COM Port (VCP) Drivers**.
2. Connect your ESP32 board to your PC using a dedicated Micro-USB data cable (ensure it is not a charge-only cable).
3. Open Windows **Device Manager**, expand **Ports (COM & LPT)**, and note your board's active port assignment (e.g., `COM4`).

### 2. Prepare the Workspace Environment
1. Clone this repository or download the source release binaries.
2. Open Windows **PowerShell** or Terminal.
3. Change directories into the specific folder containing the compiled binary assets:
   ```powershell
   cd ESP32-NAT-Router/firmware
   ```

### 3. Clear Serial Port Locks
To prevent an Access is denied permission fault (PermissionError(13)), ensure all external programs interfacing with your system's serial buses (such as the Arduino Serial Monitor, IDEs, or graphical Espressif Flash tools) are completely closed before running the script.

### 4. Execute the Flash Script via PowerShell
Rather than relying on erratic GUI tools, deploy the codebase utilizing the standalone esptool.exe wrapper bundled within your local Arduino application data folders. Copy and paste the following automated pipeline command:

```powershell
Get-ChildItem -Path $env:LOCALAPPDATA\Arduino15 -Filter esptool.exe -Recurse | Select-Object -First 1 | ForEach-Object { & $_.FullName --chip esp32 --port COM4 --baud 921600 write_flash -z 0x1000 bootloader.bin 0x8000 partition-table.bin 0xf000 ota_data_initial.bin 0x20000 esp32_nat_router.bin }
```

### 5. Firmware Memory Map
The binaries must be mapped to these exact memory offsets to avoid data overlapping and bootloader panic loops:

| Binary File | Flash Address | Description |
| :--- | :--- | :--- |
| `bootloader.bin` | `0x1000` | Microcontroller boot sequence |
| `partition-table.bin` | `0x8000` | Defines memory layout boundaries |
| `ota_data_initial.bin` | `0xf000` | Manages Over-The-Air update states |
| `esp32_nat_router.bin` | `0x20000` | Main application routing binary |

### 6. Post-Flash Sequence & Kickstarting Runtime
1. Wait for the terminal to print Writing at 0x00020000... (100%) followed by Leaving... Hard resetting via RTS pin....
2. To clear the auto-reset pin's programming state lock, physically unplug the Micro-USB cable from the board for 5 seconds to drain the power capacitors completely to zero.
3. Re-plug the board into a power source (a standalone USB wall charger adapter is recommended to bypass PC data line enumeration queries).
4. Tap the physical Reset button on the board once to exit bootloader mode and jump execution to the live application code.

### 7. Wireless Configuration
1. Scan for nearby networks on your phone or computer and connect to the newly broadcasting open network (Default SSID: ESP32_NAT_Router).
2. Open a web browser window and navigate directly to the local gateway configuration page at 192.168.4.1.
3. Enter your primary home router details in the Station Settings block to bridge the link, then hit save.

### Link Budget & Optimisation
- The performance of a single-radio NAT router is heavily dependent on its physical link budget.
- Poor Placement: An RSSI of -82 dBm or worse indicates severe attenuation, leading to high packet drop rates and slow throughput.
- Optimal Placement: Position the hardware roughly halfway between the primary router and the dead zone to achieve an uplink RSSI between -50 dBm and -65 dBm.

### Challenges and Debugging Logs
1. Driver Layer Disconnect
Symptom: The hardware was entirely unrecognized by the OS, showing no active COM connections.
Resolution: Diagnosed the issue as a missing Virtual COM Port layer and manually implemented the Silicon Labs CP2104 driver architecture to handle serial translation.

2. Memory Address Mapping & Bootloop Faults
Symptom: The bootloader panicked on startup, printing: E (78) boot: No bootable app partitions in the partition table.
Resolution: Interpreted the registers via the Arduino Serial Monitor at 115200 baud to find that standard GUI tools were pushing default firmware offsets that caused data overlapping. Manually restructured the partition map properties via command line to push ota_data_initial.bin cleanly to 0xf000 and the main app binary to 0x20000.

3. Hard-Reset Auto-Trigger Faults
Symptom: The board failed to automatically jump into runtime execution following a 100% Finish terminal command transmission.
Resolution: Isolated the issue to the auto-reset circuit hanging in a low state, resolved by introducing a manual power cycle to thoroughly drain the on-board power rails.

### License and Acknowledgements
This project is open-source and available under the terms of the MIT License.
Core firmware binaries and NAT routing logic derived from the upstream repository by martin-ger/esp32_nat_router.
