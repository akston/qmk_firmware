# QMK Corne / akston debugging handoff

Date: 2026-09-18

## Current repository state

- Repository: `/home/conor/code/qmk_firmware`
- Branch: `choc_to_keyhive`
- Keymap: `crkbd/akston`
- Target resolved by QMK: `crkbd/rev1`
- This is the legacy ATmega32U4 / Pro Micro-style Corne target.
- Submodules are synchronized to the current branch.

The working tree intentionally contains changes in:

- `keyboards/crkbd/keymaps/akston/config.h`
- `keyboards/crkbd/keymaps/akston/keymap.c`
- `keyboards/crkbd/keymaps/akston/rules.mk`

## Firmware change

The left thumb key in both base layers was changed from `KC_LSFT` to `KC_SPC`.
This fixes the reported behavior where tapping the physical space key produced a
space but holding it produced Shift. The current source now uses plain Space.

## Compatibility fixes made so this old keymap builds on this checkout

- Updated removed keycodes: `KC_LCPO` -> `SC_LCPO`, `KC_RCPC` -> `SC_RCPC`,
  `KC_GESC` -> `QK_GESC`, `KC_RAPC` -> `SC_RAPC`, `RESET` -> `QK_BOOT`,
  `EEP_RST` -> `EE_CLR`.
- Updated RGB Matrix APIs and LED count names.
- Removed obsolete `IGNORE_MOD_TAP_INTERRUPT`.
- Fixed OLED theme paths from `rpbaptist` to `akston`.
- Disabled unused `SEND_STRING_ENABLE`.
- Removed the obsolete `PRODUCT` override.
- Enabled RGB Matrix modes used by the keymap.

## Build environment

An isolated Python environment was created at `.qmk-venv` and contains the QMK
CLI plus this checkout's Python requirements. Arch AVR packages were installed:

- `avr-gcc`
- `avr-binutils`
- `avr-libc`
- `avrdude`
- `dfu-programmer`

The QMK udev rules were installed at `/etc/udev/rules.d/50-qmk.rules` and udev
rules were reloaded.

Activate the environment on another shell:

```sh
source .qmk-venv/bin/activate
```

## Successful build

This command completed successfully:

```sh
make crkbd:akston THEME=pulse ALLOW_WARNINGS=yes
```

Generated firmware:

```text
crkbd_rev1_akston.hex
```

SHA-256 at the time of writing:

```text
d55945085349df6120fadb90dc135eb0567f0f021f782a5f25e146ccce92cc7d
```

`ALLOW_WARNINGS=yes` is needed because modern GCC treats an old LUFA warning
as an error. No shared QMK source was changed for that warning.

## Hardware detection

After waiting five seconds and scanning USB, the connected keyboard appeared as:

```text
VID:PID       4653:0001
Manufacturer  foostan
Product       Corne Keyboard
USB path      3-1
Interfaces    HID, not DFU
```

This is the normal running-firmware device. `dfu-util -l` showed no DFU device.
For the configured QMK DFU bootloader, the expected bootloader ID is:

```text
03eb:2ff4
```

Three 20-second attempts were made with:

```sh
make crkbd:akston:dfu-split-left THEME=pulse ALLOW_WARNINGS=yes
```

All attempts built successfully but ended with:

```text
Bootloader not found
```

Nothing has been flashed yet.

## Remaining problem

The keyboard is healthy enough to enumerate normally as `4653:0001`, but the
connected half has not transitioned into DFU mode. Investigate:

1. Whether the physical reset button is electrically connected.
2. Whether the board actually uses QMK/Atmel DFU or a Caterina bootloader.
3. Whether the current running firmware is really the `akston` firmware.
4. Whether a different USB cable/port is needed.
5. Whether RST can be briefly shorted to GND to force bootloader entry.

Expected DFU check once reset succeeds:

```sh
dfu-util -d 03eb:2ff4 -l
```

For the right half, use:

```sh
make crkbd:akston:dfu-split-right THEME=pulse ALLOW_WARNINGS=yes
```

The split-specific targets are needed because `akston/config.h` defines
`EE_HANDS`.
