# Getting usevia.app to Work on Linux (WebHID Permissions)

How VIA (usevia.app) was made to connect to the KBD8X MKII on Omarchy (Arch Linux) on 2026-10-06, after the VIA firmware was flashed (see `01-flashing-via-firmware.md`).

**Short version:** on Linux the browser isn't allowed to open the keyboard's `/dev/hidraw*` device by default. VIA's error makes this look like a firmware problem. A udev rule fixes it, but **only if the rule file's number is below 73.**

---

## The symptom

After authorizing the device at usevia.app, VIA showed:

> VIA can see KBD8X-MKII through WebHID.
> VID: 0xA103 | PID: 0x0005
>
> KBD8X-MKII does not seem to respond like a VIA-enabled keyboard.
>
> If KBD8X-MKII should support VIA, make sure it is running VIA-compatible firmware. ...

**This message is misleading.** WebHID can *list* a device without being allowed to *open* it. VIA then gets no response and assumes the firmware is the problem. Check permissions before you reflash.

## Diagnose: check the Chromium device log

1. Open a new tab and go to `chrome://device-log`.
2. Go back to usevia.app and click **Authorize device** again.
3. Look in the device log for a line like this:

```
[16:11:51] Failed to open '/dev/hidraw5': FILE_ERROR_ACCESS_DENIED
```

- **If you see `FILE_ERROR_ACCESS_DENIED`**, it's a permissions problem. Apply the fix below.
- **If there's no access error** and VIA still says the board isn't VIA-enabled, the firmware may really be too old for current VIA. Building fresh VIA firmware from QMK source would be the next step.

The `hidraw` number (`hidraw5` here) can change between plugs and reboots. Don't hard-code it anywhere permanent.

---

## The fix: a udev rule, numbered below 73

The firmware's USB ID is `a103:0005`. Create the rule file:

```bash
echo 'KERNEL=="hidraw*", SUBSYSTEM=="hidraw", ATTRS{idVendor}=="a103", ATTRS{idProduct}=="0005", MODE="0660", TAG+="uaccess"' | sudo tee /etc/udev/rules.d/70-via.rules
sudo udevadm control --reload-rules && sudo udevadm trigger
```

Then:

1. **Unplug and replug** the keyboard.
2. **Fully quit Chromium** (every window), then reopen it.
3. Go to usevia.app, click **Authorize device**, and select KBD8X-MKII.

What the rule does: it matches the keyboard's raw HID device by vendor/product ID. `TAG+="uaccess"` tells systemd-logind to give the logged-in desktop user read/write access through an ACL. `MODE="0660"` sets the base permissions.

### The gotcha: the file number matters

The first version of this rule was saved as **`92-via.rules`** and **did nothing.**

The `uaccess` tag only takes effect if it's set **before** systemd's `73-seat-late.rules` runs, because that file turns the tag into actual permissions. udev reads rule files in numeric order, so a rule numbered **73 or higher** sets the tag too late and it's ignored.

If you've already created the file with a high number, rename it:

```bash
sudo mv /etc/udev/rules.d/92-via.rules /etc/udev/rules.d/70-via.rules
sudo udevadm control --reload-rules && sudo udevadm trigger
```

Then unplug and replug, and restart Chromium.

> Many guides online use numbers like `92-` or `99-`. Those only work when the rule uses `MODE="0666"` or `GROUP=` instead of `uaccess`. With `uaccess`, stay below 73. `70-` is a safe choice.

---

## Verify the rule is working

Run this with the keyboard plugged in:

```bash
for d in /sys/class/hidraw/hidraw*; do grep -q 'A103' $d/device/uevent 2>/dev/null && getfacl -p /dev/$(basename $d); done
```

Each matching device should list your user with read/write access:

```
user:ngn:rw-
```

If your user isn't listed, the rule isn't being applied. Check the file number, the IDs, and that you reloaded and replugged.

## Temporary workaround (for testing only)

This grants access right away, without the rule. It resets on the next replug or reboot:

```bash
sudo chmod a+rw /dev/hidraw5    # use the hidraw number from chrome://device-log
```

If VIA connects after this, permissions were the problem and the `70-via.rules` file will make the fix permanent.

---

## Other gotchas

- **Use a Chromium-based browser.** Chromium (Omarchy's default), Chrome, Brave, or Edge. **Firefox doesn't support WebHID**, so VIA can't work there.
- **Restart the browser after changing permissions.** Chromium may keep a failed handle until it's fully closed.
- **The rule is per-machine.** Any other Linux computer you use VIA from needs the same `/etc/udev/rules.d/70-via.rules` file. The keymap itself is stored on the keyboard, so you don't need VIA, or this rule, just to *use* the keyboard elsewhere.
- **If you flash different firmware** with a different USB ID, update `idVendor` and `idProduct` in the rule. Find the new ID with `lsusb` or in the VIA error dialog (`VID: 0x.... | PID: 0x....`).

## Quick checklist

- [ ] `/etc/udev/rules.d/70-via.rules` exists (number **below 73**) with `a103` / `0005` and `TAG+="uaccess"`
- [ ] Ran `sudo udevadm control --reload-rules && sudo udevadm trigger`
- [ ] Unplugged and replugged the keyboard
- [ ] Fully restarted Chromium
- [ ] `chrome://device-log` shows no `FILE_ERROR_ACCESS_DENIED`
- [ ] usevia.app → Authorize device → KBD8X-MKII loads
