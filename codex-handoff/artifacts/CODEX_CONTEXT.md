# Codex Continuation Context
Date: 2026-03-30 (latest update)

## CRITICAL FINDING
The PWM-based driver (direct NRF_PWM0 register access, 16MHz, precise 800kHz WS2812 timing)
produces THE EXACT SAME RESULT as the SPI driver. Same red LEDs, same pattern.

This means the problem is NOT the SPI waveform timing. Both SPI and PWM produce the same
broken output. The signal generation method is irrelevant.

**This can only mean one of two things:**
1. P0.17 is NOT connected to the LED DIN (user insists it is — do not change pin)
2. There is something else on P0.17 interfering (another peripheral, the kscan, etc.)

## Important: Check for pin conflicts
On nice_nano_v2, pro_micro pin 2 = GPIO P0.17. Check if any kscan GPIO or other
peripheral is also using P0.17. The kscan uses pro_micro pins: 19,20,10,6,7,8 (cols)
and 21,18,5,4,9 (rows). Pro_micro 2 is NOT in that list. But verify the actual GPIO
mapping — pro_micro numbers != GPIO numbers.

nice_nano_v2 pro_micro to GPIO mapping (verify from board DTS):
- pro_micro 0 = P0.06
- pro_micro 1 = P0.08
- pro_micro 2 = P0.17
- pro_micro 3 = P0.20
- pro_micro 4 = P0.22
- pro_micro 5 = P0.24
- pro_micro 6 = P1.00
- pro_micro 7 = P0.11
- pro_micro 8 = P1.04
- pro_micro 9 = P0.09
- pro_micro 10 = P0.10
- pro_micro 18 = P0.02
- pro_micro 19 = P0.29
- pro_micro 20 = P0.31
- pro_micro 21 = P0.30

So P0.17 should not conflict with kscan. But VERIFY from the actual board DTS file.

## NEXT STEPS TO TRY
1. **Verify the nice_nano_v2 board DTS** to confirm P0.17 mapping and check for conflicts
2. **Try a simple GPIO toggle test** — modify the driver to just toggle P0.17 at 1Hz.
   If the LED strip doesn't even blink, P0.17 is definitely not connected to DIN.
3. **Ask user to use a multimeter** in continuity mode to trace which nice!nano pin
   their DIN wire actually connects to.
4. **Try every single GPIO pin systematically** with PWM driver (user previously refused
   but evidence now overwhelmingly points to wrong pin)

## Project
Charybdis 4x6 wireless keyboard, nice_nano_v2 (nRF52840), SK6812 MINI-E RGB LEDs.
Left half only (peripheral). 29 LEDs on left half.
DIN wire connected to SPI connector on the BK nice!nano holder (user says CS port).
User insists P0.17 is correct — but P0.17 is MOSI on the SPI connector, not CS.
CS on the SPI connector = P0.20 (from trackball config).

## Repos & Paths
- **GitHub repo**: https://github.com/Strowser/charybdis_zmk.git (user: Strowser)
- **Repo clone**: G:\Downloads\zmk testing\zmk-config
- **Build workspace**: G:\zmk-build-left-20260330
- **Build overlay**: G:\zmk-build-left-20260330\module\boards\shields\charybdis-bt\charybdis_left.overlay
- **Build conf**: G:\zmk-build-left-20260330\config\charybdis_left.conf
- **Build keymap**: G:\zmk-build-left-20260330\config\charybdis.keymap
- **Output UF2**: G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2
- **NICENANO drive**: D:\ (when in bootloader mode)
- **Rebuild script**: G:\zmk-build-left-20260330\rebuild.ps1

## Build Commands
Full rebuild:
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "G:/zmk-build-left-20260330/rebuild.ps1"
```

Flash (when D: is mounted):
```powershell
powershell -NoProfile -Command "Copy-Item 'G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2' 'D:\zmk.uf2' -Force"
```

## Current Driver State
File: G:\zmk-build-left-20260330\zephyr\drivers\led_strip\ws2812_spi.c
- Modified to use direct NRF_PWM0 register access instead of SPI
- Hardcoded P0.17 as output pin
- PWM at 16MHz, COUNTERTOP=20 (800kHz), T0H=6, T1H=14
- High drive strength (H0H1)
- DMA-driven sequence playback with SEQEND0→STOP shortcut
- 80 reset words (100µs) appended after pixel data
- Still uses ws2812-spi DT compatible for drop-in compatibility
- Result: SAME broken output as all SPI tests

## Current Overlay State (build workspace)
File: G:\zmk-build-left-20260330\module\boards\shields\charybdis-bt\charybdis_left.overlay
- No pmw3610 include
- No trackball listener
- spi3 SPIM3 on P0.17 with MOSI only + H0H1 drive
- 4MHz, 0x70/0x40, GRB, chain-length=29
- (The SPI DT node is still present but the driver ignores SPI and uses PWM instead)

## Current Conf State (build workspace)
File: G:\zmk-build-left-20260330\config\charybdis_left.conf
- CONFIG_ZMK_RGB_UNDERGLOW=y, CONFIG_WS2812_STRIP=y, CONFIG_SPI=y
- Spectrum effect (2), brightness 100%, saturation 100%
- Auto-off on idle

## Current Keymap State (build workspace)
File: G:\zmk-build-left-20260330\config\charybdis.keymap
- trackball listener blocks removed (build-only, not in repo)
- RGB key bindings on FN layer

## Complete Test History (DO NOT REPEAT)

### SPI driver tests on P0.17
| Compat | Freq | Frames | Color Map | Chain | Drive | Reset | Extras | Result |
|--------|------|--------|-----------|-------|-------|-------|--------|--------|
| nrf-spim | 4M | 70/40 | GRB | 29 | normal | none | none | 8 red |
| nrf-spim | 8M | 70/40 | GRB | 29 | normal | none | none | same |
| nrf-spim | 4M | 7C/60 | RGB | 29 | normal | none | none | 8 red |
| nrf-spim | 4M | 70/40 | GRB | 29 | normal | none | hue=240 | 8 red |
| nrf-spim | 4M | 70/40 | GRB | 1 | H0H1 | 120 | none | bad |
| nrf-spim | 8M+cpha | FC/C0 | GRB | 1 | H0H1 | 120 | none | slightly diff |
| nrf-spi | 4M | 70/40 | GRB | 1 | H0H1 | 120 | non-DMA | bad |
| nrf-spi | 4M | 70/40 | GRB | 29 | H0H1 | 120 | +80 zero pad | same |
| nrf-spi | 4M | 70/40 | GRB | 1 | H0H1 | 120 | hardcoded green | still red |
| nrf-spi | 4M | 70/40 | GRB | 29 | H0H1 | 120 | hardcoded green | same |
| nrf-spi+SCK | 4M | 70/40 | GRB | 29 | H0H1 | 120 | dummy SCK P0.03 | 1 red |
| nrf-spi+SCK | 4M | 70/40 | RGB | 29 | H0H1 | 120 | dummy SCK P0.03 | drifting |
| nrf-spi+SCK | 4M | 70/40 | RGB | 29 | H0H1 | 120 | deterministic startup | more but red |
| nrf-spim | 4M | 70/40 | GRB | 29 | H0H1 | none | stock driver clean | same red |

### SPI driver tests on P0.20
| Freq | Frames | Color Map | Chain | Result |
|------|--------|-----------|-------|--------|
| 4M | 70/40 | GRB | 29 | scattered partial red |
| 8M | 70/40 | GRB | 29 | same |
| 4M | 7C/60 | RGB | 29 | same |

### PWM driver test on P0.17
| Method | Freq | Timing | Drive | Result |
|--------|------|--------|-------|--------|
| NRF_PWM0 direct regs | 16MHz | T0H=6,T1H=14,TOP=20 | H0H1 | SAME red pattern |

### Other tests
- I2S driver: BUILD FAILED (not supported in this ZMK fork)
- spi0 /delete-node/: BUILD FAILED
- ws2812-gpio: only nRF51, not available
- Settings reset (NVS clear): did not fix
- LED pattern drifts across plain resets

## Hardware Facts
- LEDs: SK6812 MINI-E, VCC connected to battery+ (~3.7V LiPo)
- At 3.7V VCC, logic threshold ~2.6V, 3.3V output is sufficient
- All 29 LEDs confirmed working with all colors with previous (lost) firmware
- nice_nano_v2 clone controllers
- Left half = BLE peripheral, right half = central with trackball
