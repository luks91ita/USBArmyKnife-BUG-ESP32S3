# BUG ESP32-S3 porting notes

This document records the hardware-specific changes and current verification status for the BUG ESP32-S3 port of [USBArmyKnife](https://github.com/i-am-shodan/USBArmyKnife). The upstream source remains the foundation; the port adds a dedicated PlatformIO environment and maps upstream hardware abstractions to the BUG board.

## PlatformIO environment

The port is defined by `[env:BUG-ESP32S3]` in `platformio.ini`. It:

- Extends the shared `core-esp32-s3` configuration.
- Uses PlatformIO board definition `esp32-s3-devkitc-1`.
- Selects the existing `POCKET_DONGLE_S3` hardware path.
- Enables SD support through `HAS_SD` and the SPI `USE_SD_INTERFACE` path.
- Enables native USB CDC at boot with `ARDUINO_USB_MODE=0` and `ARDUINO_USB_CDC_ON_BOOT=1`.
- Uploads and monitors at 115200 baud.
- Uses the normal reset before upload and a hard reset afterward.
- Adds the APA102 and FastLED libraries required by inherited and board-specific LED support.

Build the port with:

```sh
pio run -e BUG-ESP32S3
```

## Flash and partition constraints

The BUG ESP32-S3 has 4 MB of real embedded flash. Both PlatformIO's build and upload views must agree with the physical device:

```ini
board_build.flash_size = 4MB
board_upload.flash_size = 4MB
```

Leaving the upload size at 8 MB can produce an image-header mismatch such as `Detected size(4096k) smaller than image header(8192k)` even when compilation succeeds.

The environment uses:

```ini
board_build.partitions = huge_app.csv
```

The application does not fit in the app partition provided by `default.csv`. `huge_app.csv` provides the larger application partition needed by this firmware while remaining within the board's 4 MB flash limit.

## Pin remapping

### TFT display

The BUG board uses an ST7735-compatible 160 × 80 TFT. Its mapping is:

| Signal | Value |
| --- | ---: |
| `DISPLAY_RST` | 5 |
| `DISPLAY_DC` | 14 |
| `DISPLAY_MOSI` | 17 |
| `DISPLAY_CS` | 4 |
| `DISPLAY_SCLK` | 18 |
| `DISPLAY_MISO` | -1 (not connected) |
| `DISPLAY_BUSY` | -1 (not connected) |
| `DISPLAY_LEDA` | 38 |
| `DISPLAY_WIDTH` | 160 |
| `DISPLAY_HEIGHT` | 80 |

The inherited display code also uses `TFT_WIDTH=80` and `TFT_HEIGHT=160` for orientation.

### SPI microSD

| Signal | GPIO |
| --- | ---: |
| `SD_SCLK_PIN` | 12 |
| `SD_MOSI_PIN` | 11 |
| `SD_MISO_PIN` | 13 |
| `SD_CS_PIN` | 10 |

### LED, button, and UART

| Function | Configuration |
| --- | --- |
| Status LED | `LED_DI_PIN=45`, `NUM_LEDS=1` |
| Button / boot | `BTN_PIN=0` |
| UART receive | `RX_PIN=44` |
| UART transmit | `TX_PIN=43` |

The UART pin definitions are inherited from `core-esp32-s3`.

## Disabled or unsupported features

The BUG hardware port explicitly defines:

- `NO_TOUCH`
- `NO_IR`
- `NO_MIC`

Code paths requiring a touchscreen, infrared hardware, or a microphone are therefore unavailable in this environment.

## Verification status

The following behavior has been verified on the tested BUG ESP32-S3 hardware:

- Firmware boots.
- The TFT displays the USBArmyKnife status screen.
- The Wi-Fi access point starts.
- The Web UI is reachable at [http://4.3.2.1:8080](http://4.3.2.1:8080).
- The SD browser works.
- USB Serial and USB HID enumerate successfully.
- The Marauder command menu is visible.

Advanced payloads, attack flows, storage modes, networking modes, and optional modules require further board-specific testing. Visibility in the UI or command menu should not be treated as proof that every advanced operation has been validated on this port.

## Upstream and license

This port retains attribution to [USBArmyKnife](https://github.com/i-am-shodan/USBArmyKnife) and remains subject to the repository's MIT License:

> Copyright (c) 2024 i-am-shodan

See [LICENSE](./LICENSE) for the complete license text.
