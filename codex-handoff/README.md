This directory is a durable handoff for the Charybdis RGB debugging session.

What is here:

- `../CLAUDE_REVIEW_NOTE.md`
  - running operational log of hardware tests, flashes, and conclusions
- `artifacts/CODEX_HANDOFF.md`
  - earlier Codex handoff instructions
- `artifacts/CODEX_CONTEXT.md`
  - earlier Codex test history/context
- `artifacts/rebuild.ps1`
  - local rebuild script used to produce the UF2 files in the temp workspace
- `snapshots/build_workspace_charybdis_left.overlay`
  - current active build-workspace overlay used for the latest trusted `P0.17` transport test
- `snapshots/build_workspace_charybdis_left.conf`
  - current active build-workspace shield conf used for the latest trusted static-color test
- `snapshots/repo_charybdis_left.overlay`
  - current config-repo overlay snapshot
- `snapshots/repo_charybdis_left.conf`
  - current config-repo shield conf snapshot
- `patches/zephyr_ws2812_spi.diff`
  - exact Zephyr driver diff for the two-byte lead-in change in `ws2812_spi.c`
- `patches/build_keymap_vs_repo.diff`
  - exact diff between the repo keymap and the build-workspace keymap used for left-only builds
- `patches/repo_overlay_dirty.diff`
  - exact current dirty diff of the repo overlay

Current trusted test path:

- `spi3`
- dummy `SCK=P0.08`
- dummy `MISO=P0.06`
- `MOSI/DIN=P0.17`
- `nordic,drive-mode = <NRF_DRIVE_H0H1>`
- Zephyr `ws2812_spi` driver patched to prepend two raw zero bytes before each encoded frame

Current trusted flash sequence:

1. Flash `G:\zmk-build-left-20260330\build-settings-reset\zephyr\zmk.uf2`
2. Flash `G:\zmk-build-left-20260330\build-left\zephyr\zmk.uf2`

Important:

- The temp build workspace `G:\zmk-build-left-20260330` is not itself the pushed repo.
- The Zephyr driver change lives outside this config repo, so the `patches/zephyr_ws2812_spi.diff` file is the durable record of that change.
