# ESP32_HUB12

Drive a secondhand **P10 HUB12 mono (amber) LED wall** from an **ESP32 with
Ethernet** as a live, network-streamed lighting fixture — for interactive lighting
installations. Replaces the panels' original BX-5A4 store-and-play controller.

## Overview
The wall is 24× P10(1R)-V701C modules (single-color amber, 32×16 px each) wired as
8 parallel chains of 3 — a 96×128 px, 12 288-pixel surface. The panels are "dumb"
74HC595 shift-register modules with no on-board grayscale, so all intensity is
done in the time domain by the controller. The original controller can only store
and replay sign content; this project streams frames live over Ethernet instead,
so the wall can be driven in real time (e.g. via ArtNet) like any lighting fixture.

## Status
Early. Hardware is understood and the technical assumptions are verified; firmware
is not yet written. Two architecture decisions are still open (single ESP32-S3
parallel vs. 8× ESP32; 1-bit vs. 4-bit grayscale). See `reqs.md` and `todo.md`.

## Requirements
- The 24× P10(1R)-V701C HUB12 wall in its original 8×3 wiring (do not rewire).
- An ESP32 / ESP32-S3 with Ethernet (LAN8720 or W5500 — TBD).
- **74HCT245 level shifter(s)** between ESP32 (3.3 V) and the panels' 5 V logic —
  mandatory, the panel's HC inputs won't reliably read 3.3 V.
- A 5 V supply sized for the LED load.

## Installation
```sh
# TODO: toolchain (PlatformIO/Arduino), wiring diagram, and flashing steps
# once the architecture decision is made.
```

## Usage
```sh
# TODO: example — stream frames / ArtNet to the node's IP.
```

## How it works
See [`info.md`](info.md) for the architecture and [`docs/overview.md`](docs/overview.md)
for the full verified technical background (geometry, color, electronics, driving
math, ArtNet path, sourcing).

## License
TBD
