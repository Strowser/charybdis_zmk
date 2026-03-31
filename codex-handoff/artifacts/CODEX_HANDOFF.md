# Codex Handoff — Charybdis 4x6 RGB LED Project
Date: 2026-03-30

## Situation
We are trying to get 29 SK6812 MINI-E RGB LEDs working on the left half of a Charybdis 4x6 wireless keyboard running ZMK firmware on a nice_nano_v2 (nRF52840).

The user's DIN wire is connected to the SPI/trackball connector on the BK nice!nano holder. User says it's the CS port. We have been testing P0.17 as the data pin (user's choice). All 29 LEDs are confirmed working with a previous (now lost) firmware.

**After exhaustive testing, both SPI-based AND PWM-based WS2812 drivers produce the same broken output: scattered/partial red LEDs, wrong color, drifting across resets.** This proves the signal generation method is not the problem.

## Immediate Next Step
Build a **GPIO toggle test** to verify P0.17 is physically connected to the LED DIN:

1. Edit `G:\zmk-build-left-20260330\zephyr\drivers\led_strip\ws2812_spi.c`
2. In the `ws2812_spi_init()` function, after the existing init code, add a background thread or modify the update function to:
   - Configure P0.17 as a GPIO output (high drive)
   - In `ws2812_strip_update_rgb()`, before any LED data, toggle P0.17 HIGH for 500ms then LOW for 500ms (using `nrf_gpio_pin_set()` / `nrf_gpio_pin_clear()` and `k_msleep()`)
   - This will NOT produce valid WS2812 data — it will just make the first LED blink if P0.17 is connected
3. Keep everything else the same (overlay, conf, keymap)

If the first LED blinks/flashes with the toggle test → P0.17 IS connected, problem is elsewhere.
If nothing happens → P0.17 is NOT connected to LED DIN, and we need to find the real pin.

**Alternative approach:** Instead of a toggle test, do a **pin sweep**. In the init function, loop through candidate pins (P0.06, P0.08, P0.17, P0.20, P0.03, P0.22, P0.24), and for each pin:
- Send a valid WS2812 frame for 1 LED (green) using the PWM method
- Wait 2 seconds
- Move to next pin
The user watches which pin lights up the LED correctly. This finds the right pin in one flash.

## If Pin Is Confirmed Wrong
The SPI connector on the BK nice!nano holder has these pins (from trackball config):
- P0.06 = IRQ/MOT
- P0.08 = SCK
- P0.17 = MOSI/MISO (SDIO)
- P0.20 = CS

User says DIN is on "CS port" → should be P0.20. But P0.20 also gave bad results (scattered).
However, P0.20 was only tested with SPI driver, never with PWM. If the pin sweep reveals P0.20
works with PWM, that's the answer.

Also possible: the DIN is on a completely different connector or pin not on the SPI header.

## If Pin Is Confirmed Correct (P0.17 toggles the LED)
Then the problem is in how we generate WS2812 data. Possible causes:
- The PWM driver has a bug (check polarity, bit order, timing values)
- The SK6812 MINI-E needs different timing than standard WS2812
- Try wider T0H/T1H values: T0H=8 (500ns), T1H=16 (1000ns)
- Try sending just 1 LED of pure 0xFF 0xFF 0xFF (white) to see if any color appears

## Build Environment

### Paths
- Build workspace: `G:\zmk-build-left-20260330`
- Build overlay: `G:\zmk-build-left-20260330\module\boards\shields\charybdis-bt\charybdis_left.overlay`
- Build conf: `G:\zmk-build-left-20260330\config\charybdis_left.conf`
- Build keymap: `G:\zmk-build-left-20260330\config\charybdis.keymap`
- Driver to modify: `G:\zmk-build-left-20260330\zephyr\drivers\led_strip\ws2812_spi.c`
- Output UF2: `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
- Rebuild script: `G:\zmk-build-left-20260330\rebuild.ps1`

### Build Command (PowerShell)
Full clean rebuild:
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "G:/zmk-build-left-20260330/rebuild.ps1"
```

The rebuild.ps1 contains:
```powershell
$env:PATH = "C:\Users\rmari\AppData\Roaming\Python\Python310\site-packages\cmake\data\bin;C:\Users\rmari\AppData\Roaming\Python\Python310\Scripts;" + $env:PATH
& "C:\Users\rmari\AppData\Roaming\Python\Python310\Scripts\west.exe" build -d build-left -p always -s zmk/app -b nice_nano_v2 -- "-DZMK_CONFIG=G:/zmk-build-left-20260330/config" "-DSHIELD=charybdis_left" "-DBOARD_ROOT=G:/zmk-build-left-20260330/module" "-DZEPHYR_TOOLCHAIN_VARIANT=gnuarmemb" "-DGNUARMEMB_TOOLCHAIN_PATH=C:/Program Files (x86)/Arm GNU Toolchain arm-none-eabi/14.2 rel1"
```

### Flash Command (PowerShell, when D: is mounted as NICENANO)
```powershell
powershell -NoProfile -Command "Copy-Item 'G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2' 'D:\zmk.uf2' -Force"
```
D: disappearing after copy = successful flash (nice!nano reboots automatically).

To put nice!nano in bootloader: double-tap the reset button on the board.

### Shell Notes
- Codex shell is cmd.exe, NOT bash
- Use PowerShell via `powershell -NoProfile -Command "..."` for complex commands
- Paths with spaces need quoting
- Build output shows warnings about RWX permissions — these are harmless, ignore them
- Build takes ~60-90 seconds for full rebuild

### Build Workspace Notes
- The keymap at `G:\zmk-build-left-20260330\config\charybdis.keymap` has the three trackball_listener blocks REMOVED (build-only change, not in the repo)
- The repo keymap at `G:\Downloads\zmk testing\zmk-config\config\charybdis.keymap` still has them
- Do NOT sync these — the build workspace keymap is intentionally different

## Current State of Key Files

### ws2812_spi.c (CURRENTLY MODIFIED — PWM driver)
Currently contains a custom PWM-based driver using direct NRF_PWM0 register access.
The driver:
- Ignores SPI entirely (SPI DT node exists but is unused)
- Programs NRF_PWM0 directly: 16MHz clock, COUNTERTOP=20, T0H=6, T1H=14
- Configures P0.17 as output with H0H1 drive strength
- Converts pixel bytes to PWM duty cycle sequence
- Uses SEQEND0→STOP shortcut for one-shot playback
- Appends 80 zero-duty reset words (100µs)
- Result: SAME broken output as SPI (proves timing is not the issue)

To restore stock driver:
```
cd G:\zmk-build-left-20260330
git -C zephyr checkout -- drivers/led_strip/ws2812_spi.c
```

### charybdis_left.overlay (build workspace)
- No pmw3610 include, no trackball listener
- spi3 SPIM3 on P0.17, MOSI only, H0H1 drive
- 4MHz, 0x70/0x40, GRB, chain-length=29

### charybdis_left.conf (build workspace)
- RGB underglow enabled, spectrum effect, 100% brightness, auto-off idle

## Complete Test History (DO NOT REPEAT THESE)

### SPI tests on P0.17
- 4MHz 0x70/0x40 GRB chain=29 normal drive → 8 contiguous red
- 8MHz 0x70/0x40 GRB chain=29 → same
- 4MHz 0x7C/0x60 RGB chain=29 → 8 red
- 4MHz 0x70/0x40 GRB chain=29 hue=240 → 8 red
- 4MHz 0x70/0x40 GRB chain=1 H0H1 reset=120 → bad
- 8MHz+cpha 0xFC/0xC0 GRB chain=1 H0H1 → slightly diff
- non-DMA 4MHz 0x70/0x40 GRB chain=1/29 H0H1 → bad
- non-DMA + 80 zero padding → same
- non-DMA + hardcoded green bytes → still red
- non-DMA + dummy SCK on P0.03 → 1 red LED
- non-DMA + deterministic startup → drifting red
- SPIM3 clean stock driver spectrum → same red

### SPI tests on P0.20
- 4MHz 0x70/0x40 GRB chain=29 → scattered partial red
- 8MHz same → same
- 4MHz 0x7C/0x60 RGB → same

### PWM test on P0.17
- NRF_PWM0 direct registers, 16MHz, T0H=6 T1H=14 TOP=20, H0H1 → SAME red

### Other
- I2S driver: build failed
- ws2812-gpio: only nRF51
- NVS settings reset: no fix
- LED pattern drifts across plain resets (no reflash)

## Hardware
- Board: nice_nano_v2 (nRF52840) clones
- LEDs: 29× SK6812 MINI-E, VCC on battery+ (~3.7V), 3.3V logic confirmed sufficient
- Left half = BLE peripheral, right half = central + trackball
- All 29 LEDs verified working with previous firmware (source lost)
- LED DIN wire goes to SPI/trackball connector on BK nice!nano holder

## GitHub Repo
- https://github.com/Strowser/charybdis_zmk.git
- User: Strowser
- Repo state is behind build workspace (repo still has some older overlay changes)
- Build workspace is the source of truth for testing
