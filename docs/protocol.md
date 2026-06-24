# Custom UDP wire protocol — DRAFT

Packed, high-refresh streaming protocol for the P10 wall (requirement **R9**).
Companion to the ArtNet input (R6), which stays for stock-gear interop. This is a
**draft stub** — fields marked `[draft]` are proposals; `[verify]` needs the real
hardware/geometry to confirm.

## Goals
- Send only the bits the panel needs: **4 bit/pixel, no universe padding.**
- Every packet ≤ MTU → no IP fragmentation (the W5500 hardware UDP stack does **not**
  reassemble fragments).
- Packet structure maps 1:1 onto the 8-chain DMA layout; partial/region updates possible.
- ~5.3 Mbps wire @ 100 Hz, 800 pps (vs ArtNet ~11 Mbps / 2,400 pps).

## Transport
- **UDP**, fire-and-forget. Default port **`[draft]` 6454+? TBD** (avoid 6454 = ArtNet).
- One frame = **8 packets**, one per chain. Each packet is a full Ethernet frame
  (payload 776 B ≤ 1472 B MTU limit).

## Geometry mapping
- Wall: 96 px wide × 128 px tall = 12,288 px.
- **Chain c (0..7) = horizontal band of rows [c·16 .. c·16+15], full 96 px wide.**
  `[verify]` physical chain arrangement on the actual wall (vertical-band assumption).
- Per chain: 96 × 16 = 1,536 px → **768 bytes** packed.

## Packet layout `[draft]`

```
Offset  Size  Field        Notes
0       2     magic        0x48 0x32  ("H2")
2       1     version      protocol version (start 0x01)
3       1     type         0x01 = chain pixel data
4       2     frame_id     LE, wraps; identifies the frame a packet belongs to
6       1     chain_index  0..7
7       1     flags        bit0 = LATCH (present frame after this packet)
8       768   pixels       4-bit packed, 2 px/byte (see below)
----    ----
total   776   bytes payload (+ 28 IP/UDP + 18 Eth = 822 B on wire)
```

### Pixel packing
- Row-major within the chain: index = y·96 + x, x∈[0,95], y∈[0,15] (chain-local rows).
- Two pixels per byte: **high nibble = even index, low nibble = odd index** `[draft]`.
- Nibble value 0..15 = BCM grayscale level (R7).

## Frame assembly / anti-tearing `[draft]`
- Node keeps a back buffer per chain. On receiving a packet, write its 768 B into the
  back buffer slot for `chain_index`.
- **Latch** (swap back→front for DMA) when either: all 8 chains of the current
  `frame_id` have arrived, **or** a packet with `flags.LATCH` is received.
- Stale/duplicate `frame_id` packets are dropped. Late chain → that band shows the
  previous frame for one cycle (graceful single-chain glitch).

## Open questions
- Latch policy: all-8-received vs explicit LATCH flag vs frame_id change.
- Sequencing/loss handling: pure fire-and-forget vs per-chain seq numbers.
- Partial-region update packets (sub-chain rectangles) — needed, or out of scope?
- Endianness/port finalization; optional CRC.
- Whether to support 1-bit packets (192 B/chain) for the native fast path.

## Source side (R10)
External host renderer / **NDI → protocol converter**: receives an NDI video stream,
downscales/dithers to 96×128 × 4-bit amber intensity, emits the 8 per-chain packets at
the target rate. Lives outside the firmware; see R10.
