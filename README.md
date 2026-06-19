# USBArmyKnife for BUG ESP32-S3

This fork ports the [upstream USBArmyKnife project](https://github.com/i-am-shodan/USBArmyKnife) to the BUG ESP32-S3. It provides USB HID, serial, storage, networking, Wi-Fi/Bluetooth tooling, and a browser-based management interface in a compact device.

> **Safety and legal notice:** Use this project only on systems you own or for which you have explicit authorization. Do not run payloads against unauthorized systems. You are responsible for complying with applicable laws, policies, and rules of engagement.

## Tested hardware

- ESP32-S3
- 4 MB embedded flash
- 2 MB embedded PSRAM
- ST7735 160 × 80 display
- SPI microSD
- WS2812-compatible status LED on GPIO 45

The `BUG-ESP32S3` environment in `platformio.ini` is the source of truth for this port. It uses a 4 MB flash layout, the `huge_app.csv` partition table, native USB CDC at boot, and a 115200 baud upload/monitor speed. Touch, infrared, and microphone features are disabled. See [PORTING_NOTES.md](./PORTING_NOTES.md) for implementation details.

## Pinout

| Function | GPIO |
| --- | ---: |
| TFT reset | 5 |
| TFT data/command | 14 |
| TFT MOSI | 17 |
| TFT chip select | 4 |
| TFT clock | 18 |
| TFT backlight | 38 |
| TFT MISO / busy | Not connected (`-1`) |
| microSD clock | 12 |
| microSD MOSI | 11 |
| microSD MISO | 13 |
| microSD chip select | 10 |
| Status LED data | 45 |
| Button / boot | 0 |
| UART RX | 44 |
| UART TX | 43 |

## Installation

### Prerequisites for macOS

- Git
- Python 3
- [PlatformIO Core](https://docs.platformio.org/en/latest/core/installation/index.html), or Visual Studio Code with the PlatformIO extension
- A USB data cable; charge-only cables will not work
- A microSD card is recommended for scripts, files, and the SD browser
- Arduino IDE with the Arduino ESP32 core, only when using the fallback manual `flasher.py` method
- The Python `pyserial` package, only for the fallback 1200-bps reset:

  ```sh
  python3 -m pip install pyserial
  ```

### Clone and build

Replace the repository URL placeholder with the URL of this fork:

```sh
git clone <fork-repository-url> USBArmyKnife
cd USBArmyKnife
pio run -e BUG-ESP32S3
```

The first build downloads the framework and library dependencies. Build outputs are written to `.pio/build/BUG-ESP32S3/`.

### Flash with PlatformIO

This is the normal and preferred upload method. Connect the board, find its port with `ls /dev/cu.usbmodem*`, and run:

```sh
cd /path/to/USBArmyKnife
pio run -e BUG-ESP32S3 -t upload --upload-port "/dev/cu.usbmodemXXXXX"
```

To monitor serial output:

```sh
pio device monitor --port "/dev/cu.usbmodemXXXXX" --baud 115200
```

If PlatformIO cannot put the device into download mode, use the fallback below.

### Fallback: manual flashing on macOS

This method uses the `flasher.py` supplied with the Arduino ESP32 core. Paths and version directory names vary; locate the script and `boot_app0.bin` below `/path/to/Arduino15` and substitute their real paths.

1. Build all BUG ESP32-S3 images:

   ```sh
   cd /path/to/USBArmyKnife
   pio run -e BUG-ESP32S3
   ```

2. Close any serial monitor, then perform a 1200-bps touch reset on the current callout port:

   ```sh
   python3 -c 'import serial; p = serial.Serial("/dev/cu.usbmodemXXXXX", 1200); p.close()'
   ```

3. Wait a moment and identify the bootloader port:

   ```sh
   ls /dev/tty.usbmodem*
   ```

   The bootloader can enumerate with a different port name after the 1200-bps reset. Use the newly appearing `/dev/tty.usbmodemXXXXX` value in the next command, not necessarily the original `/dev/cu.usbmodemXXXXX` value.

4. Flash the four images at their required offsets:

   ```sh
   python3 "/path/to/Arduino15/packages/esp32/tools/esptool_py/<version>/flasher.py" \
     --chip esp32s3 \
     --port "/dev/tty.usbmodemXXXXX" \
     --baud 115200 \
     write_flash \
     --flash-size 4MB \
     0x0 .pio/build/BUG-ESP32S3/bootloader.bin \
     0x8000 .pio/build/BUG-ESP32S3/partitions.bin \
     0xe000 "/path/to/Arduino15/packages/esp32/hardware/esp32/<version>/tools/partitions/boot_app0.bin" \
     0x10000 .pio/build/BUG-ESP32S3/firmware.bin
   ```

5. Reset or reconnect the board after flashing. If no serial port returns, reconnect the USB data cable and list the ports again.

## First boot

After a successful flash, the TFT should show the USBArmyKnife status screen and the device should expose USB serial and HID interfaces. macOS may open Keyboard Setup Assistant and ask you to identify a new keyboard because the device exposes USB HID. This is expected; the assistant can be closed if no keyboard layout setup is needed.

Insert a microSD card if you intend to use scripts or SD-backed features. A startup script is expected at `/autorun.ds` on the card. Treat every payload as active code and use it only on an owned or explicitly authorized system.

## Web UI

The fresh-install defaults are:

| Setting | Default |
| --- | --- |
| Wi-Fi SSID | `iPhone14` |
| Wi-Fi password | `password` |
| Web UI | [http://4.3.2.1:8080](http://4.3.2.1:8080) |

Connect a phone or computer to `iPhone14`, accept the warning if it reports that the network has no internet connection, and browse to `http://4.3.2.1:8080`. The no-internet warning is normal because this access point is a local device-management network. Saved settings can override these defaults. Change the default SSID and password before using the device outside a controlled lab.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| `Failed to connect to ESP32-S3: No serial data received` | Close serial monitors, verify the USB data cable and selected port, repeat the 1200-bps reset, then use the newly enumerated bootloader port. If needed, hold BOOT (GPIO 0), tap RESET, release BOOT, and retry. |
| `screen is terminating` | A reset or USB re-enumeration disconnected the serial session. This is expected during flashing. Exit the stale session, list the ports again, and reconnect to the new port. |
| `Detected size(4096k) smaller than image header(8192k)` | An image was built with an 8 MB flash setting. Clean and rebuild with `pio run -e BUG-ESP32S3 -t clean` followed by `pio run -e BUG-ESP32S3`; confirm both flash-size settings in the environment are `4MB`. |
| No Wi-Fi AP appears | Allow the first boot to finish, verify that the TFT/serial log shows a successful boot, restart the board, and check whether saved settings disabled Wi-Fi or changed the SSID. |
| Web UI does not open | Stay connected to the device AP despite the no-internet warning, disable cellular/VPN routing temporarily if it captures the request, and enter `http://4.3.2.1:8080` exactly, including `http` and port `8080`. |
| macOS asks about a keyboard | This is expected USB HID enumeration. Close Keyboard Setup Assistant unless a layout must be configured. |
| `/autorun.ds` missing | Insert and verify the microSD card, then place a valid `autorun.ds` in the card root (or create/upload one through the SD browser). Check card formatting and SPI wiring if the SD browser cannot see the card. |

## Upstream and license

This port is based on the original [USBArmyKnife project](https://github.com/i-am-shodan/USBArmyKnife). The broader command reference and examples remain available in the [upstream wiki](https://github.com/i-am-shodan/USBArmyKnife/wiki).

USBArmyKnife is distributed under the MIT License. The original attribution is preserved:

> Copyright (c) 2024 i-am-shodan

See [LICENSE](./LICENSE) for the complete license text.
