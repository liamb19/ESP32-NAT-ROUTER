# ESP32 NAT Router & Wi-Fi Range Extender

A low-power, embedded networking solution that turns an Adafruit HUZZAH32 ESP32 Feather into a functional standalone Wi-Fi repeater. This project utilizes a lightweight Network Address Translation (NAT) engine to securely proxy data packets and extend a localized 2.4GHz wireless footprint using a single radio.

---

## Tech Stack & Hardware
- **Development Board:** Adafruit HUZZAH32 ESP32 Feather (Silicon Labs CP2104 USB-to-UART Bridge)
- **Memory Infrastructure:** 4MB Quad-SPI Flash
- **Flashing Engine:** `esptool.py` (Espressif System Image Tool) via Windows PowerShell
- **Languages/Environment:** Native Espressif Firmware / Serial Debugging via Arduino IDE

---

## System Architecture & The Single-Radio Challenge
Standard commercial Wi-Fi repeaters utilize dual-band or multiple dedicated radios to send and receive data packages simultaneously. Because the ESP32 houses only a **single 2.4GHz radio transceiver**, it cannot transmit and receive at the exact same physical instant.

[Image of Wi-Fi repeater single radio bottleneck concept]

To overcome this hardware limitation, the firmware implements aggressive **time-slicing**. The radio rapidly alternates between acting as a **Station (STA)** to fetch data from the upstream home router, and an **Access Point (AP)** to broadcast to local client devices. This architecture cuts the maximum theoretical bandwidth in half but provides a highly efficient, ultra-low-cost network extension.

---

## Installation & Deployment Guide

Since the firmware relies on specific partition tables, flashing via a standard GUI can cause address collisions. Use the following low-level deployment method via PowerShell.

### 1. Prerequisites
Ensure you have the **Silicon Labs CP2104 Virtual COM Port (VCP) drivers** installed so your operating system recognizes the board's UART bridge.

### 2. Flash Execution
Navigate into your local firmware binary directory and run the following command to map the binaries across the flash sectors cleanly:

```powershell
Get-ChildItem -Path $env:LOCALAPPDATA\Arduino15 -Filter esptool.exe -Recurse | Select-Object -First 1 | ForEach-Object { & $_.FullName --chip esp32 --port COM4 --baud 921600 write_flash -z 0x1000 bootloader.bin 0x8000 partition-table.bin 0xf000 ota_data_initial.bin 0x20000 esp32_nat_router.bin }

## License & Acknowledgments
This project is open-source and available under the terms of the [MIT License](LICENSE). 

### Credits
- Core firmware binaries and NAT routing logic derived from the upstream repository by [martin-ger/esp32_nat_router](https://github.com/martin-ger/esp32_nat_router).