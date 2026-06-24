# ESP32_HUB12 — Internal documentation

Auto-loaded with `CLAUDE.md`. How this project works, for anyone editing it.
For requirements/status see `reqs.md`; for the outward overview see `README.md`.

> Status: hardware understood and verified; architecture decided (**1× ESP32-S3
> parallel, 4-bit grayscale** — see `reqs/R3.md`); no firmware yet. Controller board
> chosen: **Waveshare ESP32-S3-ETH**. Full verified research compilation:
> [`docs/overview.md`](docs/overview.md).

## Goal in one line
Replace the BX-5A4 store-and-play controller and drive the existing P10 HUB12 mono
(amber) wall as a live, network-streamed lighting fixture from an ESP32+Ethernet.

## The wall (verified — `docs/overview.md` §1–§4)
- **24 modules**, P10(1R)-V701C, 10 mm pitch, single-color **amber** (~605–620 nm,
  AlGaInP; labelled "1R" but emits orange/amber — normal, not a defect).
- Each module 320×160 mm, **32×16 px = 512 px**, HUB12 interface, **1/4 scan**.
- Topology: **8 parallel chains × 3 modules in series**. Chain = 96×16 = 1 536 px;
  full wall = **96×128 = 12 288 px**. Keep this topology — see Conventions.
- Module electronics: **16× 74HC595** (column shift registers) + **74HC245** (HUB12
  bus buffer) — both observed on-board — plus a **74HC138 row decoder + 74HC04
  inverter + 4× IRF7314 P-MOSFETs** for 1/4-scan row drive (from the V701C schematic).
- It is a **dumb shift-register panel**: no constant-current driver, so **all
  brightness/grayscale is time-domain** (OE PWM / BCM at the controller). LED current
  is fixed by on-board resistors.

## How driving works
- Per scan phase: shift column data into the 595 chain, set row group via the 138
  (A/B select), pulse latch (STB), gate output with OE. 1/4 scan = 4 phases/frame.
- **bits/frame per chain = 1 536**, independent of how many chains run in parallel —
  parallel data lines shift one bit into all chains per clock edge. This is what makes
  a single ESP32 driving all 8 chains feasible at one chain's refresh.
- 1-bit refresh ceiling ≈ f_clk / 1 536 (~kHz at 10 MHz); old HC/DIP modules clock
  cleanly only to ~10–15 MHz. Grayscale via BCM trades refresh ≈ ÷(2^N−1). Practical
  operating point: **1-bit or 4-bit**. (See `docs/overview.md` §7.)
- **Two distinct rates:** *content/frame rate* (how often new frames arrive over the
  network, target ≥44 Hz) vs *panel refresh* (re-scan rate for flicker, target
  ≥100 Hz). Don't conflate them.

## Level shifting (mandatory — `docs/overview.md` §6)
ESP32 GPIO is 3.3 V; the panel's 74HC logic at 5 V needs V_IH ≥ 3.5 V. Insert a
**74HCT245** (TTL thresholds, V_IH ≥ 2.0 V) per chain between ESP32 and HUB12 in;
power from the panel 5 V rail with **common ground**. Do not substitute an HC part.

## Architecture (decided — 2026-06-24)
**Single ESP32-S3 driving all 8 chains in parallel, 4-bit grayscale.** One node:
shared CLK / LAT / OE / A / B plus 8 data lines (one per chain), all driven from the
S3's **LCD_CAM** peripheral via DMA. Parallel data costs no extra clocks, so the
whole wall refreshes at one chain's rate. 4-bit BCM gives 16 levels at a few-hundred-Hz
refresh — well clear of flicker (`docs/overview.md` §7). Needs forked/custom
parallel-output firmware (no stock library does single-node parallel HUB12).
Trade-offs and the rejected 8× ESP32 option: [`reqs/R3.md`](reqs/R3.md).

### Memory — internal SRAM is enough at 4-bit (PSRAM not required)
The logical framebuffer is tiny (12,288 px × 4 bit = **6 KB**). The driver's **DMA scan
buffer** is the larger structure — expanded bit-planes shifted out by LCD_CAM: ~1,536
output words/plane (384 clocks/scan-phase × 4 phases), ~3 KB/plane. At 4-bit,
double-buffered, that's ~**24 KB** (linear planes) to ~**92 KB** (binary-weighted) — well
within the **512 KB internal SRAM**, which is also the preferred target for DMA (lower
latency, no PSRAM cache contention). PSRAM would only matter at much higher depth
(8-bit ≈ 765 KB > SRAM). So the board's 8 MB octal PSRAM is **available but unused** by
the driver at our operating point — yet GPIO33–37 stay reserved regardless, because they
are physically wired to that PSRAM (see below).

Library landscape (verified): DMD32 (ESP32 HUB12) and mrcodetastic's HUB75-DMA are
reference/starting points only — HUB12 support is rough and neither does single-node
parallel output, so the I2S/LCD_CAM output path will be forked.

## Controller board — Waveshare ESP32-S3-ETH
Pinout analyzed from [`docs/waveshare_esp32-s3-ETH_pinout.webp`](docs/waveshare_esp32-s3-ETH_pinout.webp);
memory/peripheral facts cross-checked against the Waveshare wiki and Espressif docs.

- **SoC:** ESP32-S3 (R8), 16 MB flash, **8 MB PSRAM in octal mode @ 80 MHz**. Note: the
  4-bit driver runs entirely from internal SRAM (see Memory above) — the PSRAM is spare
  here, but its octal wiring is what reserves GPIO33–37.
- **Ethernet:** onboard **W5500 over SPI** (not RMII/LAN8720). This settles R1's PHY
  question. Fixed SPI pins: **MOSI=11, MISO=12, SCLK=13, CS=14, INT=10, RST=9.**
  W5500 SPI throughput (~tens of Mbps) easily covers the wall's data rate (24 ArtNet
  universes ≈ 12 KB/frame ≈ 4–5 Mbps at 44 fps).
- **microSD (SPI):** MOSI=6, MISO=5, CLK=7, CS=4 — reserved by the slot; free only if SD unused.
- **USB native:** D−=GPIO19, D+=GPIO20 (programming / JTAG / CDC log).
- **UART0 console:** TX=GPIO43, RX=GPIO44.

### ⚠️ GPIO33–37 are NOT usable
Because this is an **R8 (octal PSRAM)** part, GPIO33–37 are wired to SPIIO4–7/DQS for
the PSRAM. Espressif: configuring them for I/O typically crashes firmware. Waveshare's
header silk and "available GPIO" list show 33–37 anyway — **ignore that; treat them as
reserved.** (Espressif ESP32-S3 GPIO docs; Waveshare ESP32-S3-ETH wiki.)

### Pin budget for the parallel driver
Need **13 signals**: 8 data (one/chain) + CLK + LAT(STB) + OE + A + B (1/4-scan row select).

Header GPIOs after removing reserved/peripheral pins:

| Class | GPIOs | Use for driver? |
|---|---|---|
| **Clean** | 1, 2, 15, 16, 17, 18, 21, 38, 39, 40, 41, 42, 47, 48 | **Yes — 14 pins, 13 needed** |
| Strapping (care) | 3 (ok after boot), 0 / 45 / 46 (avoid) | 3 as a spare only |
| Reserved — octal PSRAM | 33, 34, 35, 36, 37 | No |
| Reserved — W5500 ETH | 9, 10, 11, 12, 13, 14 | No (needed for R1) |
| Reserved — microSD | 4, 5, 6, 7 | No (unless SD dropped) |
| Reserved — USB/UART | 19, 20, 43, 44 | Avoid (keep prog/log) |

The 14 clean pins cover all 13 signals with one spare (GPIO3 as a further fallback).
LCD_CAM output is routed through the GPIO matrix, so any clean pin maps to any signal —
no fixed-pin constraint on the driver side. Exact assignment is set when firmware starts.

## Network protocols (two inputs)
Both feed the same framebuffer over the onboard W5500 (SPI Ethernet):
- **ArtNet** (R6) — for stock lighting gear. 24 universes (1 ch/px), ~44 Hz, ~11 Mbps,
  2,400 pps. Convenient, interoperable, padded/heavier.
- **Packed custom UDP** (R9) — for high refresh from an external renderer. 4 bit/px,
  **8 packets/frame (one per chain, 768 B data each)**, all sub-MTU to avoid IP
  fragmentation (W5500 can't reassemble). ~5.3 Mbps @ 100 Hz, 800 pps. Optionally fed
  by an external **NDI → protocol converter** (R10). Full layout: [`docs/protocol.md`](docs/protocol.md).

Why per-chain packets: maps 1:1 onto the 8-chain DMA layout, enables partial updates,
and confines a lost packet to one chain for one frame.

## Conventions
- **Never rewire the 8×3 topology** into one long series chain — it multiplies
  bits/frame ~8× and collapses refresh.
- Treat the wall as a **single-color intensity matrix** (1 ch/px) in any lighting
  software — the hardware cannot show color regardless of input.
- Record requirements/decisions in `reqs.md` before coding; cite IDs in commits.

## Reference
- [`docs/overview.md`](docs/overview.md) — full verified research compilation
  (geometry, color physics, electronics, controller, ArtNet path, performance math,
  sourcing, rejected alternatives, open items). §0 holds the 2026-06-24 verification.
- Key external sources: led-kento V701C schematic; AusChristmasLighting P10 wiki;
  Qudor-Engineer/DMD32; mrcodetastic/ESP32-HUB75-MatrixPanel-DMA.
