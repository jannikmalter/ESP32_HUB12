# ESP32_HUB12 — Requirements

Status: Active · Updated: 2026-06-24

Drive a 96×128 px secondhand P10(1R)-V701C HUB12 mono (amber) LED wall — 8 parallel
chains × 3 modules — from a single **ESP32-S3 (Waveshare ESP32-S3-ETH)** with onboard
W5500 Ethernet, for live, interactive lighting. Replaces the original BX-5A4
store-and-play controller. Architecture decided: **1× ESP32-S3 parallel, 4-bit
grayscale** (`reqs/R3.md`). Hardware background: `docs/overview.md`; board/pinout
analysis: `info.md`.

## Goals
Why this exists. Everything below traces to one of these.

- **G1** — Drive the full P10 HUB12 wall from an ESP32+Ethernet node as a live,
  network-streamed lighting fixture (no store-and-play).
- **G2** — Fit live, interactive installation use: low latency, standard lighting
  control, flicker-free output.

## Out of scope
What this deliberately will *not* do (stops scope creep).

- Reusing or reverse-engineering the BX-5A4 controller or its serial-over-TCP path.
- Color or RGB output — the panels are single-color amber; the hardware cannot show color.
- Rewiring the array out of its 8×3 parallel/series topology (would collapse refresh).
- Onbon authoring tools (LedShowTW / LedshowYQ) or store-and-forward sign content.

## Requirements
One row each. Use "shall". `Type`: F=function, Q=quality, C=constraint.

| ID  | Type | Requirement                                                                 | Pri | Goal | Done |
|-----|------|-----------------------------------------------------------------------------|-----|------|------|
| R1  | F    | The system shall stream frame content to the wall live over TCP/IP Ethernet. | M   | G1   | ☐    |
| R2  | F    | The system shall set each pixel independently on/off (1-bit).               | M   | G1   | ☐    |
| R3  | Q    | The system shall sustain ≥44 Hz content frame rate to the full wall.        | M   | G2   | ☐    |
| R4  | C    | The system shall drive the existing P10(1R)-V701C HUB12 modules unmodified, in their 8 parallel × 3 series topology. | M | G1 | ☐ |
| R5  | C    | The ESP32 outputs shall be level-shifted to 5 V via 74HCT245 (TTL-threshold) per chain, common ground. | M | G1 | ☐ |
| R6  | F    | The system shall accept ArtNet as a control input.                          | S   | G2   | ☐    |
| R7  | F    | The system shall render grayscale via BCM (binary code modulation).         | S   | G2   | ☐    |
| R8  | Q    | The panel refresh shall be ≥100 Hz (flicker-free) at the chosen bit depth.  | S   | G2   | ☐    |
| R9  | F    | The system shall accept a packed custom UDP protocol (4-bit/px, per-chain packets) for high-refresh content. | S | G2 | ☐ |
| R10 | F    | An external host converter shall ingest an NDI stream and emit the R9 protocol. | C | G2 | ☐ |

## Bugs
Deviations from a requirement. `Ref` = the requirement broken.

| ID | Bug | Ref | Sev | Done |
|----|-----|-----|-----|------|
| _(none yet)_ | | | | |

---
*Pri:* M/S/C (must/should/could). *Sev:* Hi/Md/Lo. IDs are permanent — never reuse.
*Detail files: `reqs/<ID>.md` (e.g. `reqs/R3.md`, `reqs/B1.md`).*
