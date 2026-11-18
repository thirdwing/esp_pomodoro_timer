# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

ESP-IDF firmware (C, LVGL) for the Waveshare ESP32-S3-Touch-AMOLED-1.8 (368x448 CO5300 AMOLED, QMI8658 IMU, ES8311 codec). A single-phase pomodoro timer: standing the device on a side edge picks the duration and starts it; laying it flat pauses (or returns to idle). All firmware lives in `main/main.c`.

## Build / flash

Requires ESP-IDF `>=5.5,<6.1` (see `main/idf_component.yml`). There are no tests or linters; verification is building and flashing to hardware.

```sh
. ~/esp/esp-idf/export.sh
idf.py set-target esp32s3        # once; regenerates sdkconfig from sdkconfig.defaults
idf.py build
ls /dev/cu.usbmodem*             # port name changes on each plug-in
idf.py -p /dev/cu.usbmodemXXXX flash monitor
```

- Board support comes from the managed component `waveshare/esp32_s3_touch_amoled_1_8` (downloaded into `managed_components/`, gitignored). The `bsp_*` functions and `BSP_*` pin macros come from there.
- `sdkconfig` is gitignored; persistent config changes (fonts, PSRAM, flash size, partition table) go in `sdkconfig.defaults`. Enabling a new LVGL font requires a `CONFIG_LV_FONT_*` line there. After editing it, delete `sdkconfig` (or run `idf.py fullclean`) so the defaults are re-applied.
- New ESP-IDF components used by `main.c` must be added to `REQUIRES` in `main/CMakeLists.txt`.

## Architecture (main/main.c)

- **No task/state machine object** — everything is driven by one LVGL timer, `update_ui`, firing every 250 ms under the BSP display lock. Each tick it polls the IMU, updates timer state, handles screen power, and redraws. Module-level `s_*` statics hold all state.
- **Timer state**: `s_duration` (minutes; `0` means idle), `s_remaining` (seconds banked while paused), `s_started_us` (esp_timer timestamp of last resume), `s_running`, `s_finished`. Live remaining time = `s_remaining - (now - s_started_us)`; `set_running(false)` folds elapsed time back into `s_remaining`. Changing to a different edge restarts with the new duration; lying flat after `TIME IS UP` or with no duration resets to idle.
- **Orientation**: `edge_minutes()` reads raw accel X/Y over I2C (±8g → 4096 LSB/g) and compares against `EDGE_G` (0.70 g). It returns the minutes for that edge (`MIN_TOP/RIGHT/BOTTOM/LEFT`, the single source of truth for durations and idle-screen labels) and the matching `LV_DISPLAY_ROTATION_*`; `0` means flat. Idle always forces rotation 0.
- **Audio**: `speaker_amp_off()` drives the amp-enable pin low first thing in `app_main` (avoids a power-on pop). `speaker_init()` sets up I2S + ES8311 but leaves the codec closed; `play_done()` opens it (which powers the amp), plays, then closes it. `play_done()` synthesizes a two-note chime and writes it **synchronously** from inside `update_ui`, so the UI stalls for ~1.4 s while it plays.
- **Screen power**: after `SCREEN_OFF_SEC` idle seconds, `screen_set(false)` sets AMOLED brightness to 0; any non-idle state or a screen tap (`wake_cb`) turns it back on. Drawing is skipped while off.
- **UI**: `build_ui()` creates all widgets once (all `lv_font_unscii_16`, retro pixel look); `update_ui` only toggles `LV_OBJ_FLAG_HIDDEN`, text, colors and alignment for idle / running / paused / finished views. Alignment calls are re-run every tick, so layouts stay correct after a rotation change.
- **Boot quirk**: `release_v2_panel_reset()` toggles IO-expander pins 0–2 to reset the display on V2 boards before `bsp_display_start()`; it's a no-op if the expander is absent.

## Enclosure

`enclosure/` is a git submodule (build123d CAD models, sparse-checked out to `pomodoro_timer_enclosure/`). It is not part of the firmware build; see README for init commands.
