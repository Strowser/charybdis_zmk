# Claude Review Note

Repo: `G:\Downloads\zmk testing\zmk-config`
Date: `2026-03-30`

Critical workflow finding:

1. The rebuild output file is `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`.
2. The copied artifact path `G:\zmk-build-left-20260330\artifacts\charybdis_qwerty_left.uf2` was stale during recent flashes.
3. Verified mismatch:
   - fresh build UF2 timestamp: `2026-03-30 16:39:08`
   - stale artifact UF2 timestamp: `2026-03-30 16:22:15`
   - SHA256 differed between the two files
4. Implication:
   - several recent flashes likely used an older UF2 than the current rebuilt test
5. Corrective action:
   - future flashes must use `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2` directly, or first sync that file into the artifact path before flashing
6. Confirmed flash after finding the bug:
   - copied `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2` directly to `D:\charybdis_qwerty_left.uf2`
   - `D:` disappeared after copy, which is the expected nice!nano reboot behavior
7. User-reported result after that direct fresh-build flash:
   - still the same incorrect hardware behavior
   - no observable change from the previous bad output

Settings confounder:

1. ZMK underglow restores saved state from settings (`rgb/underglow/state`) on boot.
2. Implication:
   - `.conf` defaults for hue/effect/brightness may have been masked by NVS during testing
3. Least-destructive next isolation step now prepared:
   - built one-shot settings reset UF2 at `G:\zmk-build-left-20260330\build-settings-reset\zephyr\zmk.uf2`
   - build target: `nice_nano_v2` with `SHIELD=settings_reset`
4. Side effects if flashed:
   - clears saved settings on that MCU, including saved RGB underglow state
   - `settings_reset.conf` also disables BLE for that one-shot firmware
5. Intended workflow if approved:
   - flash `build-settings-reset\zephyr\zmk.uf2`
   - let it boot once to clear settings
   - immediately reflash the current `P0.17` test UF2 from `build-left\zephyr\zmk.uf2`
6. Reset flash status:
   - `G:\zmk-build-left-20260330\build-settings-reset\zephyr\zmk.uf2` has now been flashed to `NICENANO`
   - copied as `D:\settings_reset.uf2`
   - `D:` disappeared after copy, which is the expected reboot behavior
7. Next step pending:
   - board must be put back into bootloader once more
   - then reflash `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2` to test the current `P0.17` firmware after settings were cleared
8. Post-reset test flash status:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2` has now been reflashed after the settings reset
   - copied as `D:\charybdis_qwerty_left.uf2`
   - `D:` disappeared after copy, which is the expected reboot behavior
9. Next required observation:
   - compare the hardware behavior after this post-reset flash against the pre-reset bad result

Current repo-visible change set:

1. `boards/shields/charybdis-bt/charybdis_left.overlay`
   - removed `#include "charybdis_pmw3610.dtsi"`
   - removed the left-side `trackball_listener` block
   - kept RGB output on `P0.17` only, now testing it on `spi0`
   - added `#include <zephyr/dt-bindings/pinctrl/nrf-pinctrl.h>`
   - current LED test settings in the repo file:
     - `spi-max-frequency = <4000000>;`
     - `chain-length = <1>;`
     - `spi-one-frame = <0x70>;`
     - `spi-zero-frame = <0x40>;`
     - `color-mapping = <LED_COLOR_ID_GREEN LED_COLOR_ID_RED LED_COLOR_ID_BLUE>;`
     - `reset-delay = <120>;`
     - `nordic,drive-mode = <NRF_DRIVE_H0H1>;` on the active `P0.17` default pinctrl group
     - transport under test is `spi0` with `compatible = "nordic,nrf-spi"` to avoid SPIM EasyDMA tail corruption while keeping MOSI on `P0.17`

Current repo state that is relevant but not dirty:

1. `boards/shields/charybdis-bt/charybdis_left.conf`
   - currently set for debug:
     - `CONFIG_ZMK_RGB_UNDERGLOW_EFF_START=0`
     - `CONFIG_ZMK_RGB_UNDERGLOW_BRT_START=100`
     - `CONFIG_ZMK_RGB_UNDERGLOW_HUE_START=240`
     - `CONFIG_ZMK_RGB_UNDERGLOW_SAT_START=100`

2. `config/charybdis.keymap`
   - includes `#include <dt-bindings/zmk/rgb.h>`
   - repo file still contains the three `&vtrackball` listener blocks

Important build-only change that is NOT in the repo:

1. Temp build workspace keymap:
   - file: `G:\zmk-build-left-20260330\config\charybdis.keymap`
   - removed only these three blocks so the left half could build without left-side trackball nodes:
     - `trackball_listener`
     - `trackball_snipe_listener`
     - `trackball_scroll_listener`
   - this was done only in the temp build copy, not in the repo keymap
2. Current temporary Zephyr driver experiment:
   - file: `G:\zmk-build-left-20260330\zephyr\drivers\led_strip\ws2812_spi.c`
   - this is build-only scaffolding, not a repo patch
   - added `WS2812_SPI_RESET_PAD_BYTES = 80`
   - enlarged the transfer buffer by that amount
   - appended 80 raw `0x00` bytes to the end of each SPI transfer before the existing `reset-delay`
   - rationale: force an explicit on-wire low/reset tail instead of relying only on controller idle state plus `k_usleep()`
3. Current temporary driver override layered on top of that:
   - same file: `G:\zmk-build-left-20260330\zephyr\drivers\led_strip\ws2812_spi.c`
   - still build-only scaffolding, not a repo patch
   - ignores application-provided pixel values and always emits one logical pure-green pixel on the wire
   - rationale: remove ZMK underglow state/effect/color logic from the test and isolate only the physical signal path on `P0.17`
4. Important interpretation correction:
   - when `chain-length = <1>` is used on a physically longer WS2812/SK6812 chain, only the first LED receives a complete new pixel frame
   - downstream LEDs can keep their previously latched state until they receive fresh data
   - implication: the recent one-pixel tests were only reliable for the head of the chain and cannot by themselves prove full-chain behavior

Latest built test:

1. Latest flashed trusted `spi3` baseline retest on `P0.17` used:
   - `chain-length = <1>;`
   - `reset-delay = <120>;`
   - `nordic,drive-mode = <NRF_DRIVE_H0H1>;` on the active `spi3_default` group
   - `color-mapping = <LED_COLOR_ID_GREEN LED_COLOR_ID_RED LED_COLOR_ID_BLUE>;`
   - `spi3` with Zephyr's normal SPI waveform:
     - `spi-max-frequency = <4000000>;`
     - `spi-one-frame = <0x70>;`
     - `spi-zero-frame = <0x40>;`
2. That exact baseline waveform was flashed directly from:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
3. Post-flash hardware observation for that exact `spi3` baseline retest:
   - exactly the same incorrect hardware behavior
   - no observable change from the prior bad output
4. Current repo patch now prepared for the next same-pin test is:
   - `chain-length = <1>;`
   - `reset-delay = <120>;`
   - `nordic,drive-mode = <NRF_DRIVE_H0H1>;`
   - `color-mapping = <LED_COLOR_ID_GREEN LED_COLOR_ID_RED LED_COLOR_ID_BLUE>;`
   - `spi0` on `P0.17`
   - switched from `compatible = "nordic,nrf-spim"` to `compatible = "nordic,nrf-spi"` to remove EasyDMA from the transfer path
   - retained Zephyr's normal SPI waveform:
     - `spi-max-frequency = <4000000>;`
     - `spi-one-frame = <0x70>;`
     - `spi-zero-frame = <0x40>;`
5. Rebuilt successfully with the current repo overlay plus the temp build-only keymap adjustment above.
6. Generated DTS for the current non-DMA retest now confirms:
   - `spi0` is `compatible = "nordic,nrf-spi"`
   - `P0.17` is the active MOSI output
   - `chain-length = <1>`
   - `spi-max-frequency = <4000000>`
   - `spi-one-frame = <0x70>`
   - `spi-zero-frame = <0x40>`
   - `color-mapping = <0x2 0x1 0x3>` (`GRB`)
   - `reset-delay = <120>`
   - `nordic,drive-mode = <0x3>` (`NRF_DRIVE_H0H1`)
7. Build log confirmation for this retest:
   - non-DMA SPI objects were compiled: `spi_nrfx_spi.c` and `nrfx_spi.c`
8. Current UF2 ready to flash directly:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
9. This non-DMA retest has now been flashed directly from:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
10. Flash verification for the current non-DMA retest:
   - copied to `D:\charybdis_qwerty_left.uf2`
   - `D:` disappeared after copy, which is the expected nice!nano reboot behavior
11. Previous generated DTS for the last flashed `spi3` baseline retest confirmed:
   - `spi3` MOSI on `P0.17`
   - `chain-length = <1>`
   - `spi-max-frequency = <4000000>`
   - `spi-one-frame = <0x70>`
   - `spi-zero-frame = <0x40>`
   - `color-mapping = <0x2 0x1 0x3>` (`GRB`)
   - `reset-delay = <120>`
   - `nordic,drive-mode = <0x3>` (`NRF_DRIVE_H0H1`)
12. Previous generated DTS for the earlier flashed `spi3` timing variant confirmed:
   - `spi3` MOSI on `P0.17`
   - `chain-length = <1>`
   - `spi-max-frequency = <8000000>`
   - `spi-cpha`
   - `spi-one-frame = <0xFC>`
   - `spi-zero-frame = <0xC0>`
   - `color-mapping = <0x2 0x1 0x3>` (`GRB`)
   - `reset-delay = <120>`
   - `nordic,drive-mode = <0x3>` (`NRF_DRIVE_H0H1`)
13. Current build-only driver-padded retest keeps the same repo-visible `spi0` non-DMA `P0.17` settings above and additionally:
    - uses the temporary `ws2812_spi.c` patch that appends 80 raw `0x00` bytes after the pixel data
14. Rebuilt successfully with that driver-padded experiment.
15. Current UF2 ready to flash directly:
    - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
16. This driver-padded retest has now been flashed directly from:
    - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
17. Flash verification for the current driver-padded retest:
    - copied to `D:\charybdis_qwerty_left.uf2`
    - `D:` disappeared after copy, which is the expected nice!nano reboot behavior

Latest observed hardware result:

1. User-reported result from photo `G:\Downloads\zmk testing\photo_2026-03-30_16-23-57.jpg`:
   - same incorrect behavior after the `P0.17` single-LED, high-drive, long-reset test
2. Because the failure persisted even with `chain-length = <1>`, stronger drive, longer reset delay, a same-pin `GRBW` packet test on `spi3`, and then the same result again on `spi0`, the next follow-up test keeps `P0.17` on `spi3` but switches to a more different SPI waveform family.
3. The flashed `spi3` timing variant also includes:
   - `reset-delay = <120>;`
   - `nordic,drive-mode = <NRF_DRIVE_H0H1>;`
   - `spi-cpha;`
   - `spi-max-frequency = <8000000>;`
   - `spi-one-frame = <0xFC>;`
   - `spi-zero-frame = <0xC0>;`
4. User-reported result after flashing that same `P0.17` timing variant immediately after settings reset:
   - new photo: `G:\Downloads\zmk testing\photo_2026-03-30_16-52-20.jpg`
   - only a small change occurred
   - one additional red LED disappeared
   - output is still fundamentally wrong, not corrected
5. User-reported result after flashing the later direct `spi3` baseline retest:
   - exactly the same incorrect behavior
   - no observable change from the prior bad output
6. User-reported result after flashing the later direct `spi0` non-DMA retest:
   - exactly the same incorrect behavior
   - no observable change from the prior bad output
7. User-reported result after flashing the later driver-padded `spi0` non-DMA retest:
   - nothing changed
   - no observable change from the prior bad output
8. Current build-only hardcoded-color retest keeps the same repo-visible `spi0` non-DMA `P0.17` settings and additionally:
   - uses the temporary `ws2812_spi.c` override that always emits one logical pure-green pixel
9. Rebuilt successfully with that hardcoded-color experiment.
10. Current UF2 ready to flash directly:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
11. This hardcoded-color retest has now been flashed directly from:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
12. Flash verification for the current hardcoded-color retest:
   - copied to `D:\charybdis_qwerty_left.uf2`
   - `D:` disappeared after copy, which is the expected nice!nano reboot behavior
13. User-reported result after flashing the hardcoded-color `spi0` non-DMA retest with `chain-length = <1>`:
   - same result
   - no observable change from the prior bad output
13. Current repo patch now prepared for the next same-pin full-chain retest is:
   - same `spi0` non-DMA `P0.17` path
   - same `4 MHz`, `0x70 / 0x40`, `GRB`, `reset-delay = <120>`, `NRF_DRIVE_H0H1`
   - `chain-length` restored from `1` to `29`
   - same build-only hardcoded pure-green override remains active in `ws2812_spi.c`
14. Rebuilt successfully with that full-chain hardcoded-color experiment.
15. Generated DTS for the current full-chain retest now confirms:
   - `spi0` is `compatible = "nordic,nrf-spi"`
   - `P0.17` is the active MOSI output
   - `chain-length = <29>`
   - `spi-max-frequency = <4000000>`
   - `spi-one-frame = <0x70>`
   - `spi-zero-frame = <0x40>`
   - `reset-delay = <120>`
   - `nordic,drive-mode = <0x3>` (`NRF_DRIVE_H0H1`)
16. Current UF2 ready to flash directly:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
17. This full-chain hardcoded-color retest has now been flashed directly from:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
18. Flash verification for the current full-chain hardcoded-color retest:
   - copied to `D:\charybdis_qwerty_left.uf2`
   - `D:` disappeared after copy, which is the expected nice!nano reboot behavior
19. User-reported result after flashing the full-chain hardcoded-color `spi0` non-DMA retest with `chain-length = <29>`:
   - same result
   - no observable change from the prior bad output
20. Current repo patch now prepared for the next same-pin controller-pinning retest is:
   - same `spi0` non-DMA `P0.17` path
   - same `chain-length = <29>`, `4 MHz`, `0x70 / 0x40`, `GRB`, `reset-delay = <120>`, `NRF_DRIVE_H0H1`
   - keeps the build-only hardcoded pure-green override active in `ws2812_spi.c`
   - adds a dummy `SPIM_SCK` assignment on unused `P0.03` so the Nordic SPI controller is fully pinned out while LED DIN remains on `P0.17`
21. Rebuilt successfully with that controller-pinning experiment.
22. Generated DTS for the current controller-pinning retest now confirms:
   - `spi0` is `compatible = "nordic,nrf-spi"`
   - `P0.03` is assigned as dummy `SPIM_SCK`
   - `P0.17` remains the active MOSI output
   - `chain-length = <29>`
   - `spi-max-frequency = <4000000>`
   - `spi-one-frame = <0x70>`
   - `spi-zero-frame = <0x40>`
   - `reset-delay = <120>`
   - `nordic,drive-mode = <0x3>` (`NRF_DRIVE_H0H1`)
23. Current UF2 ready to flash directly:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
24. This controller-pinning retest has now been flashed directly from:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
25. Flash verification for the current controller-pinning retest:
   - copied to `D:\charybdis_qwerty_left.uf2`
   - `D:` disappeared after copy, which is the expected nice!nano reboot behavior
26. User-reported result after flashing the controller-pinning retest:
   - new photo: `G:\Downloads\zmk testing\photo_2026-03-30_18-35-12.jpg`
   - behavior changed materially
   - only one LED is now on
   - that LED is still red, not the requested hardcoded green
27. Current repo patch now prepared for the next same-pin color-order retest is:
   - same `spi0` non-DMA `P0.17` path with dummy `SPIM_SCK` on `P0.03`
   - same `chain-length = <29>`, `4 MHz`, `0x70 / 0x40`, `reset-delay = <120>`, `NRF_DRIVE_H0H1`
   - same build-only hardcoded pure-green override remains active in `ws2812_spi.c`
   - changes `color-mapping` from `GRB` to `RGB` because the hardcoded green packet is currently being decoded as red

Rejected same-pin fallback:

1. `worldsemi,ws2812-gpio` is not usable in this local Zephyr version for `nice_nano_v2` / `nRF52840`.
2. Reason:
   - local `CONFIG_WS2812_STRIP_GPIO` depends on `SOC_SERIES_NRF51X`
   - local `ws2812_gpio.c` is hard-coded for `nRF51`

Local validation already performed:

1. In the latest built left DTS before this patch, the active LED path was on `spi0` / `P0.17`.
2. After removing the PMW3610 include from the left overlay, the current left build no longer carries leftover left-side `pmw3610` / trackball nodes in DTS.
3. The current `4 MHz`, `0x70`, `0x40`, `GRB` SPI encoding matches Zephyr's normal `ws2812-spi` baseline for nRF52-class setups, so recent testing shifted to isolation (`chain-length = 1`) instead of continuing to churn frame values.
4. The current repo patch now also tests two remaining software-side levers on the same pin:
   - longer latch time via `reset-delay = <120>;`
   - stronger output drive on `P0.17` via `NRF_DRIVE_H0H1`

Review focus for Claude:

1. Validate whether removing the PMW3610 include and left listener from the left overlay is sufficient repo-side design, given the shared keymap still references `&vtrackball`.
2. Validate the `P0.17` underglow configuration itself across both tested SPI transport choices (`spi3` and current `spi0`).
3. Treat the temp keymap edit as test scaffolding, not as a committed repo change.

Latest operational updates:

1. Trusted latest user result is still the photo at `G:\Downloads\zmk testing\photo_2026-03-30_18-35-12.jpg`:
   - only one LED is now on
   - that LED is still red
   - this is the current best-behaved same-pin result so far
2. The repo overlay was then patched for the next same-pin color-order retest:
   - still `spi0` non-DMA on `P0.17`
   - still dummy `SPIM_SCK` on `P0.03`
   - still `chain-length = <29>`, `4 MHz`, `0x70 / 0x40`, `reset-delay = <120>`, `NRF_DRIVE_H0H1`
   - same build-only hardcoded pure-green override remains active in `ws2812_spi.c`
   - `color-mapping` changed from `GRB` to `RGB`
3. Operational correction:
   - the current built `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2` was older than that `RGB` overlay patch
   - so the RGB-order retest still needs a fresh rebuild before it can be trusted or flashed
4. The RGB-order retest has now been rebuilt successfully from the patched overlay.
5. Generated DTS for the trusted current build now confirms:
   - `spi0` is `compatible = "nordic,nrf-spi"`
   - dummy `SPIM_SCK` remains on `P0.03`
   - LED DIN / MOSI remains on `P0.17`
   - `chain-length = <29>`
   - `spi-max-frequency = <4000000>`
   - `spi-one-frame = <0x70>`
   - `spi-zero-frame = <0x40>`
   - `color-mapping = <0x1 0x2 0x3>` (`RGB`)
   - `reset-delay = <120>`
6. Fresh trusted UF2 ready to flash directly:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
   - timestamp `2026-03-30 18:45:11`
7. Same-pin sanity check from a subagent:
   - flashing this `RGB` retest is the right next step
   - if it still fails, the next stronger same-pin software-only test would be a build-only driver patch that bypasses `color_mapping` entirely and writes raw channel byte order directly
8. New reset-only observation from `G:\Downloads\zmk testing\photo_2026-03-30_18-50-52.jpg`:
   - no new firmware was flashed
   - board was only reset, not mounted
   - visible LED state changed from one red LED to two red LEDs at different positions
   - this means the currently displayed LED pattern is not stable across resets and should not be treated as a deterministic output from the prepared `RGB` retest, which has still not been flashed
9. Newer reset-only observation from `G:\Downloads\zmk testing\photo_2026-03-30_18-52-20.jpg`:
   - again, no new firmware was flashed
   - after another plain reset, the visible state changed again
   - many red LEDs are now lit in a different pattern
   - this further confirms the currently displayed LEDs are drifting/latching across resets and are not a trustworthy proxy for the prepared `RGB` retest
10. Subagent interpretation of the escalating reset-only drift:
   - the dominant problem now looks like startup / signal-integrity instability on the `P0.17` path
   - plain color-order changes should not cause the number and positions of lit LEDs to drift across plain resets
   - therefore the more meaningful next same-pin test is deterministic startup behavior, not another passive observation of reset-only LED state
11. Current build-only next test prepared in `G:\zmk-build-left-20260330\zephyr\drivers\led_strip\ws2812_spi.c`:
   - still same `spi0` non-DMA path
   - still dummy `SPIM_SCK` on `P0.03`
   - still LED DIN / MOSI on `P0.17`
   - still full chain `29`
   - still hardcoded pure-green output in the local driver
   - adds explicit startup stabilization behavior on every update:
     - sends a full black frame first
     - waits at least `1000 us`
     - sends the target frame
     - waits again
     - resends the same target frame a second time
12. This deterministic-startup test rebuilt successfully.
13. Fresh trusted UF2 ready to flash directly:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
   - timestamp `2026-03-30 18:54:48`
14. This deterministic-startup test has now been flashed directly from:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
15. Flash verification for the current deterministic-startup test:
   - copied to `D:\charybdis_qwerty_left.uf2`
   - `D:` disappeared after copy, which is the expected nice!nano reboot behavior
16. Workspace divergence discovered after Claude produced:
   - `G:\zmk-build-left-20260330\CODEX_HANDOFF.md`
   - `G:\zmk-build-left-20260330\CODEX_CONTEXT.md`
17. Actual current build workspace files now match Claude's handoff, not the older staged note above:
   - `G:\zmk-build-left-20260330\zephyr\drivers\led_strip\ws2812_spi.c` is currently a build-only direct `NRF_PWM0` WS2812 driver on `P0.17`
   - `G:\zmk-build-left-20260330\module\boards\shields\charybdis-bt\charybdis_left.overlay` is currently `spi3` / `P0.17` / `GRB` / `chain-length = <29>`
   - `G:\zmk-build-left-20260330\config\charybdis_left.conf` is currently underglow-on-start, spectrum effect, brightness 100
18. Latest user result after the most recent flash is:
   - photo: `G:\Downloads\zmk testing\photo_2026-03-30_19-01-10.jpg`
   - a couple more LEDs are now lit
   - they are still red
19. Current source of truth for continuation:
   - `G:\Downloads\zmk testing\zmk-config\CLAUDE_REVIEW_NOTE.md`
   - `G:\zmk-build-left-20260330\CODEX_HANDOFF.md`
   - `G:\zmk-build-left-20260330\CODEX_CONTEXT.md`
20. Board mapping / conflict check performed against current local sources:
   - `G:\zmk-build-left-20260330\zmk\app\boards\arm\nice_nano\arduino_pro_micro_pins.dtsi` confirms `pro_micro` pin `2` maps to `gpio0 17` (`P0.17`)
   - current generated DTS confirms the active LED pinctrl is `spi3_default` / `spi3_sleep` on `P0.17`
   - current generated DTS kscan rows/cols do not use `pro_micro 2`
   - current generated DTS does not show another active consumer of `P0.17` besides the LED path
21. Subagent sanity check on the next same-pin test:
   - a slow `HIGH` / `LOW` GPIO blink on `P0.17` is not a meaningful WS2812/SK6812 DIN visibility test
   - stronger next step is to bypass ZMK color state and color mapping entirely and emit known raw byte-slot patterns
22. Current build-only test now prepared in `G:\zmk-build-left-20260330\zephyr\drivers\led_strip\ws2812_spi.c`:
   - keeps the direct `NRF_PWM0` driver on `P0.17`
   - ignores ZMK `update_rgb` payloads
   - starts a background thread at driver init
   - continuously cycles the full 29-LED strip through:
     - all off for `400 ms`
     - raw byte slot 0 high only (`FF 00 00`) for `1500 ms`
     - all off for `400 ms`
     - raw byte slot 1 high only (`00 FF 00`) for `1500 ms`
     - all off for `400 ms`
     - raw byte slot 2 high only (`00 00 FF`) for `1500 ms`
23. This raw-byte-slot test rebuilt successfully.
24. Fresh trusted UF2 ready to flash directly:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
   - timestamp `2026-03-30 19:31:56`
25. This raw-byte-slot test has now been flashed directly from:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
26. Flash verification for the current raw-byte-slot test:
   - copied to `D:\charybdis_qwerty_left.uf2`
   - `D:` disappeared after copy, which is the expected nice!nano reboot behavior
27. New regression reported by user after the build-only PWM / raw-byte driver work:
   - keyboard is no longer being recognized properly
28. Immediate recovery plan:
   - restore `G:\zmk-build-left-20260330\zephyr\drivers\led_strip\ws2812_spi.c` to stock from git
   - keep the current build workspace overlay/conf unchanged for now
   - rebuild and flash that recovery firmware before doing any more LED experiments
29. Recovery action completed:
   - restored `G:\zmk-build-left-20260330\zephyr\drivers\led_strip\ws2812_spi.c` to stock from git
   - rebuilt successfully using `G:\zmk-build-left-20260330\rebuild.ps1`
   - fresh recovery UF2: `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
   - timestamp `2026-03-30 19:45:03`
30. Recovery firmware has now been flashed directly:
   - copied to `D:\charybdis_qwerty_left.uf2`
   - `D:` disappeared after copy, which is the expected nice!nano reboot behavior
31. Current purpose of this firmware:
   - restore normal keyboard recognition behavior by removing the build-only PWM/raw-byte driver experiments
   - LED behavior after this flash is secondary until keyboard recognition is confirmed healthy again
32. Recovered crucial context from another Codex thread:
   - previously best-working transport is not `P0.17`
   - recovered working combination is:
     - DIN on `D3 / P0.20`
     - `spi3`
     - dummy `SCK = D0 / P0.08`
     - dummy `MISO = D1 / P0.06`
     - high-drive enabled
     - stock `ws2812_spi` driver with a build-only two-byte raw zero lead-in before each encoded frame
   - that two-byte lead-in is what reportedly unlocked blue and realigned the payload on the strip
33. Local board/mapping validation for the recovered transport:
   - `G:\zmk-build-left-20260330\zmk\app\boards\arm\nice_nano\arduino_pro_micro_pins.dtsi` confirms:
     - `D0 -> P0.08`
     - `D1 -> P0.06`
     - `D3 -> P0.20`
   - current left keyscan does not use `D0` or `D1`
   - current generated DTS has `uart0` and `i2c0` disabled, so those dummy pins are available for `spi3`
34. Current build workspace has now been patched to that recovered working transport:
   - `G:\zmk-build-left-20260330\module\boards\shields\charybdis-bt\charybdis_left.overlay`
   - `spi3` with `SCK=P0.08`, `MISO=P0.06`, `MOSI/DIN=P0.20`
   - `nordic,drive-mode = <NRF_DRIVE_H0H1>`
   - `4 MHz`, `0x70 / 0x40`, `GRB`, `chain-length = <29>`
35. Current build-only driver shim has now been patched to match that recovered transport:
   - `G:\zmk-build-left-20260330\zephyr\drivers\led_strip\ws2812_spi.c`
   - stock SPI path restored
   - prepends exactly two raw `0x00` bytes before the encoded pixel stream
36. The recovered full-color validation build rebuilt successfully.
37. Generated DTS for the trusted current validation build confirms:
   - `spi3` active and `compatible = "nordic,nrf-spim"`
   - `spi-max-frequency = <4000000>`
   - `chain-length = <29>`
   - `spi-one-frame = <0x70>`
   - `spi-zero-frame = <0x40>`
   - `color-mapping = <0x2 0x1 0x3>` (`GRB`)
   - `psels = <0x40008>, <0x60006>, <0x50014>` (`SCK=P0.08`, `MISO=P0.06`, `MOSI=P0.20`)
   - `nordic,drive-mode = <0x3>` (`H0H1`)
38. Fresh trusted UF2 ready to flash directly:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
   - timestamp `2026-03-30 20:07:33`
39. User clarified the physical DIN wire is on the pin marked `17`, not the `CS/P0.20` pad.
40. Current build workspace has been repatched to replicate the recovered working transport on `P0.17` only:
   - `G:\zmk-build-left-20260330\module\boards\shields\charybdis-bt\charybdis_left.overlay`
   - still `spi3`
   - still dummy `SCK=P0.08` and `MISO=P0.06`
   - still `nordic,drive-mode = <NRF_DRIVE_H0H1>`
   - still `4 MHz`, `0x70 / 0x40`, `GRB`, `chain-length = <29>`
   - only `MOSI/DIN` moved from `P0.20` to `P0.17`
41. The `P0.17` replica transport rebuilt successfully.
42. Generated DTS for the current `P0.17` validation build confirms:
   - `spi3` active and `compatible = "nordic,nrf-spim"`
   - `spi-max-frequency = <4000000>`
   - `chain-length = <29>`
   - `spi-one-frame = <0x70>`
   - `spi-zero-frame = <0x40>`
   - `color-mapping = <0x2 0x1 0x3>` (`GRB`)
   - `psels = <0x40008>, <0x60006>, <0x50011>` (`SCK=P0.08`, `MISO=P0.06`, `MOSI=P0.17`)
   - `nordic,drive-mode = <0x3>` (`H0H1`)
43. Fresh trusted `P0.17` UF2 ready to flash directly:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
   - timestamp `2026-03-30 20:12:01`
44. The trusted `P0.17` replica build was flashed directly from:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
   - copied to `D:\charybdis_qwerty_left.uf2`
   - `D:` disappeared after copy, which is the expected nice!nano reboot behavior
45. Result from `G:\Downloads\zmk testing\photo_2026-03-30_20-19-54.jpg`:
   - many LEDs now decode as visibly different colors
   - after a few seconds, the strip either shuts off entirely or changes to a different color/pattern
   - this is materially different from the old static red/partial corruption and confirms the current transport path is substantially more correct
46. Current conclusion from the hardware evidence:
   - this is not a simple wrong-pin diagnosis
   - the same transport style works meaningfully better on more than one MOSI pin
   - the remaining problem is now more likely runtime/state/stream stability on top of an otherwise valid transport path
47. Source inspection of `G:\zmk-build-left-20260330\zmk\app\src\rgb_underglow.c` confirms:
   - underglow state is restored from settings via `SETTINGS_STATIC_HANDLER_DEFINE(rgb_underglow, "rgb/underglow", ...)`
   - underglow state is persisted with `settings_save_one("rgb/underglow/state", &state, sizeof(state))`
   - `CONFIG_ZMK_RGB_UNDERGLOW_AUTO_OFF_IDLE` subscribes to `zmk_activity_state_changed` and can blank LEDs on idle
48. Active shield conf for the current validation build has now been tightened for a stable test:
   - `G:\zmk-build-left-20260330\module\boards\shields\charybdis-bt\charybdis_left.conf`
   - keep solid effect and current startup color
   - disable `CONFIG_ZMK_RGB_UNDERGLOW_AUTO_OFF_IDLE` so the strip does not shut off during validation
49. Planned clean validation sequence from here:
   - flash one-shot settings reset from `G:\zmk-build-left-20260330\build-settings-reset\zephyr\zmk.uf2`
   - immediately flash the fresh rebuilt static transport test from `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
50. The static `P0.17` validation build has now been rebuilt with idle blanking disabled.
51. Merged `.config` for the current build confirms:
   - `CONFIG_ZMK_RGB_UNDERGLOW_EFF_START=0`
   - `CONFIG_ZMK_RGB_UNDERGLOW_HUE_START=240`
   - `CONFIG_ZMK_RGB_UNDERGLOW_SAT_START=100`
   - `CONFIG_ZMK_RGB_UNDERGLOW_BRT_START=100`
   - `CONFIG_ZMK_RGB_UNDERGLOW_ON_START=y`
   - `# CONFIG_ZMK_RGB_UNDERGLOW_AUTO_OFF_IDLE is not set`
52. Fresh static UF2 ready to flash after the settings reset:
   - `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
   - timestamp `2026-03-30 20:24:38`
53. One-shot settings reset has now been flashed from:
   - `G:\zmk-build-left-20260330\build-settings-reset\zephyr\zmk.uf2`
   - copied to `D:\settings_reset.uf2`
   - `D:` disappeared after copy, which is the expected reboot behavior
54. The second step is still pending:
   - re-enter bootloader and flash `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
   - this is the fresh static `P0.17` validation build with idle blanking disabled
55. The second step has now been completed:
   - flashed `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`
   - copied to `D:\charybdis_qwerty_left.uf2`
   - `D:` disappeared after copy, which is the expected reboot behavior
56. Current trusted test state on hardware:
   - saved settings were wiped immediately before the flash
   - current firmware is the fresh static `P0.17` transport test
   - idle blanking is disabled
   - startup effect is solid with the configured startup color
57. Video evidence from `G:\Downloads\zmk testing\video_2026-03-30_20-36-07.mp4`:
   - local `ffmpeg` was installed to inspect the clip
   - extracted frames show a substantially valid multicolor decode across much of the strip
   - the pattern is not a static single-color result
   - several LEDs on the right side visibly shift between blue/purple/red over the clip
   - sampled frames do not show a total shutoff, but they do show that the output is still not holding a fully stable intended solid color
