# Sleep / Wake / Battery Drain Investigation

> Living document. Captures the state of the investigation into the
> "Corne won't wake from sleep after 20 min idle; battery lasts only ~2 days"
> regression on this config. Intended to bring a fresh Claude Code session
> back up to speed without re-deriving everything.

Last updated: 2026-05-28.

---

## 1. Symptoms

1. **Wake bug.** After ~15–20 minutes of inactivity on battery, the Corne
   becomes unresponsive to the host. Pressing matrix keys does nothing.
   Pressing the **physical reset button** on (one or both?) halves makes BT
   reconnect within a few seconds.
2. **Battery bug.** Battery life is ~2 days. Stock-Corne expectation on a
   100–300 mAh LiPo with working deep sleep is 1–3 weeks.

Both symptoms appeared (or became noticeable) after the TrackPoint + encoder
work landed. See § 4 for why they are likely the same underlying issue.

---

## 2. Hardware & software context

- **Boards:** two `nice_nano_v2` halves (nRF52840).
- **Shield:** custom `corne_tp` shield at
  `boards/shields/corne_tp/` — handwired Corne (4×6 + 3 thumbs per side) with
  one MEH01 mouse encoder per side and a Lenovo TrackPoint on the right half.
- **Right half is central** (inverted from stock Corne; see
  `boards/shields/corne_tp/Kconfig.defconfig`). TrackPoint and PS/2 driver
  live on the central side.
- **ZMK fork:** `infused-kim/zmk@pr-testing/mouse_ps2_module_base`,
  pinned in `config/west.yml`. Snapshot of upstream ZMK around
  April 2024 (Zephyr v3.5.0+zmk-fixes).
- **PS/2 driver:** `infused-kim/kb_zmk_ps2_mouse_trackpoint_driver@main`.
- **`zmk-tri-state`:** pinned to commit `9e44394` (post-`9e44394` adds
  `binding_event.source` which the fork doesn't have — see comment in
  `config/west.yml`).
- **Encoder:** MEH01 (Kailh 5 mm mouse encoder). Important: 2 quadrature
  transitions per detent, not 4 like a tactile EC11; `steps=<40>` is what
  matches that hardware empirically (see `corne_tp.dtsi` comment).
- **Power-relevant configs in `config/corne_tp.conf`:**
  - `CONFIG_ZMK_SLEEP=y`
  - `CONFIG_ZMK_IDLE_TIMEOUT=30000` (30 s → idle state)
  - `CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=900000` (15 min → deep sleep attempt)
  - `CONFIG_BT_CTLR_TX_PWR_PLUS_8=y` (radio ~8 dB hotter; deliberate)
  - `CONFIG_ZMK_USB_LOGGING=y` (kept on for TP tuning; keeps controller
    awake **when USB is plugged in only** — irrelevant on battery)

---

## 3. Branch and commit ledger

Created during this investigation:

| Branch | Base | Purpose | Status |
|---|---|---|---|
| `experiment/disable-ps2` | `main` (`6707afa`) | Disable PS/2 stack and (later) interrupt-priority remap | Pushed; tested |
| `baseline/pre-tp` | `eeb3100` (commit before first TP commit `1ca756a`) | Stock Corne config — verify whether wake worked before TP work | Pushed; tested |

Key commits to know:

- `6707afa` — current `main` HEAD.
- `1ca756a` — `feat: add TrackPoint and per-side encoders on a local corne_tp shield`.
  **First TP-related commit.** Everything before this is "pre-TP".
- `eeb3100` — parent of `1ca756a`; tip of the pre-TP era.
- `72e36c5` — `add deep sleep, conditional layer 3, replace NON_US_BSLH`.
  **First commit that enables `CONFIG_ZMK_SLEEP=y`.** Earlier than the TP
  work. So deep sleep was nominally enabled for a window with the stock
  Corne shield, but the user did not specifically test 15-min-battery-idle
  wake in that window.
- `574819b` (on `experiment/disable-ps2`) — `chore(tp): gate PS/2 stack
  behind EXPERIMENT_NO_PS2 flag`. Experiment 2.
- `208db65` (on `experiment/disable-ps2`) — `chore(tp): gate
  interrupt-priority remap behind #if 0`. Experiment 3.

---

## 4. What we tested, what we learned

### Experiment 2 — disable PS/2 stack (commit `574819b`)

**Change:** `#define EXPERIMENT_NO_PS2` in `config/corne_tp.keymap` gates
`config/include/mouse_tp.dtsi` so `mouse_ps2`, `mouse_ps2_input_listener`,
`uart_ps2`, and `uart0` all remain `status="disabled"`. PS/2 driver C source
isn't compiled. `U_MSS_*` layer-6 keys fall through to `&none`.

**Result:** Wake from 20-min idle still fails. PS/2 is **not** the cause
on its own.

### Experiment 3 — also revert interrupt-priority remap (commit `208db65`)

**Change:** wrapped the priority-override block in
`boards/shields/corne_tp/corne_tp_right.overlay:73-119` in `#if 0`. With PS/2
out, GPIOTE@0 and the SoftDevice-critical RADIO/RTC0/TIMER0 demotions
serve no purpose anyway.

**Result:** Wake still fails. The priority remap is **not** the cause.

### Baseline test — pre-TP commit (`baseline/pre-tp` from `eeb3100`)

**Change:** stock Corne shield, left=central, RGB underglow + WS2812 strip
enabled, no encoders, no kscan_composite, no PS/2 — but
**`CONFIG_ZMK_SLEEP=y` and the same 15-min timeout are present**.

**Result (user-reported):** After ~15 min idle, keyboard is unresponsive
to the host. **Pressing keys on both halves wakes it back up in 1–2 s.**
The user described this as "regular deep sleep behaviour."

This is the key finding. See § 5.

---

## 5. Current leading theory

**Baseline does *not* actually reach System OFF.** Pre-TP it goes idle but
the chip never powers down. Current build *does* reach System OFF, and
that's what breaks wake.

Reasoning:

1. The matrix kscan driver on this fork is structurally incapable of
   waking the chip from System OFF. Confirmed by reading
   `.zmk/zmk/app/module/drivers/kscan/kscan_gpio_matrix.c`:

   ```c
   static int kscan_matrix_pm_action(const struct device *dev, ...) {
       case PM_DEVICE_ACTION_SUSPEND:
           kscan_matrix_disconnect_inputs(dev);    // GPIO_DISCONNECTED on every row
           kscan_matrix_disconnect_outputs(dev);   // GPIO_DISCONNECTED on every col
           return kscan_matrix_disable(dev);
       ...
   }
   ```

   `gpio_pin_configure_dt(gpio, GPIO_DISCONNECTED)` sets the nRF GPIO's
   `INPUT` field to Disconnect. With INPUT disconnected, SENSE cannot
   detect transitions, and SENSE is the **only** mechanism that wakes
   nRF52 from System OFF. Confirmed: `grep -rn "nrf_gpio_cfg_sense"
   .zmk/zmk/app/module/drivers/kscan/` returns nothing. `wakeup-source`
   in DT is declared but **ignored by the driver's PM action**.

   Same situation for `kscan_gpio_direct.c` (encoder clicks).

2. If wake from System OFF is impossible on this driver, then the
   baseline "wake works" observation can only be explained by the
   baseline **not entering System OFF**. The wake timing (1–2 s) and the
   "press both halves" requirement both fit normal-idle wake, not
   System-OFF wake (which is a full chip reset → 5–10 s, treated by the
   host as a fresh pairing).

3. The deep-sleep entry path in `.zmk/zmk/app/src/activity.c:75-92`:

   ```c
   if (inactive_time > MAX_SLEEP_MS && !is_usb_power_present()) {
       set_state(ZMK_ACTIVITY_SLEEP);
       if (zmk_pm_suspend_devices() < 0) {       // <-- failure here = no sys_poweroff
           LOG_ERR("Failed to suspend all the devices");
           zmk_pm_resume_devices();
           return;                                // bail out → stay in normal idle
       }
       sys_poweroff();
   }
   ```

   If `zmk_pm_suspend_devices()` returns < 0, `sys_poweroff()` is
   skipped. The chip stays in normal Zephyr idle, where matrix presses
   wake the CPU via standard GPIO interrupts.

4. The most plausible reason baseline fails to suspend cleanly:
   **RGB underglow + WS2812 strip** is enabled in baseline `corne.conf`
   but not in current `corne_tp.conf` (P0.06 was repurposed as PS/2 SCL,
   so the strip had to go). The WS2812 driver / SPI3 backing it is the
   classic culprit for returning errors from `PM_DEVICE_ACTION_SUSPEND`,
   which would bail out `zmk_pm_suspend_devices()` exactly as needed.
   **This is unverified speculation** but fits the pattern.

5. The ~2-day battery on the current build is the normal-idle drain rate
   for nRF52 + BT-connected + radio at +8 dBm. The user has been *not*
   in System OFF much of the time even on the broken build — the
   keyboard is in active or shallow-idle states until 15 min idle, then
   it enters System OFF and stays there until reset.

Falsified along the way:

| Earlier hypothesis | Status |
|---|---|
| PS/2 driver's missing PM hooks cause `zmk_pm_suspend_devices()` to fail | Falsified by experiment 2 |
| Interrupt-priority remap breaks BT recovery on wake | Falsified by experiment 3 |
| Matrix kscan can't arm SENSE → no wake from System OFF | Still believed true — but the relevant question is whether System OFF is being entered |

---

## 6. Open questions

1. **Does the current build actually reach `sys_poweroff()`?** This is the
   pivot of the theory. Can be verified two ways:
   - Add RTT logging (USB logging is useless here — USB plugged = no deep
     sleep). Look for the `"Failed to suspend all the devices"` log line.
     Absence of that line + 15 min idle + no wake = chip is in System
     OFF.
   - Or proxy via § 7 Experiment 4.
2. **Which device on baseline was failing suspend** (if any)? RGB/WS2812
   is the leading candidate but unconfirmed. Could also be
   `display_dev` (if OLED was ever in the picture), or the BT controller
   itself.
3. **Has upstream ZMK added wake-on-matrix support since this fork
   snapshot?** Probably yes (the upstream PM rework around mid-2024
   added proper wakeup-source handling for kscan). Not verified.

---

## 7. Next experiments (planned, not yet done)

### Experiment 4 — disable deep sleep on `experiment/disable-ps2`

Set `CONFIG_ZMK_SLEEP=n` in `config/corne_tp.conf`. Re-flash both halves.

**Predictions if theory in § 5 is right:**
- Wake behavior becomes indistinguishable from the baseline: unresponsive
  after 15 min idle, but presses on both halves recover in 1–2 s.
- Battery life roughly the same as today (~2 days) — because we weren't
  actually getting deep-sleep savings before either.

**If wake still fails after this:** theory is wrong. Look further.

### Experiment 5 (only if § 7 confirms) — re-enable underglow

Re-enable `CONFIG_ZMK_RGB_UNDERGLOW=y` + `CONFIG_WS2812_STRIP=y` on the
current build (even with no LEDs wired). Caveat: `&spi3` is disabled in
`boards/shields/corne_tp/boards/nice_nano_v2.overlay` because P0.06 is
the PS/2 SCL pin — so the strip might fail at init rather than at
suspend, making this test inconclusive.

### Option C — actual fix, if we want real deep sleep

Patch `kscan_gpio_matrix.c` (and `kscan_gpio_direct.c`) on the fork to
arm SENSE on suspend when `wakeup-source` is set in DT. Approximately:

```c
case PM_DEVICE_ACTION_SUSPEND:
    if (pm_device_wakeup_is_enabled(dev)) {
        // For each input pin: configure SENSE high (col2row matrix) or
        // SENSE low (row2col) so any key press triggers DETECT.
        // Outputs may need to be driven actively (rather than
        // GPIO_DISCONNECTED) so the SENSE on inputs sees a transition.
    } else {
        kscan_matrix_disconnect_inputs(dev);
        kscan_matrix_disconnect_outputs(dev);
    }
    return kscan_matrix_disable(dev);
```

Note for col2row matrices (this config): on suspend, every column needs
to be driven *active* (1 for active-high) so that pressing any key pulls
the input row to active state, which SENSE detects. This is how upstream
ZMK's newer wake-on-matrix works.

Invasive — would carry as a vendor patch via west manifest or a small
out-of-tree module. Probably the right long-term answer only if the user
wants to stay on this fork.

### Long-term fix

Migrate off `infused-kim/zmk@pr-testing/mouse_ps2_module_base` when:
- Upstream ZMK's pointing-device PR merges, OR
- A newer fork with both PS/2 support and modern kscan wake-up support
  becomes available.

---

## 8. Important file references

### In this repo

- `config/corne_tp.conf` — power configs.
- `config/corne_tp.keymap` — `#define EXPERIMENT_NO_PS2` lives at line ~50
  on `experiment/disable-ps2`.
- `config/include/mouse_tp.dtsi` — gated TP-related DT overrides.
- `config/west.yml` — module pins, including the critical
  `zmk-tri-state` revision pin.
- `boards/shields/corne_tp/corne_tp.dtsi` — `kscan0`, `kscan_encoder_clicks`,
  `kscan_composite`, encoders. Note `wakeup-source` is set on the kscans
  but **not on `kscan_composite`** (this may or may not matter).
- `boards/shields/corne_tp/corne_tp_right.overlay` — PS/2 hardware,
  interrupt-priority remap, central role. `#if 0` block on
  `experiment/disable-ps2`.
- `boards/shields/corne_tp/boards/nice_nano_v2.overlay` — disables `&spi3`
  (since P0.06 is the PS/2 SCL pin).
- `CLAUDE.md` (repo root) — has more hardware context. Read if context is
  thin.
- `new_log*.txt` (untracked) — user-captured serial logs. Note that
  `new_log_3.txt` shows USB enumeration at the start, meaning the device
  was on USB — `is_usb_power_present()` returns true, deep sleep is
  skipped, logs end before the 15-min mark. **Logs from USB sessions
  cannot show the deep-sleep transition.**

### In the ZMK fork checkout (`.zmk/zmk/`)

Available locally after a west build runs at least once; otherwise fetch
fresh.

- `app/src/activity.c:75-92` — idle/sleep state machine, `sys_poweroff()`
  call site.
- `app/src/pm.c:30-65` — `zmk_pm_suspend_devices()`. Drivers with NULL PM
  hooks return `-ENOSYS` and are silently skipped (lines 50–53). A
  non-recoverable error (anything else < 0) bails out the whole loop.
- `app/src/pm.c:90-135` — `zmk_pm_soft_off()` and `HAS_WAKERS` /
  `zmk_soft_off_wakeup_sources` — a **separate**, user-triggered soft-off
  feature. Not what's happening on idle-driven sleep, but the only path
  in this fork that explicitly arms wake sources.
- `app/src/physical_layouts.c:472-481` — at init, enables
  `pm_device_wakeup_enable(pl->kscan, true)` on layout-attached kscans.
  No `physical-layout` node in this config = this loop is empty.
- `app/module/drivers/kscan/kscan_gpio_matrix.c:472-486` — PM action.
  `GPIO_DISCONNECTED` on suspend, no SENSE arming.
- `app/module/drivers/kscan/kscan_gpio_direct.c:358-371` — same pattern.
- `app/module/drivers/kscan/kscan_composite.c:158-168` — composite PM
  action cascades to children.

### In the PS/2 driver repo (`infused-kim/kb_zmk_ps2_mouse_trackpoint_driver`)

Not checked out locally; fetch with `gh api repos/.../contents/<path>
--jq '.content' | base64 -d` as needed.

- `src/drivers/ps2/ps2_uart.c:1360` — `DEVICE_DT_INST_DEFINE(... NULL ...)`,
  no PM action callback.
- `src/drivers/ps2/ps2_uart.c:331,352` — `PINCTRL_STATE_DEFAULT` /
  `PINCTRL_STATE_SLEEP` are **repurposed** by the driver for RX/TX modes.
  The Zephyr "sleep" pinctrl state is hijacked for runtime TX mode, not
  for system PM sleep.
- `src/drivers/input/input_mouse_ps2.c:1854` — same NULL-PM pattern.
- `src/drivers/input/input_mouse_ps2.c:1622-1640` — `zmk_mouse_ps2_init_thread`
  spawns a polling thread on cold boot. `MOUSE_PS2_INIT_ATTEMPTS = 10`,
  `MOUSE_PS2_POWER_ON_RESET_TIME = 600 ms`, `ps2_read` timeout 2 s →
  worst-case ~70 s init time. Runs at `K_PRIO_COOP(10)`. Does **not**
  block BT init (separate threads).

---

## 9. How to continue

If you're a fresh Claude Code session reading this:

1. **Read this whole document first.** Especially § 5 (current theory).
2. **Check what branch is currently checked out**
   (`git -C /home/theo/projects/zmk-config-corne branch --show-current`)
   and what the user wants to test next.
3. **Don't redo experiments 2 or 3** — they're committed on
   `experiment/disable-ps2` and the user has already tested them.
4. **The likely next action** is Experiment 4 (set `CONFIG_ZMK_SLEEP=n`
   in `corne_tp.conf` on the `experiment/disable-ps2` branch, push,
   user tests). The intended diagnostic value is in § 7.
5. **Don't claim that fixing PS/2 PM hooks will solve the wake bug.**
   Experiment 2 falsified that, but it took a while to figure out.
6. **The matrix kscan driver structurally can't wake from System OFF on
   this fork.** That part of the theory is still believed true. The open
   question is whether System OFF is being entered at all, and that
   varies with which devices are enabled.
7. **Practical advice to the user, if they ask for an immediate fix:**
   set `CONFIG_ZMK_SLEEP=n`. Accept the ~2-day battery life. The 2 days
   is what nRF52 + BT-connected idle costs on this fork; deep sleep was
   never actually working for them. The real fix is § 7 Option C
   (kscan patch) or the long-term migration.
8. **CI:** every push to any branch builds artifacts via
   `.github/workflows/build.yml` → `zmkfirmware/zmk` build-user-config
   workflow. Build matrix: `nice_nano_v2 + corne_tp_left` / `corne_tp_right`
   on TP branches, `corne_left` / `corne_right` on `baseline/pre-tp`.
9. **`gh` CLI is authenticated as `tbr-wa`** but the fork is at
   `bauertheo/zmk-config-corne` (personal account). User pushes manually.
10. **User wants you to commit changes without the `Co-Authored-By:
    Claude` trailer** — the auto-mode classifier blocks it as
    impersonation. Just use a plain conventional-commits message.

---

## 10. Appendix — quick verification commands

```bash
# Check current branch and recent commits
git -C /home/theo/projects/zmk-config-corne log --oneline -10

# Inspect the kscan PM action that probably can't wake from System OFF
grep -nE "PM_DEVICE_ACTION_SUSPEND|GPIO_DISCONNECTED|nrf_gpio_cfg_sense" \
  /home/theo/projects/zmk-config-corne/.zmk/zmk/app/module/drivers/kscan/kscan_gpio_matrix.c

# Inspect the sleep entry path
sed -n '70,95p' /home/theo/projects/zmk-config-corne/.zmk/zmk/app/src/activity.c

# Inspect zmk_pm_suspend_devices and the silent-skip for NULL-PM drivers
sed -n '30,75p' /home/theo/projects/zmk-config-corne/.zmk/zmk/app/src/pm.c

# Fetch a fresh copy of the PS/2 driver source if not checked out locally
gh api repos/infused-kim/kb_zmk_ps2_mouse_trackpoint_driver/contents/src/drivers/ps2/ps2_uart.c \
  --jq '.content' | base64 -d > /tmp/ps2_uart.c
```
