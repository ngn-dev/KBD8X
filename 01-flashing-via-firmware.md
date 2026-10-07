# Flashing VIA Firmware on the KBDfans KBD8X MKII (Linux)

How the KBD8X MKII was flashed with VIA-compatible firmware from Omarchy (Arch Linux) on 2026-10-06.

After flashing, the browser still couldn't talk to the board. That fix is in `02-via-linux-permissions.md`.

---

## Files in this folder

| File | What it is |
|---|---|
| `kbdfans_kbd8x_mk2_via.hex` | The VIA firmware that was flashed. USB strings: "KBD8X-MKII" by "KBDfans", for the ATmega32U4 chip. USB ID once running: `a103:0005`. |
| `dark-tangies-kbd8x_mkii.layout.json` | My keymap, exported from VIA. Load it back in after any reflash. |

## Key facts about the board

- **MCU:** ATmega32U4
- **Bootloader:** Atmel DFU. In bootloader mode its USB ID is `03eb:2ff4`.
- **Flashing can't brick the board.** The bootloader sits in a protected area that flashing doesn't touch. If a flash fails, the board comes back up in bootloader mode and you can flash again.
- **Flashing wipes the keymap** back to the firmware default. Re-import the layout JSON afterwards (see Step 5).

---

## Gotcha 1: QMK Toolbox doesn't run on Linux

QMK Toolbox, the usual GUI flasher, only ships Windows and macOS builds. It isn't in the Omarchy repos or the AUR, and **you don't need to build it from source.** On Windows it's just a front end for the command-line tool `dfu-programmer`, which is the right flasher for this board's Atmel DFU bootloader. On Linux you run `dfu-programmer` directly.

## Gotcha 2: In bootloader mode the keyboard can't type

When the board is in bootloader mode it stops acting as a keyboard until flashing finishes and it resets. If it's your only keyboard, you can't type the flash commands at that point.

**Fix:** type the whole command and enter your sudo password **first**, with a command that waits for the bootloader to show up and then flashes on its own. You only put the board into bootloader mode after that. No second keyboard is needed.

---

## Step 1: Install dfu-programmer

```bash
sudo pacman -S dfu-programmer
# if pacman can't find it:
yay -S dfu-programmer
```

## Step 2: Go to the firmware folder

```bash
cd ~/Documents/KBD8X
```

## Step 3: Start the wait-then-flash command

```bash
sudo sh -c 'until lsusb -d 03eb:2ff4 >/dev/null; do sleep 1; done; dfu-programmer atmega32u4 erase --force && dfu-programmer atmega32u4 flash kbdfans_kbd8x_mk2_via.hex && dfu-programmer atmega32u4 reset'
```

Enter your sudo password **now**, while the keyboard still works. The terminal will then sit waiting.

What the command does:

1. `until lsusb -d 03eb:2ff4 ...; do sleep 1; done` checks once a second until the Atmel DFU bootloader appears on USB.
2. `erase --force` clears the flash memory.
3. `flash <file>.hex` writes the firmware and verifies it.
4. `reset` restarts the board out of bootloader mode, and it works as a keyboard again.

The `&&` between steps means a failed step stops the rest from running.

> **Older dfu-programmer:** if `erase --force` returns an error, your version predates that flag. Use plain `erase` instead.

## Step 4: Put the board into bootloader mode

Do one of these:

- Press the **reset button** on the underside of the PCB. It may be reachable through a hole in the case.
- **Unplug** the board, **hold Esc**, and plug it back in.

The waiting command picks the board up within about a second and flashes it. Use a data-capable USB cable plugged straight into the PC, not through a hub.

### Successful output

```
Erasing flash...  Success
Checking memory from 0x0 to 0x6FFF...  Empty.
Checking memory from 0x0 to 0x59FF...  Empty.
0%                            100%  Programming 0x5A00 bytes...
[>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>]  Success
0%                            100%  Reading 0x7000 bytes...
[>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>]  Success
Validating...  Success
0x5A00 bytes written into 0x7000 bytes memory (80.36%).
```

`Validating... Success` means the firmware was written and verified. The board should type again right away.

## Step 5: Connect VIA and restore the keymap

1. Complete the one-time permissions setup in `02-via-linux-permissions.md`. VIA won't connect on Linux without it.
2. Open **https://usevia.app** in **Chromium** (or Chrome/Edge). Firefox doesn't support WebHID.
3. Click **Authorize device** and select **KBD8X-MKII**.
4. To restore my layout, go to VIA's **Save + Load** section, click **Load**, and select `dark-tangies-kbd8x_mkii.layout.json`.

VIA writes changes straight to the keyboard's onboard memory. The layout stays on the board when you plug it into another computer, even one without VIA, and you don't need to flash again to change keys.

---

## If something goes wrong

| Symptom | What to do |
|---|---|
| The command waits forever after you press reset | Check the board is in bootloader mode with `lsusb -d 03eb:2ff4` (from another keyboard, or by trying again). Try the Esc-while-plugging method, a different cable, or a different port without a hub. |
| `erase --force` errors | Older dfu-programmer. Replace `erase --force` with `erase`. |
| Flash fails partway; board is stuck in bootloader and won't type | Unplug and replug it. It comes back in bootloader mode because the bootloader is untouched. Run the Step 3 command again, using another keyboard if needed. |
| `Permission denied` / device not opened | Make sure the command ran with `sudo`. |
| VIA says "does not seem to respond like a VIA-enabled keyboard" | Almost always a browser permissions problem on Linux, not the firmware. See `02-via-linux-permissions.md`. |
| VIA connects but shows the board as unsupported or unknown | Turn on **Show Design tab** in VIA's settings and load the KBD8X MKII JSON definition (from KBDfans or the `the-via/keyboards` GitHub repo). This wasn't needed this time. |
