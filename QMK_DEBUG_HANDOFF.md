# QMK Corne / akston handoff

Updated: 2026-09-18

## Current status: working, both halves flashed

The user confirmed both halves type correctly, holding Space no longer produces
Shift, and the custom displays are back to normal. There is no outstanding
reported problem. USB is connected to the left half.

Both halves run the Pulse-theme OLED-fix firmware, with their respective
left/right EEPROM handedness settings. Firmware and EEPROM validation succeeded
on both halves. Windows subsequently reported normal keyboard/HID devices with
status `OK` and USB sharing state `Not shared`.

- Repository: `/home/conor/code/qmk_firmware`
- Branch: `choc_to_keyhive`
- Keymap: `crkbd/akston`, resolving to `crkbd/rev1`
- Controller: ATmega32U4, QMK/Atmel DFU bootloader (confirmed on hardware)
- Handedness: `EE_HANDS` in the keymap config
- Firmware: `crkbd_rev1_akston.hex`, 25,458 / 28,672 bytes (88%)
- Flashed firmware SHA-256:

```text
421685160d67ef4650d424fac43aeea0542856eabc4bed7175d7053a037b2284
```

Generated firmware and the Python environment are local build artifacts, not
files to commit. Submodules are initialized at the branch's pinned revisions.

## Changes retained

Commit `444d70fa90` contains the earlier compatibility work and plain `KC_SPC`
on the left thumb in both base layers. That work updated removed keycodes,
RGB Matrix APIs and LED counts, corrected OLED font paths, removed obsolete
configuration, and enabled the RGB effects used by this keymap.

The subsequent OLED fix updates `config.h`, `keymap.c`, and `rules.mk`:

- Replace obsolete `OLED_DRIVER_ENABLE` with `OLED_ENABLE`.
- Use `is_keyboard_master()` instead of the removed global `is_master`.
- Use `host_keyboard_led_state()` for Num Lock and Caps Lock indicators.
- Return `false` from the bool `oled_task_user()` callback, including its timeout
  path, to prevent generic keyboard rendering after custom rendering.

Previously the obsolete guards silently excluded the custom display code, so
Corne's generic renderer displayed layer names that did not match this keymap.
It displayed `Undef` for UTIL. No utility-layer logic or mappings were changed:
UTIL activates when SYM and NUM are both active (hold both middle thumb keys).

`NUM` under `Mode` on the custom display means the host reports **Num Lock on**.
It is not the number layer. Layer names appear separately as `Sym`, `Num`, or
`Util`. The user asked about this indicator; it does not require a firmware fix.

## Build environment on jayne

This machine runs Ubuntu 22.04 under WSL2. Installed locally:

- `.qmk-venv`: system Python 3.10, QMK CLI 1.2.0, and `requirements.txt`.
  Excluded through `.git/info/exclude`.
- AVR GCC 5.4.0, AVR binutils/libc, avrdude 6.3, dfu-programmer 0.6.1,
  dfu-util 0.9, build tools, and USB/HID libraries.
- ARM GCC 10.3 and newlib for general QMK tooling; this Corne uses AVR.
- QMK udev rules at `/etc/udev/rules.d/50-qmk.rules`.

The standard uaccess rules did not grant this WSL session write access. Added
`/etc/udev/rules.d/60-qmk-wsl-dfu.rules` with:

```udev
SUBSYSTEM=="usb", ATTR{idVendor}=="03eb", ATTR{idProduct}=="2ff4", GROUP="plugdev", MODE="0660"
```

The user belongs to `plugdev`. After reloading and triggering udev, DFU commands
work without sudo. These system files and packages are not part of the commit.

Build:

```sh
source .qmk-venv/bin/activate
make crkbd:akston THEME=pulse ALLOW_WARNINGS=yes
```

The final build passed without compiler warnings. `ALLOW_WARNINGS=yes` was
retained from the previous Arch setup, whose newer compiler reported a LUFA
warning. No shared QMK source was changed. The final build log is temporarily
available at `/tmp/qmk-akston-oled-build.log`.

`qmk doctor` found all dependencies installed and submodules up to date. At the
time of checking, its remaining warnings concerned uncommitted changes and
absence of an official `upstream` remote. APT also reported an unrelated
HashiCorp repository signing-key error; Ubuntu package installation succeeded.
The HashiCorp configuration was not changed.

## Flashing from WSL

The user uses this keyboard to communicate. Once they announce DFU entry, do
not wait for another typed reply: complete the flash, then report when typing
is available again. If physical reset is needed after a failure, say so. For
the last two-half update, the user requested left first, a prompt to switch,
a 10-second wait, then right automatically. Coordinate future hardware swaps
before taking the keyboard offline.

Disconnect USB power before connecting or disconnecting the inter-half cable.
Identify the connected half before selecting its EEPROM file.

- Normal keyboard: `4653:0001`
- DFU bootloader: `03eb:2ff4`, `ATm32U4DFU`
- Last Windows BUSID: `4-4`; recheck rather than assuming it is unchanged
- `dfu-programmer atmega32u4 get bootloader-version` returns `0x00`

Windows has usbipd-win 5.0.0 installed. It reports an incompatible USBPcap
filter requiring forced binding. The DFU binding remains persisted as
`e193ecc4-11c9-48bf-9d22-16028fac083b`; normal keyboard mode is not bound.

After physical reset, inspect `usbipd.exe list`. If the bootloader needs binding,
run this in administrator PowerShell (elevation may show UAC):

```powershell
usbipd bind --force --busid <BOOTLOADER_BUSID>
```

Attach from WSL, allow enumeration to settle, and verify access:

```sh
usbipd.exe attach --wsl --busid <BOOTLOADER_BUSID>
sleep 2
lsusb
dfu-programmer atmega32u4 get bootloader-version
```

Flash with commands chained by `&&` so any failed write stops the sequence:

```sh
dfu-programmer atmega32u4 erase &&
dfu-programmer atmega32u4 flash-eeprom quantum/split_common/eeprom-lefthand.eep &&
dfu-programmer atmega32u4 flash crkbd_rev1_akston.hex &&
dfu-programmer atmega32u4 reset
```

For the right half, substitute `eeprom-righthand.eep`. These are the operations
used by the checkout's `dfu-split-left` / `dfu-split-right` targets. AVR DFU uses
**dfu-programmer**, not dfu-util. Successful writes report validation and byte
counts (15 EEPROM bytes, 25,458 firmware bytes for this build).

Reset returned exit code 1 as USB disconnected, but normal firmware immediately
enumerated in Windows. Verify `4653:0001`, `Not shared`, and healthy keyboard/HID
status rather than treating reset's exit code alone as a failed flash.

USB attachment dropped several times, including once during the left firmware
write. The bootloader remained accessible in Windows. Reattachment followed by
a complete erase/EEPROM/firmware sequence recovered the half and validated.
Do not switch halves after a failed write; recover and verify the current half
first. Firmware and EEPROM backup reads failed before the first flash, so no
usable backup of the original device contents exists.

If cleanup is needed, `usbipd detach --busid <BUSID>` releases an attached device;
administrator `usbipd unbind --busid <BUSID>` removes forced binding. Neither is
needed for normal typing now: the running keyboard is already back in Windows.

Reference: https://learn.microsoft.com/en-us/windows/wsl/connect-usb
