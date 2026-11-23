# 2.8" ESP32-2432S028R Board Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Run the unchanged cyberviber UI on the 2.8" ESP32-2432S028R ("CYD") by adding a second board target, `cyd28r`, next to the existing 2.4" `cyd24r`.

**Architecture:** Every board difference moves out of `firmware/src/board.h` into a new LovyanGFX-free header, `firmware/src/board_config.h`, selected by the build flag `-DCV_BOARD_CYD28R`. `board.h` builds the LovyanGFX device from those values, so the UI, layouts, protocol and daemon stay untouched. The values that can only be learned from the physical panel (driver, rotation, colour order, inversion, read-back order) are macros with defaults, overridable from the shell with `PLATFORMIO_BUILD_FLAGS`, so bring-up is a flash-and-look loop with no code edits until the right values are baked in.

**Tech Stack:** PlatformIO (`espressif32@6.10.0`, Arduino), LovyanGFX 1.2.7, LVGL 9.2.2, host `c++` (Xcode command line tools) for the compile-time pin check.

**Spec:** No separate spec document. Requirements come from the build guide at https://cyberviber.de/#build, `README.md` (Firmware section) and the board pinout at https://github.com/witnessmenow/ESP32-Cheap-Yellow-Display/blob/main/PINS.md.

## Global Constraints

- The 2.4" `cyd24r` build must keep its current values exactly: rotation 5, panel 320×240, BGR, no inversion, backlight GPIO 27, touch sharing HSPI (SCLK 14, MOSI 13, MISO 12), touch CS 33, IRQ 36, shifted read-back.
- 2.8" pins (from PINS.md): TFT on HSPI with SCLK 14, MOSI 13, MISO 12, DC 2, CS 15; backlight GPIO 21; XPT2046 touch on its own pins, CLK 25, MOSI 32, MISO 39, CS 33, IRQ 36.
- The UI renders 240×320 portrait only (`W = 240, H = 320` in `firmware/src/app.h`). Do not change layouts or the serial protocol.
- `default_envs` stays `cyd24r`; the 2.8" is always built with `-e cyd28r`.
- Commit style follows the repo: gitmoji, scope in parentheses, sentence case (`✨ (firmware): Add the 2.8" board target`). No `Co-Authored-By` trailers.
- Work on a local branch `cyd28r` in `~/Documents/code/cyberviber`. The remote is upstream (`d135-1r43/cyberviber`); do not push to it.
- Power and flash the board from a Mac with a USB-C-to-micro-USB cable in the micro-USB port, or the kit's USB-A cable through a USB-C-to-USB-A adapter. A USB-C-to-USB-C cable into the board's USB-C port gives it no power.

## Review Focus

1. **2.4" regression:** someone with the original board pulls this branch and flashes the default env; it must behave exactly as before. Pinned by the host check's 2.4" branch (Task 1) and a `cyd24r` build in every task.
2. **USB port at the bottom:** after flipping orientation, taps must land where the finger is, not mirrored. Pinned by Task 4, Step 7.
3. **Dimmed backlight on GPIO 21:** the idle dim and the brightness setting must fade GPIO 21, not leave it at full or off. Pinned by Task 4, Step 5.
4. **Re-calibration at boot:** holding a finger on the screen while plugging in must rerun calibration on the 2.8" as it does on the 2.4". Pinned by Task 4, Step 6.
5. **Screenshot colours:** `cyberviber screenshot` must produce correct colours, since the 2.4" read-back byte shift is panel-specific. Pinned by Task 5, Step 4.

---

## File Structure

| File | Change | Responsibility |
|---|---|---|
| `firmware/src/board_config.h` | Create | Per-board pins and panel settings as `constexpr` values, no LovyanGFX, host-compilable |
| `firmware/hostcheck/board_config_check.cpp` | Create | `static_assert`s pinning both boards' values; compiling it is the test |
| `firmware/src/board.h` | Modify (rewrite) | Builds the LovyanGFX `Board` from `board_config.h` |
| `firmware/src/main.cpp:153-161` | Modify | Screenshot pixel unpacking follows `board::READBACK_SHIFTED` |
| `firmware/platformio.ini` | Modify | Adds `cyd28r` and `cyd28r-touchlog` envs |
| `README.md:74-86` | Modify | Documents the 2.8" target and how to tune it |

The host check lives in `hostcheck/`, not `test/`, because PlatformIO treats `test/` as its own unit-test tree.

---

### Task 1: Move board values into `board_config.h` without changing the 2.4" build

**Files:**
- Create: `firmware/src/board_config.h`
- Create: `firmware/hostcheck/board_config_check.cpp`
- Modify: `firmware/src/board.h` (full rewrite)
- Modify: `firmware/src/main.cpp:153-161`

**Interfaces:**
- Produces: namespace `board` with `constexpr int TFT_SCLK, TFT_MOSI, TFT_MISO, TFT_DC, TFT_CS, BL, TOUCH_SCLK, TOUCH_MOSI, TOUCH_MISO, TOUCH_CS, TOUCH_IRQ, PANEL_W, PANEL_H, ROTATION`, `constexpr bool TOUCH_OWN_BUS, PANEL_BGR, PANEL_INVERT, READBACK_SHIFTED`; macros `CV_PANEL_ST7789` (0/1) and optional `CV_TOUCH_SOFT_SPI`. `board.h` keeps exporting `BOARD_ROTATION` and `class Board`, which `main.cpp` already uses.

- [ ] **Step 1: Set up the branch and toolchain**

```bash
cd ~/Documents/code/cyberviber
git switch -c cyd28r
uv tool install platformio        # or: pipx install platformio
pio --version
c++ --version
```

Expected: PlatformIO prints a version; `c++` prints Apple clang.

- [ ] **Step 2: Record a baseline 2.4" build**

```bash
pio run -d firmware -e cyd24r 2>&1 | tail -5
```

Expected: `SUCCESS`. The first run downloads the toolchain and libraries, which takes several minutes.

- [ ] **Step 3: Write the failing host check**

Create `firmware/hostcheck/board_config_check.cpp`:

```cpp
// Compile-time check of the per-board values. Compiling is the test:
//   c++ -std=c++17 -fsyntax-only -I firmware/src firmware/hostcheck/board_config_check.cpp
//   c++ -std=c++17 -fsyntax-only -DCV_BOARD_CYD28R -I firmware/src firmware/hostcheck/board_config_check.cpp
#include "board_config.h"

static_assert(board::TFT_SCLK == 14 && board::TFT_MOSI == 13 && board::TFT_MISO == 12, "TFT bus");
static_assert(board::TFT_DC == 2 && board::TFT_CS == 15, "TFT control pins");
static_assert(board::TOUCH_CS == 33 && board::TOUCH_IRQ == 36, "touch CS and IRQ");

#if defined(CV_BOARD_CYD28R)
static_assert(board::BL == 21, "2.8\": backlight on GPIO 21");
static_assert(board::TOUCH_OWN_BUS, "2.8\": XPT2046 has its own pins");
static_assert(board::TOUCH_SCLK == 25 && board::TOUCH_MOSI == 32 && board::TOUCH_MISO == 39, "2.8\": touch pins");
static_assert(board::PANEL_W == 240 && board::PANEL_H == 320, "2.8\": portrait panel memory");
#else
static_assert(board::BL == 27, "2.4\": backlight on GPIO 27");
static_assert(!board::TOUCH_OWN_BUS, "2.4\": touch shares HSPI");
static_assert(board::TOUCH_SCLK == 14 && board::TOUCH_MOSI == 13 && board::TOUCH_MISO == 12, "2.4\": touch pins");
static_assert(board::PANEL_W == 320 && board::PANEL_H == 240, "2.4\": landscape panel memory");
static_assert(board::ROTATION == 5, "2.4\": rotation 5");
static_assert(board::PANEL_BGR && !board::PANEL_INVERT && board::READBACK_SHIFTED, "2.4\": panel settings");
static_assert(CV_PANEL_ST7789 == 0, "2.4\": ILI9341");
#endif

int main() { return 0; }
```

- [ ] **Step 4: Run it and see it fail**

```bash
c++ -std=c++17 -fsyntax-only -I firmware/src firmware/hostcheck/board_config_check.cpp
```

Expected: FAIL with `'board_config.h' file not found`.

- [ ] **Step 5: Create `firmware/src/board_config.h` with the 2.4" values only**

```cpp
// Per-board pins and panel settings, kept free of LovyanGFX so the host can check them
// (firmware/hostcheck/board_config_check.cpp). The 2.4" ESP32-2432S024R is the default;
// -DCV_BOARD_CYD28R selects the 2.8" ESP32-2432S028R.
#pragma once

namespace board {

constexpr int TFT_SCLK = 14, TFT_MOSI = 13, TFT_MISO = 12, TFT_DC = 2, TFT_CS = 15;
constexpr int TOUCH_CS = 33, TOUCH_IRQ = 36;

// ESP32-2432S024R: TFT and XPT2046 touch share the HSPI bus.
// Verified on the unit: the controller's memory is 320×240 (landscape) with BGR colour order;
// rotation 5 shows it upright as 240×320 portrait with the USB port at the top. Rotation 7 (the
// same turned by 180°) garbles the picture on this panel, so "USB at the bottom" stays in rotation 5
// and turns pixels and touch in software (main.cpp).
#ifndef CV_ROTATION
#define CV_ROTATION 5
#endif
#ifndef CV_PANEL_BGR
#define CV_PANEL_BGR 1
#endif
#ifndef CV_PANEL_INVERT
#define CV_PANEL_INVERT 0
#endif
#ifndef CV_READBACK_SHIFTED
#define CV_READBACK_SHIFTED 1  // read-back arrives as (b, r, g) instead of (r, g, b)
#endif
constexpr int BL = 27;
constexpr bool TOUCH_OWN_BUS = false;
constexpr int TOUCH_SCLK = 14, TOUCH_MOSI = 13, TOUCH_MISO = 12;
constexpr int PANEL_W = 320, PANEL_H = 240;

#ifndef CV_PANEL_ST7789
#define CV_PANEL_ST7789 0  // ILI9341 unless the board branch above says otherwise
#endif

constexpr int ROTATION = CV_ROTATION;
constexpr bool PANEL_BGR = CV_PANEL_BGR, PANEL_INVERT = CV_PANEL_INVERT;
constexpr bool READBACK_SHIFTED = CV_READBACK_SHIFTED;

}  // namespace board
```

- [ ] **Step 6: Run the host check and see it pass**

```bash
c++ -std=c++17 -fsyntax-only -I firmware/src firmware/hostcheck/board_config_check.cpp && echo PASS
```

Expected: `PASS`.

- [ ] **Step 7: Rewrite `firmware/src/board.h` to read from `board_config.h`**

```cpp
// The LovyanGFX device for whichever board board_config.h selects.
#pragma once

#define LGFX_USE_V1
#include <LovyanGFX.hpp>

#include "board_config.h"

static constexpr int BOARD_ROTATION = board::ROTATION;

// Rough raw XPT2046 range; replaced by the calibration stored in NVS (see main.cpp).
#define CV_TOUCH_X_MIN 300
#define CV_TOUCH_X_MAX 3800
#define CV_TOUCH_Y_MIN 250
#define CV_TOUCH_Y_MAX 3800

class Board : public lgfx::LGFX_Device {
#if CV_PANEL_ST7789
  lgfx::Panel_ST7789 panel_;
#else
  lgfx::Panel_ILI9341 panel_;
#endif
  lgfx::Bus_SPI bus_;
  lgfx::Light_PWM light_;
  lgfx::Touch_XPT2046 touch_;

 public:
  Board() {
    {
      auto c = bus_.config();
      c.spi_host = HSPI_HOST;
      c.spi_mode = 0;
      c.freq_write = 40000000;
      c.freq_read = 16000000;
      c.spi_3wire = false;
      c.use_lock = true;
      c.dma_channel = SPI_DMA_CH_AUTO;
      c.pin_sclk = board::TFT_SCLK;
      c.pin_mosi = board::TFT_MOSI;
      c.pin_miso = board::TFT_MISO;
      c.pin_dc = board::TFT_DC;
      bus_.config(c);
      panel_.setBus(&bus_);
    }
    {
      auto c = panel_.config();
      c.pin_cs = board::TFT_CS;
      c.pin_rst = -1;  // tied to EN
      c.pin_busy = -1;
      c.panel_width = board::PANEL_W;
      c.panel_height = board::PANEL_H;
      c.offset_rotation = 0;
      c.readable = true;
      c.invert = board::PANEL_INVERT;
      c.rgb_order = board::PANEL_BGR;
      c.dlen_16bit = false;
      c.bus_shared = !board::TOUCH_OWN_BUS;
      panel_.config(c);
    }
    {
      auto c = light_.config();
      c.pin_bl = board::BL;
      c.invert = false;
      c.freq = 12000;
      c.pwm_channel = 7;
      light_.config(c);
      panel_.setLight(&light_);
    }
    {
      auto c = touch_.config();
      c.x_min = CV_TOUCH_X_MIN;
      c.x_max = CV_TOUCH_X_MAX;
      c.y_min = CV_TOUCH_Y_MIN;
      c.y_max = CV_TOUCH_Y_MAX;
      c.pin_int = board::TOUCH_IRQ;
      c.bus_shared = !board::TOUCH_OWN_BUS;
      c.offset_rotation = 0;
#ifdef CV_TOUCH_SOFT_SPI
      c.spi_host = -1;  // bit-banged, for boards where the VSPI peripheral does not reach the touch pins
#else
      c.spi_host = board::TOUCH_OWN_BUS ? VSPI_HOST : HSPI_HOST;
#endif
      c.freq = 1000000;
      c.pin_sclk = board::TOUCH_SCLK;
      c.pin_mosi = board::TOUCH_MOSI;
      c.pin_miso = board::TOUCH_MISO;
      c.pin_cs = board::TOUCH_CS;
      touch_.config(c);
      panel_.setTouch(&touch_);
    }
    setPanel(&panel_);
  }
};
```

- [ ] **Step 8: Make the screenshot unpacking follow the board**

In `firmware/src/main.cpp`, delete the comment line `// This panel's read-back is shifted by one byte: what arrives as (r, g, b) is really (b, r, g).` and replace the pixel loop:

```cpp
    for (int i = 0; i < W * N; i++) px[i] = (rgb[i].g >> 3) << 11 | (rgb[i].b >> 2) << 5 | rgb[i].r >> 3;
```

with:

```cpp
    for (int i = 0; i < W * N; i++)
      px[i] = board::READBACK_SHIFTED  // see board_config.h
                  ? (rgb[i].g >> 3) << 11 | (rgb[i].b >> 2) << 5 | rgb[i].r >> 3
                  : (rgb[i].r >> 3) << 11 | (rgb[i].g >> 2) << 5 | rgb[i].b >> 3;
```

- [ ] **Step 9: Build the 2.4" target**

```bash
pio run -d firmware -e cyd24r 2>&1 | tail -5
```

Expected: `SUCCESS`. Values are identical to the old `board.h`, which the host check from Step 6 pins.

- [ ] **Step 10: Commit**

```bash
git add firmware/src/board_config.h firmware/src/board.h firmware/src/main.cpp firmware/hostcheck/board_config_check.cpp
git commit -m "♻️ (firmware): Move board pins and panel settings into board_config.h"
```

---

### Task 2: Add the 2.8" board target

**Files:**
- Modify: `firmware/src/board_config.h`
- Modify: `firmware/platformio.ini`

**Interfaces:**
- Consumes: the `board` namespace and macros from Task 1.
- Produces: build envs `cyd28r` and `cyd28r-touchlog`; tuning macros `CV_ROTATION`, `CV_PANEL_BGR`, `CV_PANEL_INVERT`, `CV_PANEL_ST7789`, `CV_READBACK_SHIFTED`, `CV_TOUCH_SOFT_SPI`, settable through `PLATFORMIO_BUILD_FLAGS`.

- [ ] **Step 1: Run the host check for the 2.8" and see it fail**

```bash
c++ -std=c++17 -fsyntax-only -DCV_BOARD_CYD28R -I firmware/src firmware/hostcheck/board_config_check.cpp
```

Expected: FAIL on `2.8": backlight on GPIO 21`.

- [ ] **Step 2: Add the 2.8" branch to `board_config.h`**

Replace everything from `// ESP32-2432S024R: TFT and XPT2046 touch share the HSPI bus.` through `constexpr int PANEL_W = 320, PANEL_H = 240;` (leave the `CV_PANEL_ST7789` default and the `constexpr` lines after it in place) with:

```cpp
#if defined(CV_BOARD_CYD28R)
// ESP32-2432S028R: TFT on HSPI, XPT2046 touch on its own pins (VSPI), backlight on GPIO 21.
// The panel's memory is 240×320 (portrait). Rotation, colour order, inversion and read-back are
// the values found on the unit in Task 3 of docs/superpowers/plans/2026-11-23-cyd28r-board-support.md.
#ifndef CV_ROTATION
#define CV_ROTATION 0
#endif
#ifndef CV_PANEL_BGR
#define CV_PANEL_BGR 1
#endif
#ifndef CV_PANEL_INVERT
#define CV_PANEL_INVERT 0
#endif
#ifndef CV_READBACK_SHIFTED
#define CV_READBACK_SHIFTED 0
#endif
constexpr int BL = 21;
constexpr bool TOUCH_OWN_BUS = true;
constexpr int TOUCH_SCLK = 25, TOUCH_MOSI = 32, TOUCH_MISO = 39;
constexpr int PANEL_W = 240, PANEL_H = 320;
#else
// ESP32-2432S024R: TFT and XPT2046 touch share the HSPI bus.
// Verified on the unit: the controller's memory is 320×240 (landscape) with BGR colour order;
// rotation 5 shows it upright as 240×320 portrait with the USB port at the top. Rotation 7 (the
// same turned by 180°) garbles the picture on this panel, so "USB at the bottom" stays in rotation 5
// and turns pixels and touch in software (main.cpp).
#ifndef CV_ROTATION
#define CV_ROTATION 5
#endif
#ifndef CV_PANEL_BGR
#define CV_PANEL_BGR 1
#endif
#ifndef CV_PANEL_INVERT
#define CV_PANEL_INVERT 0
#endif
#ifndef CV_READBACK_SHIFTED
#define CV_READBACK_SHIFTED 1  // read-back arrives as (b, r, g) instead of (r, g, b)
#endif
constexpr int BL = 27;
constexpr bool TOUCH_OWN_BUS = false;
constexpr int TOUCH_SCLK = 14, TOUCH_MOSI = 13, TOUCH_MISO = 12;
constexpr int PANEL_W = 320, PANEL_H = 240;
#endif
```
- [ ] **Step 3: Run both host checks and see them pass**

```bash
c++ -std=c++17 -fsyntax-only -I firmware/src firmware/hostcheck/board_config_check.cpp && echo PASS-24
c++ -std=c++17 -fsyntax-only -DCV_BOARD_CYD28R -I firmware/src firmware/hostcheck/board_config_check.cpp && echo PASS-28
```

Expected: `PASS-24` and `PASS-28`.

- [ ] **Step 4: Add the envs to `firmware/platformio.ini`**

Replace the header comment lines

```ini
; cyberviber firmware for the ESP32-2432S024R (ILI9341 240×320, XPT2046 resistive touch)
; Flash:  cyberviber release && pio run -t upload
```

with

```ini
; cyberviber firmware. Default: ESP32-2432S024R (2.4"). The 2.8" ESP32-2432S028R builds with -e cyd28r.
; Flash:  cyberviber release && pio run -t upload            (2.4")
;         cyberviber release && pio run -e cyd28r -t upload  (2.8")
```

and append at the end of the file:

```ini

; ESP32-2432S028R (2.8"): backlight on GPIO 21, touch on its own pins; see src/board_config.h
[env:cyd28r]
extends = env:cyd24r
build_flags = ${env:cyd24r.build_flags} -DCV_BOARD_CYD28R

[env:cyd28r-touchlog]
extends = env:cyd28r
build_flags = ${env:cyd28r.build_flags} -DCV_TOUCH_LOG
```

- [ ] **Step 5: Build all four envs**

```bash
for e in cyd24r cyd24r-touchlog cyd28r cyd28r-touchlog; do pio run -d firmware -e $e >/dev/null 2>&1 && echo "$e ok" || echo "$e FAILED"; done
PLATFORMIO_BUILD_FLAGS="-DCV_PANEL_ST7789=1" pio run -d firmware -e cyd28r 2>&1 | tail -2
```

Expected: four `ok` lines, then `SUCCESS` for the ST7789 variant. If `VSPI_HOST` is undeclared, the platform's IDF names it `SPI3_HOST`; use that in `board.h`.

- [ ] **Step 6: Commit**

```bash
git add firmware/src/board_config.h firmware/platformio.ini
git commit -m "✨ (firmware): Add the 2.8\" ESP32-2432S028R board target"
```

---

### Task 3: Bring up the display on the 2.8" board

On-device task. Each try is one flash; the shell variable carries the candidate values so nothing is edited until the end. The calibration screen is the test card: it appears on every boot until calibration completes, and its colours are known.

| Calibration screen element | Correct look |
|---|---|
| Background | very dark navy (`#1a1b26`) |
| "touch calibration / tap each arrow tip" text | light lavender, upright, readable left to right, centred |
| Corner arrow | blue (`#7aa2f7`), in the top-left corner first |
| USB port | at the top edge of the board |

**Files:**
- Modify: `firmware/src/board_config.h` (2.8" defaults, Step 6)

**Interfaces:**
- Consumes: env `cyd28r` and the tuning macros from Task 2.
- Produces: verified 2.8" defaults for `CV_ROTATION`, `CV_PANEL_BGR`, `CV_PANEL_INVERT`, `CV_PANEL_ST7789`.

- [ ] **Step 1: Record what the board is**

Look at the short edge and the back. Note: one micro-USB port, or USB-C plus micro-USB; the model text printed in gold near the speaker connector. Plug in with the cable from Global Constraints and run:

```bash
ls /dev/cu.usbserial-* /dev/cu.wchusbserial-* 2>/dev/null
```

Expected: one serial device. macOS's built-in CH34x driver needs no install.

- [ ] **Step 2: Erase the board and flash the defaults**

Erasing wipes the factory demo and any stored settings; re-flashing restores a working board.

```bash
pio run -d firmware -e cyd28r -t erase
pio run -d firmware -e cyd28r -t upload
```

Expected: upload ends with `Hard resetting via RTS pin...` and `SUCCESS`. If it stalls at `Connecting....`, hold BOOT, tap RST, release BOOT, and retry.

- [ ] **Step 3: Check the backlight**

Expected: the screen lights up. If it stays dark but the upload succeeded, check `board::BL` is 21 with the host check from Task 2, Step 3.

- [ ] **Step 4: Fix the colours**

Compare against the table. Change one variable per flash:

| What you see | Next flash |
|---|---|
| Background near white, text dark | `export CV='-DCV_PANEL_INVERT=1'` |
| Arrow orange instead of blue | add `-DCV_PANEL_BGR=0` to `CV` |
| Noise, stripes, or a picture offset by a band | `export CV='-DCV_PANEL_ST7789=1 -DCV_PANEL_INVERT=1'` |

```bash
PLATFORMIO_BUILD_FLAGS="$CV" pio run -d firmware -e cyd28r -t upload
```

Expected: background dark navy, arrow blue.

- [ ] **Step 5: Fix the orientation**

With the colours right, find the rotation that shows upright, left-to-right text with the USB port at the top:

| What you see | Next value |
|---|---|
| Text upright, USB at the bottom | `-DCV_ROTATION=2` |
| Text mirrored | `-DCV_ROTATION=4`, then `6` |
| Text sideways | the panel memory is landscape: try `-DCV_ROTATION=1`, then `3`, `5`, `7` |

```bash
PLATFORMIO_BUILD_FLAGS="$CV -DCV_ROTATION=2" pio run -d firmware -e cyd28r -t upload
```

Expected: centred, upright text; USB at the top; first arrow top-left.

- [ ] **Step 6: Bake the values in**

Write the working values into the 2.8" branch of `firmware/src/board_config.h` (the four `#define` defaults under `#if defined(CV_BOARD_CYD28R)`). If the panel needed ST7789, also add inside that branch, above `#ifndef CV_ROTATION`:

```cpp
#ifndef CV_PANEL_ST7789
#define CV_PANEL_ST7789 1
#endif
``` Replace the branch comment's "the values found on the unit in Task 3 …" sentence with what was verified, for example: `Verified on the unit: rotation 0 shows it upright with the USB port at the top; BGR, not inverted.` Then:

```bash
unset CV
c++ -std=c++17 -fsyntax-only -DCV_BOARD_CYD28R -I firmware/src firmware/hostcheck/board_config_check.cpp && echo PASS-28
pio run -d firmware -e cyd28r -t upload
```

Expected: `PASS-28`, and the plain flash shows the correct calibration screen.

- [ ] **Step 7: Commit**

```bash
git add firmware/src/board_config.h
git commit -m "🔧 (firmware): Set the 2.8\" panel's rotation and colour order from the unit"
```

---

### Task 4: Bring up touch on the 2.8" board

**Files:**
- Modify: `firmware/src/board_config.h` or `firmware/platformio.ini` (only if the soft-SPI fallback in Step 3 is needed)

**Interfaces:**
- Consumes: env `cyd28r-touchlog` from Task 2, panel defaults from Task 3.
- Produces: working touch, a stored calibration, and verified USB-bottom taps.

- [ ] **Step 1: Flash the touch-log build**

```bash
pio run -d firmware -e cyd28r-touchlog -t upload
pio device monitor -d firmware -e cyd28r-touchlog
```

- [ ] **Step 2: Read raw touches**

Press firmly with a fingernail near each screen corner in turn.

Expected: `{"type":"log","raw":[x,y],...}` lines while pressing, raw values spreading across roughly 200–3900 between corners. The calibration screen also advances to the next arrow on each press.

- [ ] **Step 3: If nothing is logged, switch touch to software SPI**

No log lines, or raw stuck at `[0,0]` or `[4095,4095]`, means the touch bus is not reaching the XPT2046. Flash once with the fallback:

```bash
PLATFORMIO_BUILD_FLAGS="-DCV_TOUCH_SOFT_SPI" pio run -d firmware -e cyd28r-touchlog -t upload
```

If that logs touches, make it permanent by changing both 2.8" envs' flags in `firmware/platformio.ini`:

```ini
[env:cyd28r]
extends = env:cyd24r
build_flags = ${env:cyd24r.build_flags} -DCV_BOARD_CYD28R -DCV_TOUCH_SOFT_SPI
```

- [ ] **Step 4: Calibrate and use the UI**

Tap the four arrow tips. Flash the normal build:

```bash
pio run -d firmware -e cyd28r -t upload
```

Expected: the main screen appears (no daemon, so no sessions). Swipe left and right: the layout changes. Long-press the top bar: the settings screen opens.

- [ ] **Step 5: Check backlight dimming on GPIO 21**

The firmware reads serial only after calibration, so this comes after Step 4. With no daemon installed, send a config line from the monitor:

```bash
pio device monitor -d firmware -e cyd28r
```

Type, then Enter: `{"type":"config","brightness":20}`

Expected: a `hello` line echoing `"brightness":20`, and the screen visibly dims. Send `{"type":"config","brightness":100}` to restore. Then send `{"type":"config","dim_after":1}`, leave the screen untouched for a minute, and expect it to fade to nearly off; a touch wakes it. Send `{"type":"config","dim_after":10}` to restore. Exit with Ctrl-C.

- [ ] **Step 6: Re-run calibration from boot**

Unplug, hold a finger on the screen, plug back in.

Expected: the calibration screen returns. Complete it; the main screen follows.

- [ ] **Step 7: Check taps with the USB port at the bottom**

In settings, switch the orientation to USB at the bottom. Turn the board over.

Expected: the UI is upright, and a long press on the top bar (now away from the USB port) opens settings. Switch back to USB at the top.

- [ ] **Step 8: Commit (only if Step 3 changed files)**

```bash
git add firmware/platformio.ini
git commit -m "🐛 (firmware): Read the 2.8\" touch controller over software SPI"
```

---

### Task 5: End to end with the daemon, and screenshot colours

**Files:**
- Modify: `firmware/src/board_config.h` (only if Step 4 needs the other read-back order)

**Interfaces:**
- Consumes: the flashed `cyd28r` firmware.
- Produces: a verified full build: sessions shown, tap-to-focus, correct screenshots.

- [ ] **Step 1: Install the daemon and hooks**

This writes hooks into `~/.claude/settings.json`, a status line, and a launchd daemon; `cyberviber uninstall` removes all three.

```bash
cd ~/Documents/code/cyberviber
go test ./...
go install ./cmd/cyberviber && ~/go/bin/cyberviber install
```

Expected: tests pass; install prints the four `✓` lines from the build guide.

- [ ] **Step 2: Check the link**

```bash
~/go/bin/cyberviber status
```

Expected: the session table, and the device lists your open Claude Code sessions within a few seconds.

- [ ] **Step 3: Tap a session**

Expected: its terminal comes to the front.

- [ ] **Step 4: Check screenshot colours**

```bash
~/go/bin/cyberviber screenshot /tmp/cv-shot.png && open /tmp/cv-shot.png
```

Expected: the PNG matches the screen. If red and blue or all three channels are rotated, flip the read-back order and reflash (the daemon frees the port for 120 s):

```bash
~/go/bin/cyberviber release 120
PLATFORMIO_BUILD_FLAGS="-DCV_READBACK_SHIFTED=1" pio run -d firmware -e cyd28r -t upload
~/go/bin/cyberviber release 1
```

If that fixes it, change the 2.8" default `CV_READBACK_SHIFTED` to `1` in `board_config.h`, rerun the 2.8" host check, reflash without the variable, and commit:

```bash
git add firmware/src/board_config.h
git commit -m "🐛 (firmware): Use the shifted read-back order on the 2.8\" panel"
```

---

### Task 6: Document the 2.8" target

**Files:**
- Modify: `README.md:74-86`

- [ ] **Step 1: Update the Firmware section**

Replace the line

```markdown
Board: ESP32-2432S024R (2.4" TFT, XPT2046 resistive touch, CH340 USB). [PlatformIO](https://platformio.org) builds and flashes it:
```

with

```markdown
Board: ESP32-2432S024R (2.4" TFT, XPT2046 resistive touch, CH340 USB), or the 2.8" ESP32-2432S028R with `-e cyd28r` on every `pio` command. [PlatformIO](https://platformio.org) builds and flashes it:
```

and append after the paragraph ending in `firmware/tools/gen-fonts.sh`.:

```markdown

Board pins and panel settings live in `firmware/src/board_config.h`; `c++ -std=c++17 -fsyntax-only [-DCV_BOARD_CYD28R] -I firmware/src firmware/hostcheck/board_config_check.cpp` checks them. A 2.8" board whose picture looks wrong can be tuned without edits, one flash per try: `PLATFORMIO_BUILD_FLAGS="-DCV_ROTATION=2 -DCV_PANEL_INVERT=1" pio run -d firmware -e cyd28r -t upload`. The other knobs are `CV_PANEL_BGR`, `CV_PANEL_ST7789`, `CV_READBACK_SHIFTED` and `CV_TOUCH_SOFT_SPI`.
```

- [ ] **Step 2: Verify the documented commands run**

```bash
c++ -std=c++17 -fsyntax-only -DCV_BOARD_CYD28R -I firmware/src firmware/hostcheck/board_config_check.cpp && echo PASS-28
pio run -d firmware -e cyd24r >/dev/null 2>&1 && echo cyd24r-ok
```

Expected: `PASS-28`, `cyd24r-ok`.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "📝 (readme): Document the 2.8\" board target"
```
