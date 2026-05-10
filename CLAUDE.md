# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ZMK firmware configuration for a **Corne Choc** (42-key split) keyboard with **nice!nano v2** controllers, plus a **PS/2 TrackPoint** (right half, central) and **one rotary encoder with push-switch per side**. The owner is an embedded software developer who primarily uses nvim, types in both German and English, and uses **US International** keyboard layout (sometimes regular US on other machines).

## Build System

There is **no local build**. Firmware is built via GitHub Actions on every push to `main`. The workflow (`.github/workflows/build.yml`) calls the ZMK reusable workflow. The build matrix in `build.yaml` produces three artifacts: `nice_nano_v2 + corne_tp_left`, `nice_nano_v2 + corne_tp_right`, and `nice_nano_v2 + settings_reset` (for wiping BLE pairings).

The west manifest pulls **`infused-kim/zmk@pr-testing/mouse_ps2_module_base`** (a fork that includes the mouse PR and is the only ZMK build currently confirmed to compile cleanly with the `kb_zmk_ps2_mouse_trackpoint_driver` module). Re-evaluate the fork choice once the upstream pointing-device PR merges.

## Key Files

- `config/corne_tp.keymap` — Keymap (7 layers, custom behaviors, encoder sensor-bindings)
- `config/corne_tp.conf` — Kconfig: mouse, PS/2, USB logging, BT TX power, power management. Filename must match shield (`corne_tp`) — ZMK's conf discovery looks for `<shield>.conf`, not `corne.conf`.
- `config/west.yml` — West manifest (infused-kim ZMK fork + PS/2 driver module + dhruvinsh/zmk-tri-state)
- `build.yaml` — GitHub Actions build matrix
- `boards/shields/corne_tp/` — Local shield (replaces upstream Corne):
  - `corne_tp.dtsi` — Shared base: matrix transform (5×12), kscan-composite, encoders, sensors
  - `corne_tp_left.overlay` — Left half (peripheral): matrix col-gpios, enables left encoder
  - `corne_tp_right.overlay` — Right half (central): matrix col-gpios, enables right encoder, full PS/2 high-freq pin block + interrupt-priority demotions
  - `boards/nice_nano_v2.overlay` — Disables `&spi3`; the upstream Corne shield bound it to WS2812 on `P0.06`, which is now PS/2 SCL
  - `Kconfig.shield`, `Kconfig.defconfig`, `corne_tp.zmk.yml` — shield metadata; defconfig forces RIGHT as central
- `config/include/mouse_tp.dtsi` — TP runtime-tuning macros (`U_MSS_*`) and side-conditional overrides; included from the keymap

## Pin allocation (nice!nano v2)

| Function           | Left (D)        | Right (D)       | nRF52 pin |
|--------------------|-----------------|-----------------|-----------|
| Matrix rows        | 4, 5, 6, 7      | 4, 5, 6, 7      | -         |
| Matrix cols        | 14, 15, 18-21   | 14, 15, 18-21   | -         |
| Encoder A          | 16              | 16              | P0.10     |
| Encoder B          | 10              | 10              | P0.09     |
| Encoder click      | 8               | 8               | P1.04     |
| TP SCL             | -               | 1               | P0.06     |
| TP SDA             | -               | 0               | P0.08     |
| TP POR             | -               | 9               | P1.06     |
| **Free spares**    | 0, 1, 2, 3, 9   | 2, 3            | -         |

`P0.06` is reused: it was the upstream Corne's WS2812 underglow MOSI (SPI3) and is now PS/2 SCL. RGB underglow is **deliberately not built** — there is no `WS2812` binding anywhere in this config.

## Keymap Architecture

### Layers

| Index | Name | Access |
|-------|------|--------|
| 0 | Colemak-DH | Base layer |
| 1 | Symbols + Numpad | Right thumb `mo 1` |
| 2 | Extend / Navigation | Left inner thumb `mo 2` |
| 3 | Function + Media | **Conditional**: hold L1 + L2 simultaneously |
| 4 | Bluetooth + TP-set entry | `mo 4` from layer 2 |
| 5 | Mouse / TrackPoint | **Auto-activated** by TP movement (250 ms debounce) |
| 6 | TrackPoint settings | `&tog 6` from layers 4 and 5 |

### Key position map

42 matrix positions + 2 encoder click positions = 44 positions total.

```
Row 0:    0   1   2   3   4   5       6   7   8   9  10  11
Row 1:   12  13  14  15  16  17      18  19  20  21  22  23
Row 2:   24  25  26  27  28  29      30  31  32  33  34  35
Thumb:           36  37  38              39  40  41
Encoder:                42                       43
```

The matrix-transform extends to `rows = <5>`; row 4 columns 0 and 6 host the encoder clicks (left and right respectively). The right side's transform `col-offset = <6>` handles both matrix and encoder click shifts.

### Encoder bindings (rotation)

- **Left encoder**: `&inc_dec_msc SCRL_RIGHT SCRL_LEFT` — horizontal scroll
- **Right encoder**: `&inc_dec_msc SCRL_DOWN SCRL_UP` — vertical scroll

Both encoders click as **middle mouse button** (`&mkp MCLK`). Layer 3 overrides rotation with media: volume +/- and track next/prev.

### Thumb cluster layout

```
Left:  mt(LCTRL,ESC)  SPACE  mo2  |  mo1  LSHFT  BSPC  :Right
```

- `&mt LCTRL ESC` — tap for ESC (nvim normal mode), hold for Ctrl
- Space and Shift on separate hands for fast German capitalization (`Space → Shift+letter`)
- Layers 1 and 2 use `&trans` on the opposite layer key position to enable the conditional layer 3

### Custom behaviors

- **`swapper`** (tri-state, from `zmk-tri-state` module): Alt-Tab window switcher.
- **`skh`** (sticky-or-hold): Tap = sticky modifier, hold = plain modifier (no latch on release).
- **`inc_dec_msc`** (sensor-rotate-var bound to `&msc`): scroll-on-rotation; the stock `inc_dec_kp` only binds to `&kp`.

### Design decisions

- **No numpad keycodes for numbers**: Layer 1 uses `N0`–`N9` instead of `KP_N0`–`KP_N9` to avoid NumLock dependency. KP operators are kept since they are NumLock-independent.
- **`BSLH` over `NON_US_BSLH`**: Regular backslash works on both US and US International layouts.
- **`RA(S)` and `RA(N5)`**: AltGr+S for ß and AltGr+5 for € (US International).
- **5 BT profiles** on layer 4 (`BT_SEL 0`–`4`) with `BT_CLR` on the same row.
- **Right half is central** (BLE peripheral): mandatory because the PS/2 driver must run on the central side. Set in `boards/shields/corne_tp/Kconfig.defconfig`.
- **High-frequency PS/2 pins** (D0/D1/D9): chosen over the low-freq variant for cleaner Bluetooth coexistence; cost is no RGB underglow.

## TrackPoint Tuning

- Hold inner thumb (mo MOUSE_KEYS) → press outer-top key → enter MOUSE_TP_SET layer.
- Settings keys (`U_MSS_TP_*`) increment/decrement sensitivity, value6 (max accel), negative inertia, press-to-select threshold.
- `U_MSS_LOG` prints the current settings to USB log; copy them into `&mouse_ps2 { … }` in `corne_tp_right.overlay` for permanent settings.
- `U_MSS_RESET` wipes flash settings and restores TP defaults.
- Settings are saved to flash 60 s after the last change (`CONFIG_ZMK_SETTINGS_SAVE_DEBOUNCE`).

## Conventions

- **ASCII art diagrams** in keymap comments document each layer visually. Always update these when modifying bindings.
- **`[L1]`/`[L2]` notation** in diagrams marks `&trans` keys (pass-through for conditional layer activation).
- Keymap uses ZMK devicetree syntax with includes for `behaviors.dtsi`, `keys.h`, `bt.h`, `mouse.h`.
- `corne.conf` uses Kconfig syntax.

## Power Management

Configured in `corne.conf` for wireless battery life:
- Idle after 30 s (`ZMK_IDLE_TIMEOUT=30000`)
- Deep sleep after 15 min (`ZMK_IDLE_SLEEP_TIMEOUT=900000`) — drops power ~100x
- BT TX power +8 dBm (`CONFIG_BT_CTLR_TX_PWR_PLUS_8`) for stronger split-half link
- USB logging is currently **on** (`CONFIG_ZMK_USB_LOGGING=y`) for TP tuning. Disable once tuning is done — logging keeps the controller awake and adds flash wear.
