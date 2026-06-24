# Todos

Work items. Reference the ID they advance.

## Decisions to lock first (block firmware)
- [x] Decide topology: **1× ESP32-S3 parallel** (Waveshare ESP32-S3-ETH). (R3)
- [x] Decide target bit depth: **4-bit** (1-bit = native fallback). (R2/R7)
- [x] Decide Ethernet PHY: **W5500 over SPI** (onboard the chosen board). (R1)
- [x] Decide wire protocol: **both** ArtNet (R6) + packed custom UDP (R9). (R1)

## Hardware verification (on the actual modules + board)
- [ ] Confirm HUB12 pinout and clock polarity (`ESP32_INVERT_CLK`). (R4)
- [ ] Confirm practical max clock before ghosting (~10–15 MHz). (R3)
- [ ] Confirm LED package (DIP346 vs DIP546) and rough peak nm. (sourcing)
- [ ] Build/validate one 74HCT245 level-shifter stage, common ground. (R5)
- [ ] Assign the 13 driver signals to clean GPIOs (see info.md) and confirm GPIO33–37
      are unusable on the actual board (expected, R8 PSRAM). (R3)
- [ ] Design buffered, length-matched fan-out of shared CLK/LAT/OE/A/B to all 8 chains. (R4)

## Firmware — MUST path (1× ESP32-S3, LCD_CAM parallel)
- [ ] Bring up one chain (3 modules) at 1-bit via LCD_CAM DMA. (R2, R4)
- [ ] Extend to all 8 chains in parallel (8 data lines, shared control). (R3, R4)
- [ ] Add W5500 Ethernet link + live frame ingest into the framebuffer. (R1)
- [ ] Hit and measure ≥44 Hz content rate to the full wall. (R3)

## Firmware — SHOULD path
- [ ] ArtNet input mapped to the framebuffer (512 px/universe, 24 universes). (R6)
- [ ] BCM grayscale (start 4-bit); verify ≥100 Hz refresh holds. (R7, R8)
- [ ] Packed custom UDP input: 8 per-chain packets → framebuffer, latch/anti-tearing. (R9)
- [ ] Finalize protocol open questions in docs/protocol.md (latch policy, port, loss). (R9)

## External tooling
- [ ] NDI → custom-protocol converter (host app): NDI in, 96×128×4-bit dither out. (R10)

## Project setup
- [x] Verify overview.md and relocate to docs/. 
- [x] Establish reqs.md / todo.md / info.md / README.md per LLMbootstrap.
- [ ] Pick toolchain (PlatformIO vs Arduino IDE) and scaffold the firmware repo.
