# ESP32_HUB12

Drive a secondhand **P10 HUB12 mono (amber) LED wall** from a single **ESP32-S3 with
Ethernet** as a live, network-streamed lighting fixture — for interactive lighting
installations. Replaces the panels' original BX-5A4 store-and-play controller.

## Overview
The wall is 24× P10(1R)-V701C modules (single-color amber, 32×16 px each) wired as
8 parallel chains of 3 — a 96×128 px, 12 288-pixel surface. The panels are "dumb"
74HC595 shift-register modules with no on-board grayscale, so all intensity is done in
the time domain by the controller. The original controller can only store and replay
sign content; this project streams frames live over Ethernet instead, so the wall can
be driven in real time like any lighting fixture.

A single **ESP32-S3 (Waveshare ESP32-S3-ETH)** drives all 8 chains in parallel via its
LCD_CAM peripheral and DMA, rendering **4-bit grayscale** (BCM). Frames arrive over the
onboard **W5500 Ethernet** through either of two inputs:

- **ArtNet** — for interop with stock lighting gear (24 universes, ~44 Hz).
- **Packed custom UDP** — a dense 4-bit protocol (8 per-chain packets/frame) for
  high-refresh content (≥100 Hz), optionally fed by an external **NDI → protocol**
  converter running on a host computer.

## Status
Early — design phase. Hardware is understood and the technical assumptions verified;
firmware is not yet written. Architecture is **decided**: single ESP32-S3 parallel,
4-bit grayscale, W5500 Ethernet, dual input protocols. See [`reqs.md`](reqs.md) for
requirements and [`todo.md`](todo.md) for the plan.

## Hardware
- The 24× P10(1R)-V701C HUB12 wall in its original 8×3 wiring (**do not rewire** —
  it would multiply bits/frame ~8× and collapse refresh).
- **Waveshare ESP32-S3-ETH** controller (ESP32-S3 R8, onboard W5500 SPI Ethernet).
- **74HCT245 level shifter per chain** between the ESP32-S3 (3.3 V) and the panels'
  5 V HC logic — mandatory; the panel inputs won't reliably read 3.3 V.
- Buffered, length-matched fan-out of the shared control lines (CLK/LAT/OE/A/B) to all
  8 chains.
- A 5 V supply sized for the LED load.

Note: the board's octal PSRAM reserves GPIO33–37; the 4-bit driver itself runs from
internal SRAM. Pin budget and full pinout analysis are in [`info.md`](info.md).

## Installation
```sh
# TODO: toolchain (PlatformIO), pin map, wiring diagram, and flashing steps.
```

## Usage
```sh
# TODO: example — stream ArtNet or the custom protocol to the node's IP.
```

## How it works
See [`info.md`](info.md) for the architecture, board pinout, and driving model;
[`docs/protocol.md`](docs/protocol.md) for the custom UDP protocol; and
[`docs/overview.md`](docs/overview.md) for the full verified technical background
(geometry, color physics, electronics, driving math, performance, sourcing).

## License
TBD
