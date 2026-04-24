# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **ZMK firmware user configuration** for the Sofle split ergonomic keyboard. The repo does not contain ZMK itself — it only holds keymap and config files that GitHub Actions compiles against the upstream ZMK firmware.

Hardware target: **Nice Nano v2** (nRF52840) + Sofle shield (left and right halves).

## Building Firmware

Builds run automatically via GitHub Actions on every push. There is no local build toolchain set up — to trigger a build, simply push a commit.

The build matrix is defined in `build.yaml` and produces two artifacts merged into a single `firmware` zip:
- `sofle_left-nice_nano__zmk-zmk.uf2`
- `sofle_right-nice_nano__zmk-zmk.uf2`

If setting up local builds with the ZMK SDK:
```sh
west build -b nice_nano//zmk -- -DSHIELD=sofle_left
west build -b nice_nano//zmk -- -DSHIELD=sofle_right
```

### Board naming — Zephyr 4.1 breaking change

ZMK migrated to Zephyr 4.1 which introduced a `board/soc/variant` naming format. The Nice Nano v2 was renamed:

| Before (Zephyr < 4.1) | After (Zephyr 4.1+) |
|----------------------|---------------------|
| `nice_nano_v2` | `nice_nano//zmk` |

The `//` means the SoC qualifier (`nrf52840`) is implicit, defined by ZMK's internal `board.yml`. Always use `nice_nano//zmk` in `build.yaml`.

## Flashing Firmware

Downloaded firmware goes to `~/dev-playground/sofle-firmware/`.

1. Connect the **left** half via USB to a PC (Mac blocks bootloader mount — use another machine)
2. Double-tap reset → drive `NICENANO` appears
3. Copy `sofle_left-nice_nano__zmk-zmk.uf2` to the drive → auto-dismounts
4. Repeat with **right** half using `sofle_right-nice_nano__zmk-zmk.uf2`

## Key Files

| File | Purpose |
|------|---------|
| `config/sofle.keymap` | Layer definitions and key bindings (ZMK devicetree syntax) |
| `config/sofle.conf` | Kconfig feature flags (Bluetooth, display, encoders, RGB) |
| `config/west.yml` | ZMK dependency version (points to zmkfirmware/zmk main) |
| `build.yaml` | GitHub Actions build matrix |

## Keymap Architecture

The keymap defines 4 layers using `#define` constants:

| Layer | Number | Activation |
|-------|--------|------------|
| `BASE` | 0 | Default |
| `LOWER` | 1 | Hold left thumb `mo LOWER` key |
| `RAISE` | 2 | Hold right thumb `mo RAISE` key |
| `ADJUST` | 3 | `conditional_layers`: LOWER + RAISE simultaneously |

**Base layer layout** (QWERTY):
- Row 0: ESC, 1-0, DEL
- Row 1: `` ` ``, Q-P, BSPC
- Row 2: TAB, A-;, `'`
- Row 3: LSHIFT, Z-/, RSHIFT | encoders: mute (left), BT_NXT (right)
- Thumb row: LCTRL, LALT, LGUI, LOWER, SPACE | ENTER, RAISE, RCTRL, RALT, RGUI

**Encoder bindings** (per layer):
- BASE: left = volume up/down, right = brightness inc/dec
- LOWER/RAISE: left = volume, right = page up/down

**ADJUST layer key positions** (hold LOWER + RAISE):
- Row 0: BT_CLR, BT_SEL 0-4
- Row 1: EP_TOG (backtick key), RGB_HUD, RGB_HUI, RGB_SAD, RGB_SAI, RGB_EFF
- Row 2: —, RGB_BRD, RGB_BRI
- Encoder: RGB_TOG

## sofle.conf — Feature Flags

Currently active:
```
CONFIG_BT_CTLR_TX_PWR_PLUS_8=y                    # Increased BT TX power
CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y                 # Experimental BT connection mode
CONFIG_ZMK_DISPLAY=y                               # OLED display enabled
CONFIG_ZMK_DISPLAY_BLANK_ON_IDLE=n                 # Keep display always on
CONFIG_ZMK_DISPLAY_STATUS_SCREEN_BUILT_IN=y
CONFIG_ZMK_WIDGET_LAYER_STATUS=y                   # Active layer name (left side)
CONFIG_ZMK_WIDGET_OUTPUT_STATUS=y                  # USB/BT connection status (left side)
CONFIG_ZMK_WIDGET_BATTERY_STATUS_SHOW_PERCENTAGE=y # Battery % (left side)
CONFIG_ZMK_WIDGET_PERIPHERAL_STATUS=y              # Right-half connection status (right side)
```

Commented-out options available to enable:
- `CONFIG_EC11=y` + `CONFIG_EC11_TRIGGER_GLOBAL_THREAD=y` — rotary encoder support
- `CONFIG_ZMK_RGB_UNDERGLOW=y` — RGB underglow
- `CONFIG_ZMK_RGB_UNDERGLOW_EXT_POWER=n` — decouple RGB from external power rail

### All available display widgets

| Config | What it shows | Side |
|--------|--------------|------|
| `CONFIG_ZMK_WIDGET_LAYER_STATUS=y` | Active layer | Left (central) |
| `CONFIG_ZMK_WIDGET_BATTERY_STATUS=y` | Battery level icon | Left (central) |
| `CONFIG_ZMK_WIDGET_BATTERY_STATUS_SHOW_PERCENTAGE=y` | Battery % text | Left (central) |
| `CONFIG_ZMK_WIDGET_OUTPUT_STATUS=y` | USB/BT connection | Left (central) |
| `CONFIG_ZMK_WIDGET_PERIPHERAL_STATUS=y` | Right-half BLE connection | Right (peripheral) |
| `CONFIG_ZMK_WIDGET_WPM_STATUS=y` | Words per minute | Left (central) |

`PERIPHERAL_STATUS` shows a Bluetooth icon on the right display only when the right half has **lost** connection to the left — useful for diagnosing when the right side stops responding.

## Split Keyboard — Important Notes

- Communication between halves is **always wireless BLE** — there is no wired split cable
- The **left half is always the central** (manages BT with host, receives keys from right via BLE)
- The **right half connected via USB** only charges — it does not act as keyboard over that cable
- Both halves can be USB-connected simultaneously: left = active keyboard, right = charging only
- After flashing, Bluetooth pairings must be re-established (use RAISE layer: BT_CLR then BT_SEL 0-4)

## OLED Display — ext_power Issue

The Nice Nano v2 controls power to external peripherals (including the OLED) via GPIO0 pin 13 (`zmk,ext-power-generic`). **This state is persisted to NVS flash.**

If the display only flashes briefly on boot then goes dark, the saved ext_power state is `off`. Fix:

### Option 1 — Toggle via ADJUST layer
Hold LOWER + RAISE → press the backtick key (`` ` `` position) → `EP_TOG` enables ext_power and saves `on` to flash.

### Option 2 — Settings reset (nuclear option, also clears BT pairings)
Add a temporary build entry to `build.yaml`:
```yaml
- board: nice_nano//zmk
  shield: settings_reset
  artifact-name: settings_reset
```
Flash `settings_reset-nice_nano__zmk-zmk.uf2` to **both** halves → NVS is wiped → on next boot ext_power defaults to `on`. Then re-flash normal firmware and re-pair Bluetooth.

Remove the `settings_reset` entry from `build.yaml` after use.

## ZMK Keymap Syntax Reference

- `&kp KEY` — keypress
- `&mo LAYER` — momentary layer
- `&bt BT_SEL N` — select Bluetooth profile N (0-4)
- `&bt BT_CLR` — clear current BT profile
- `&bt BT_NXT` — next BT profile
- `&rgb_ug RGB_*` — RGB underglow controls
- `&ext_power EP_TOG` — toggle external power (controls OLED power on Nice Nano)
- `&trans` — transparent (pass through to lower layer)
- `&none` — no-op (block pass-through)
- `&inc_dec_kp A B` — encoder: clockwise = A, counter-clockwise = B
