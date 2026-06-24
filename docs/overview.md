# P10 LED Panel Array — Technical Overview

Compilation of established and derived facts for the secondhand P10 wall and its
intended ArtNet integration.

**Confidence tags:** `[stated]` given directly or read off the hardware ·
`[observed]` from visual/photo inspection · `[derived]` inferred from specs/physics ·
`[verify]` open / unconfirmed.

> **Reference document.** Bulky background; linked from `info.md`, read on demand.
> Requirements and status live in `reqs.md`, not here.

---

## 0. Verification notes (checked 2026-06-24)

External cross-check of the decision-critical claims below. Findings folded into
`info.md`; original text kept intact for traceability.

- **Panel V701C — confirmed**, with one addition. The published V701C schematic
  carries not only the observed **16× 74HC595 + 74HC245**, but also a **74HC138
  row decoder, 74HC04 inverter, and 4× IRF7314 P-channel MOSFETs** driving the
  1/4-scan row groups. §4 lists only the two ICs read off the board; the row-scan
  stage is also present and matters for the driver (A/B select + OE timing).
  Sources: led-kento V701C schematic; AusChristmasLighting P10 wiki.
- **44 Hz / ArtNet — confirmed.** 44 Hz is the ceiling for a *full* 512-channel
  universe (~22.7 ms/packet); partial universes go faster. Confirms §7: the 44 fps
  figure is a **content/motion rate**, independent of panel flicker-refresh.
  Sources: open-lighting list; Y-Link DMX timing guide.
- **Level shifting (HC vs HCT) — confirmed** as stated in §4/§6. HC at Vcc=5 V
  needs V_IH ≥ 3.5 V; HCT ≥ 2.0 V accepts 3.3 V. Mandatory, not advisory.
- **Driver libraries — §6 is optimistic.** DMD32 (Qudor-Engineer) targets ESP32
  HUB12 but has documented HUB12 defects; mrcodetastic's library is HUB75-first
  (HUB12 only via an open discussion/fork). Treat "drive it directly" as a
  *starting point*; the realistic path — especially the single-ESP32 parallel
  option — is forked/custom firmware. Sources: Qudor-Engineer/DMD32 issues;
  mrcodetastic discussion #526.
- **Still open** (unchanged from §10): LED package 346 vs 546, exact peak nm,
  clock polarity on these modules, practical max clock, and the two architecture
  decisions (8× ESP32 vs 1× ESP32-S3 parallel; 1-bit vs 4-bit depth).

---

## 1. Array geometry

| Property | Value | Src |
|---|---|---|
| Modules total | 24 | `[stated]` |
| Topology | 8 parallel chains × 3 modules in series | `[stated]` |
| Module size | 320 × 160 mm, 32 × 16 px = 512 px | `[stated]` |
| Chain | 96 × 16 px = 1 536 px | `[derived]` |
| Full wall | 96 × 128 px = **12 288 px** | `[derived]` |
| ArtNet mapping | 512 px = 1 universe @ 1 ch/px → 3 universes/chain → **24 universes** | `[derived]` |

The 8-parallel / 3-series layout is deliberate: it caps series length at 3 to keep
per-chain shift load low. Re-wiring into one 24-long series chain would multiply
bits/frame ~8× and collapse refresh — do not do it.

## 2. Module identity

| Property | Value | Src |
|---|---|---|
| Model | P10(1R)-V701C | `[stated]` |
| Pitch | 10 mm | `[stated]` |
| Color SKU | 1R (single-color, red **category**) | `[stated]` |
| Usage | outdoor capable, typ. IP65 | `[stated]` |
| Interface | HUB12 (single data line) | `[stated]` |
| Scan | 1/4 | `[stated]` |
| LED package | DIP oval through-hole — DIP346 or **DIP546** | `[observed]` |

The model code fixes PCB, pinout, scan, and LED family but **not** the LED bin.
"1R" is an SKU class distinguishing it from 1G/1B/1W/1Y/dual/RGB; it carries no
wavelength guarantee. `[derived]`

`[verify]` Exact LED package (346 vs 546) — measure lamp body or read silk/spec.
"DIP254" is not a standard designation; the real options are DIP346/DIP546.

## 3. Emission color

| Property | Value | Src |
|---|---|---|
| Chip chemistry | AlGaInP (on GaAs) | `[derived]` |
| Observed hue | orange-amber, **not** red | `[observed]` |
| Estimated peak | ~600–620 nm | `[observed/derived]` |
| Exact peak | unknown — needs datasheet/spectrometer | `[verify]` |

Reference mapping of AlGaInP single-color display LEDs:

| peak | perceived |
|---|---|
| ~660 nm | deep red (also AlGaAs) |
| ~640 nm | red |
| ~625 nm | red-orange |
| ~615 nm | orange-red |
| ~605 nm | orange |
| ~590 nm | amber |

An orange/amber panel labelled "1R" is normal physics, not a defect or mislabel:
the label asserts the red SKU, the junction emits ~605–620 nm. Excluded explanations:

- **Phosphor aging** — N/A. Direct-bandgap emitter, no phosphor conversion.
- **Temperature** — not the cause. AlGaInP *redshifts* when hot (≈ +0.1 nm/°C); the
  observed orange is intrinsic chip wavelength, not thermal/drive artifact.
- **Camera/white balance** — ruled out from the photo: ambient renders neutral, so the
  orange is not a global WB tint. Brightest glyphs skew yellow from red-channel sensor
  clipping, so the photo likely *overstates* orange; true peak may be marginally redder.

A photograph bounds hue qualitatively only; it cannot establish peak nm.

## 4. Panel electronics

| IC | Role | Src |
|---|---|---|
| SN74HC595 | serial-in/parallel-out shift registers (data path: HUB12 R+CLK in, STB latches to LED columns) | `[stated]` |
| SN74HC245 | octal bus buffer on HUB12 connector; cleans/repeats control+data, buffers to next panel | `[stated]` |

Implications `[derived]`:

- Plain 595 + 245 = a **dumb shift-register panel**. No smart constant-current driver
  (MBI5xxx etc.), so no per-frame register init — DMD32 / SmartMatrix drive it directly.
- LED current is fixed by on-board resistors; **all brightness/grayscale is time-domain**
  (OE PWM / BCM at the controller). This is why grayscale depth trades against refresh.
- **HC, not HCT.** At Vcc 5 V, HC needs V_IH ≥ 0.7·Vcc = **3.5 V**. An ESP32 3.3 V GPIO
  (≈ 2.6–3.0 V under load) is below threshold → intermittent failure. **Level shifting is
  mandatory, not advisory** (see §6).

## 5. Original controller — BX-5A4 (Onbon) → discard

| Property | Value | Src |
|---|---|---|
| Type | asynchronous store-and-play sign controller | `[stated/derived]` |
| "WiFi" | transparent serial-over-TCP bridge (not frame streaming) | `[derived]` |
| Authoring | Onbon Windows software (LedShowTW / LedshowYQ) | `[stated]` |
| Outputs | HUB12 / HUB08, single/dual color | `[stated]` |
| Real-time / ArtNet | **none** — store-and-forward only | `[derived]` |

Protocol is partially reverse-engineered (text/bitmap push over TCP) but remains a
command set, not a frame pipe; cannot reach DMX/ArtNet refresh. **Bypass entirely for
lighting integration.**

## 6. ArtNet integration — ESP32 (chosen path)

Disconnect BX-5A4, drive HUB12 directly from an ESP32 ArtNet node.

- **Drivers:** DMD32 or SmartMatrix HUB12 mode (ESP32, gives software grayscale via DMA);
  DMDESP (ESP8266, weaker / 1-bit); mrcodetastic HUB75-DMA (fork for HUB12 mapping).
- **ArtNet in:** ArtnetWifi (rstephan) writing into the HUB12 framebuffer.
- **Mapping:** 1 DMX channel = 1 pixel intensity. Configure as a **single-color
  intensity matrix** in Titan/Resolume — the hardware cannot show color regardless of input.

### Level shifting (mandatory)

Insert a **74HCT245** (or 74AHCT245) per chain between ESP32 and the panel HUB12 input.
HCT has TTL thresholds (V_IH ≥ 2.0 V) → reliably accepts 3.3 V and outputs clean 5 V into
the panel's HC logic. Power from the panel 5 V rail; **common ground** with the ESP32.
Do not substitute another HC part (same threshold problem). Some panels also need clock
inversion (`ESP32_INVERT_CLK`). `[verify]` on these specific modules.

### Topology options

| Option | Description | Trade |
|---|---|---|
| **8× ESP32** (default) | one node per chain, 3 universes each | off-the-shelf firmware, failure-isolated, ~€6/node |
| **1× ESP32** | all 8 chains parallel: shared CLK/LAT/OE/A/B + 8 data lines (13 ≤ 16 I/O) | **same refresh as one chain**; needs custom parallel-output firmware; single point of failure; buffered, length-matched fan-out required |
| 2–4× ESP32 | each drives 2–4 chains in parallel | middle ground |

Parallel data adds **no** clock cycles — each edge shifts one bit into all chains at once.
That is what makes single-ESP32 driving of the whole wall feasible.

`[verify]` The single-ESP32 config is not what stock libraries do; DMD32 / SmartMatrix /
HUB75-DMA assume fixed pin maps. Requires forking the I2S/LCD parallel output.

## 7. Performance / frame rate

Governing relation: **refresh = f_clk / (bits per frame)**, with bits/frame per chain =
96 × 16 = **1 536** (independent of how many chains run in parallel).

### 1-bit (native mode), shift-bound ceiling

| f_clk | frame shift | refresh ceiling |
|---|---|---|
| 10 MHz | 154 µs | ~6.5 kHz |
| 15 MHz | 102 µs | ~9.8 kHz |
| 20 MHz | 77 µs | ~13 kHz |

Derate ~30 % for latch/OE/DMA overhead → realistically **low-thousands Hz** in 1-bit.

### Grayscale (BCM) — refresh ≈ 1-bit-ceiling / (2^N − 1), at 10 MHz

| depth | levels | ~refresh |
|---|---|---|
| 1-bit | on/off | ~6.5 kHz |
| 4-bit | 16 | ~430 Hz |
| 6-bit | 64 | ~100 Hz |
| 8-bit | 256 | ~25 Hz (flickers) |

Double at 20 MHz. The asymmetry is fundamental (MSB plane held 2^(N-1)× longer), so 8-bit
on a 1-bit-native red panel is pointless. **Operating point: 1-bit or 4-bit.**

### Practical caveats `[derived]`

- These are old DIP, HC-logic modules → signal integrity limits clean clocking to
  **~10–15 MHz**; above that expect ghosting/smearing. Plan around the 10 MHz column.
- Use **ESP32-S3** if pushing clock/depth: LCD_CAM peripheral clocks cleaner than the
  original's I2S-parallel hack; PSRAM holds the larger DMA buffer (scales ~linearly with
  bit depth; 8-bit on the full wall needs PSRAM).
- 1-bit brightness = global OE PWM, costs no refresh.

### Content rate vs panel refresh (distinct numbers)

- **Content/motion rate (ArtNet):** ~44 fps ceiling per universe (legacy DMX512 timing).
  This is perceived motion fps. sACN / ArtNet-sync can exceed it if the source supports it.
- **Panel refresh (flicker):** set by the driver (kHz in 1-bit), unrelated to ArtNet fps.

**Net:** a single ESP32-S3 can drive all 24 panels at ~1–4 kHz (1-bit) or a few hundred Hz
(4-bit), well clear of flicker — given custom parallel firmware. Eight cheap ESP32s reach
the same per-chain refresh out of the box.

## 8. Replacement sourcing

| Item | Note |
|---|---|
| Availability | V701C is current catalogue stock; scarcity is EU-local only. Single units from China / AliExpress (~$19–20). |
| Match strategy | Order the **V701C** code (fixes PCB/pinout/scan/LED family), not a generic P10 1R. |
| Package | Confirm DIP346 vs DIP546 first. |
| Channels (Berlin) | AliExpress (ships DE, VAT at checkout) · eBay DE · Alibaba MOQ-1 (B2B freight/duty dominates single-unit cost). |

**Real constraint is bin matching, not stock** `[derived]`:

- A new module differs in red peak wavelength and brightness from a different batch/year.
  Suppliers warn to buy one batch per screen.
- Existing modules have aged — red AlGaInP loses output over hours, so even an
  identical-bin new module reads brighter than aged neighbours.
- Mismatch is most visible at low drive / grayscale, least at full-on text.
- Mitigations: place new module at an edge/corner; or swap a whole row so the
  discontinuity falls on a panel boundary; per-module brightness trim helps but cannot
  fix wavelength difference. Hue match requires matching peak nm — rarely listed.

## 9. Rejected alternative — Colorlight 5A-75E

HUB75 RGB receiver card; **wrong interface** (your panels are HUB12 mono) and **wrong
architecture** (sender→receiver video-wall pipeline over a proprietary L2 Ethernet
protocol, not ArtNet). Its advantages (RGB, 8-bit grayscale, calibration, multi-kHz
refresh) are wasted on 1-bit red panels. Only sensible if the project changes to HUB75
RGB panels driven as a Resolume video surface — at which point ArtNet drops out of the
path. For HUB12 mono + ArtNet, ESP32 is the correct fit. `[derived]`

## 10. Open items

- `[verify]` LED package: DIP346 vs DIP546.
- `[verify]` LED peak wavelength (nm) — datasheet or spectrometer.
- `[verify]` Clock polarity (`ESP32_INVERT_CLK`) on these modules.
- `[verify]` Practical max clock on the actual panels (ghosting threshold).
- Decision: 8× ESP32 (low effort) vs 1× ESP32-S3 parallel (custom firmware).
- Decision: target bit depth (1-bit vs 4-bit).
