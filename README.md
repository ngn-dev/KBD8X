# KBDfans KBD8X MKII: VIA on Linux

Firmware, keymap, and notes from flashing VIA firmware onto a **KBDfans KBD8X MKII** (ATmega32U4, Atmel DFU bootloader) and getting [usevia.app](https://usevia.app) working on **Linux** (Omarchy / Arch).

> Everything here is specific to the **KBD8X MKII**. Don't flash this `.hex` file onto any other board.

## Contents

| File | Description |
|---|---|
| [`01-flashing-via-firmware.md`](01-flashing-via-firmware.md) | How to flash with `dfu-programmer`, since QMK Toolbox has no Linux build. Includes a command that waits for bootloader mode, so you can flash using only the keyboard being flashed. |
| [`02-via-linux-permissions.md`](02-via-linux-permissions.md) | How to fix VIA's misleading "does not seem to respond like a VIA-enabled keyboard" error. It's a WebHID permissions problem, fixed with a udev rule that must be numbered **below 73**. |
| `kbdfans_kbd8x_mk2_via.hex` | The VIA-compatible firmware that was flashed. USB ID: `a103:0005`. |
| `dark-tangies-kbd8x_mkii.layout.json` | My keymap, exported from VIA. Load it from VIA's **Save + Load** section. |

## Quick start

1. Flash the firmware by following [`01-flashing-via-firmware.md`](01-flashing-via-firmware.md).
2. Add the udev rule from [`02-via-linux-permissions.md`](02-via-linux-permissions.md), then replug the keyboard and restart the browser.
3. Open usevia.app in a Chromium-based browser (Firefox lacks WebHID), authorize the KBD8X-MKII, and load the layout JSON.
