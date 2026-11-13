<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Configurations](configurations.md) | [Index](README.md#index) | [Next: Test Platform →](test-platform.md)

# Acceptance Criteria and Reference Measurements

Boards:
- **8 GiB reference board:** Banana Pi BPI-F3, 8 GiB, LPDDR4X Samsung
  `K4UCE3Q4AA-MGCL`, 2 ranks, x8 dies (8 Gb per die).
- **16 GiB board:** Banana Pi BPI-F3, 16 GiB, LPDDR4X Micron `MT53E4G32D8`,
  2 ranks, x8 dies (16 Gb per die).

Every value below is what the **hardware** of these boards produces when it is
programmed as described in [boot-sequence.md](boot-sequence.md) and the files it
references. The values serve as acceptance ranges; they are not the algorithm.
Do not hard-code them.

## Acceptance criteria

The hardware has reached the expected state when:

- `PHY_STATUS == 0` after every training pass (four passes for LPDDR4X).
- Gate: right after RGATE, `(PI_STATUS & 6) == 6` (and `PHY_STATUS` bit 4 == 0),
  and the code is `0x0001ggtg` with equal bytes. At the end of the pass both
  bytes are in range (equality not required).
- After the second fp0 pass the gate is ≈29–40 taps (not the ≈101–118 of the
  cold first pass). 29–32 is within range.
- At 2400 MT/s the first fp2 pass gives Rx index 4–6 with minimum margin ≥10
  and Tx MR14 `0x1d`–`0x1f` (29–31) with minimum margin ≥10.
  Margins of 8/9 there mean the clock is actually 2666 (check the PLL1 recipe,
  [clocks-and-pmu.md](clocks-and-pmu.md)); at 2666 MT/s 8/9 is the expected result.
- At 1333 MT/s (fp0 with the 2666 VCO) the fp0 slice gives RX window 20/32,
  TX 17/48 and PI minima 21/17; the fp2 slice remains at its 2400/2666 values.
- MR14 read after the SR exit equals `CTL_TXVREF[21:16]` on all four
  (rank, channel) pairs.
- TX DQ0 ≈30–36 taps and RX DQ0 ≈17–19 taps at the final fp.
- WL in `+0x24` distinct per byte and per rank, ≈17–24, inside the 5-bit field;
  the passing window per byte is in the table below.
- The capacity derived from MR8 is 8192 MB (8 GiB board) or 16384 MB (16 GiB
  board).
- The whole DRAM holds data without errors or aliasing, and code can be loaded
  into it and executed.

## Final rates (measured)

| Final rate | fp | RX window (of 32) | TX window (of 48) | PI Rx min | PI Tx min |
|---|---|---|---|---|---|
| 1066 | 1 | 25 | 22 | 26 | 24 |
| 1200 | 0 | 22 | 19 | 24 | 20–21 |
| 1333 | 0 (2666 VCO) | 20 | 17 | 21 | 17 |
| 2400 | 2 | 8 | 9 | 10 | 10 |
| 2666 | 2 | 7 | 6 | 8–9 | 8–9 |

Windows are the delay-line sweep at CS0 on the 8 GiB board; the PI minima are
from the training pass. 1333 uses the same fp0 slice as 1200; only the PLL1 VCO
changes. 1066 is the widest, 1200 next. Linux throughput/latency is in
[Memory throughput](#memory-throughput-measured-from-linux).

## 16 GiB Micron board (measured, 1066 MT/s final)

Second BPI-F3 with 16 GiB (Micron `MT53E4G32D8`, eight 16 Gb x8 dies, 2 CS;
[configurations.md](configurations.md)). When the requested final rate is 1066,
the boot uses the 2400 PLL1 VCO recipe (`+0x39c = 0x2a203b00`, `+0x3a0 =
0x50dd6432`, `clksel 0x3b50`): the first two passes train fp0 at **1200 MT/s**,
the third trains fp1 at **1066 MT/s** (`0x3b04`), and the fourth trains fp2 at
**2400 MT/s** (`0x3b40`). `PHY_STATUS = 0` in all four passes; capacity is
16384 MB. `MR14` after each pass: `0x18 / 0x15 / 0x18 / 0x1b`; `MR14 ==
CTL_TXVREF[21:16]` at the end.

| Pass | fp | clksel | RX idx / Vref / min | TX idx / Vref / min |
|---|---|---|---|---|
| fp0, first | 0 | `0x00033b50` | 5 / 26 / 25 | 3 / 24 / 23 |
| fp0, second | 0 | `0x00033b50` | 5 / 26 / 25 | 0 / 21 / 23 |
| fp1 | 1 | `0x00033b04` | 3 / 24 / 27 | 3 / 24 / 27 |
| fp2 | 2 | `0x00033b40` | 5 / 26 / 11 | 6 / 27 / 10 |

`idx` is the position in the 21..36 Vref seed (`Vref = 21 + idx`); the minimum
is the worst DQ across both ranks. Per-step minimum margin curves (step 0 → 15,
corresponding to Vref 21 → 36 for TX):

```
fp0a RX  22 23 23 23 24 25 24 24 24 23 22 21 21 17 15 14
fp0b RX  22 22 23 23 24 25 24 24 23 23 22 21 20 16 14 11
fp1  RX  25 26 26 27 27 27 26 26 26 25 24 23 22 18 16 15
fp2  RX   8  9  9 10 10 11 11 10 10  9  8  8  5  0  0  0
fp0a TX  22 22 22 23 23 23 23 23 23 23 23 23 23 23 23 23
fp0b TX  23 22 23 23 23 23 23 23 23 23 23 23 23 23 23 23
fp1  TX  26 26 26 27 27 27 27 27 27 27 27 27 27 27 27 26
fp2  TX   9  9  9  9  9  9 10 10 10 10 10 10 10  9  9  9
```

Mode registers before training, identical on all four (CS, CH) pairs:
MR5 = `0xFF` (Micron), MR6 = `0x07`, MR8 = `0x58` (x8, 16 Gb), MR12 = `0x5D`,
MR14 = `0x19`. Final MC map: `0x00110001` / `0x05030832` / `+0x20c = 2`. An
unprimed MR5 read immediately after the first DFC to fp2 returns `0x00` on the
first pair and `0xFF` on the remaining three; treat the first read after a DFC
as a dummy read ([mode-registers.md](mode-registers.md)).

Trained delays per byte (`b0 b1 b2 b3`), CS0 / CS1:

| Pass | WL | Gate | RX DQ0 | TX DQ0 |
|---|---|---|---|---|
| fp0a | `20 23 19 22` / `19 22 19 22` | `112 110 114 110` / `112 110 114 110` | `18 18 18 17` / `18 18 18 17` | `34 36 33 35` / `33 35 33 35` |
| fp0b | `20 22 19 22` / `19 23 19 22` | `32 30 32 30` / `33 31 33 29` | `18 18 19 17` / `18 18 18 17` | `34 35 33 35` / `33 36 33 35` |
| fp1 | `20 23 19 22` / `20 23 19 22` | `32 30 34 30` / `34 30 34 30` | `17 17 18 16` / `17 16 18 16` | `34 36 33 35` / `34 36 33 35` |
| fp2 | `21 24 20 23` / `21 24 19 23` | `34 32 34 32` / `36 32 34 32` | `17 17 18 17` / `18 18 18 17` | `35 37 34 36` / `35 37 33 37` |

Acceptance ranges on this 16 GiB SKU (2400 VCO): RX minimum 25–27 of 32 in
fp0/fp1 and 11 in fp2; TX minimum 23–27 of 48 in fp0/fp1 and 10 in fp2 (with the
2666 VCO: RX 22/27/9–10, TX 21/27/8); WL 19–24; RX DQ0 16–19; TX DQ0 33–37; gate
after the x8 PHY re-init 29–34; MR14 after fp2 `0x1b`–`0x1d` (2400 VCO) or
`0x16`–`0x17` (2666 VCO). The minimum margin is repeatable across 15 consecutive
runs, whereas the chosen index inside the flat TX plateau and the resulting MR14
vary by ±2–3 — validate against the margin and plateau range, not a single MR14
code.

Independent software sweeps reproduce the same sampling points on this board:
RX centres 17/17/17 against the PI's 18/17/17, TX centres 35/35/36 against
34/34/35, with windows 24/27/10 (RX) and 22/24/8 (TX) on the 2400 VCO, and
21/27/9 (RX) and 20/24/6 (TX) on the 2666 VCO. At fp2, **per-lane RX centring**
is required so that multi-pattern verification passes with 0 errors
([eye-training.md](eye-training.md#on-the-16-gib-micron-board-measured-1066-final)).

### Per-rank (CS1) Rx and Tx windows

With the rank boundary in CPU address space established (**10 GiB** on the
16 GiB board, **6 GiB** on the 8 GiB board on the final map, aliasing DRAM
offsets 8 GiB and 4 GiB; see
[address-map.md](address-map.md#rank-boundary-in-cpu-address-space-measured)),
the independent Rx/Tx delay sweeps at fp2 across both ranks give:

| Board / VCO | Rx CS0 | Rx CS1 | Tx CS0 | Tx CS1 |
|---|---|---|---|---|
| 8 GiB / 2400 | 14..20 (7) | 14..21 (8) | 31..39 (9) | 31..38 (8) |
| 8 GiB / 2666 | 15..20 (6) | 15..21 (7) | 31..36 (6) | 31..36 (6) |
| 16 GiB / 2400 | 13..21 (9) | 13..21 (9) | 34..40 (7) | 34..40 (7) |
| 16 GiB / 2666 | 15..20 (6) | 14..20 (7) | 34..38 (5) | 34..38 (5) |

CS1 is symmetric to CS0: Rx windows match within ±1 code and Tx windows are
identical (with a 1-code difference on the 8 GiB / 2400 row). Per-rank software
training can use the same algorithm on both ranks.

## Per-pass results at 2400 MT/s

PLL1 programmed with the 2400 read-modify-write recipe; 9 runs of the full
sequence. Gate in taps; index / MR14 with the minimum margin in parentheses;
MR14 in decimal.

| Pass | fp | Gate | Rx index (min) | Tx MR14 (min) | PHY_STATUS |
|---|---|---|---|---|---|
| fp0, first | 0 | 111–118 | 5–6 (24) | 26–36 (20–21) | 0 |
| fp0, second | 0 | 32–38 | 5–6 (24) | 26–36 (20–21) | 0 |
| fp1 | 1 | 32–38 | 3–4 (26) | 31–35 (24) | 0 |
| fp2 | 2 | 34–40 | 5–6 (10) | 29–31 (10) | 0 |

In all 9 runs:

- end of fp2: WL 19–23 distinct per byte and rank, Rx DQ0 18–19, Tx DQ0 32–36;
- end state STATUS `0x33`, `CTL_FREQ = 0xa0800400`, capacity 8192 MB;
- MR14 `0x1e`–`0x1f` on all four pairs, always equal to `CTL_TXVREF[21:16]`;
- DFC to fp1 and fp2 in 29–30 µs, final DFC to fp2 in 25–26 µs.

A wider envelope at 2400 MT/s (other measurement sets on the same board):

| Pass | Gate | Rx index / margin | Tx MR14 | PHY_STATUS |
|---|---|---|---|---|
| fp0, first | ≈105–118 | 4–7 / 24 | ≈`0x19`–`0x1c` | 0 |
| fp0, second | ≈32–40 | 4–5 / 24 | ≈`0x1a`–`0x1c` | 0 |
| fp1 | ≈33–38 | 3–5 / 26 | ≈`0x1f` | 0 |
| fp2 | ≈35–40 | 4–6 / 10 | ≈`0x1d`–`0x1f` | 0 |

### Repeated training passes at fp2, 2400 MT/s (6 runs, 2 ranks)

A training pass ([training-pass.md](training-pass.md)) run again on fp2 of a controller
that is already initialised and trained:

- Rx winner always index 6; per-step minimums
  `7 7 8 8–9 9 9 10 10 9 9 8 7 6–7 0 0 0`.
- Tx plateau at margin 10 from index 8–10 on (MR14 29–31).
- WL CS0 `19 22 20 23`, CS1 `19 21 20 23`.
- Rx DQ0 18–19, Tx DQ0 32–36.
- Each PI operation ≤17 µs.
- DRAM contents survive the pass.

Register state after such a pass:

| Register | Value |
|---|---|
| `CTL_FREQ` | `0xa0800400` |
| `PI_CFG` | `0x8` |
| CS0/CS1 map | `0x00100001` |
| `PI_STATUS` | `0x80008067` |
| VREF0 / VREF1 | `0x6666a937` / `0x6666a927` |
| `CTL_TXVREF` | `0x9d1f4314` (MR14 field `0x1f`, CA Vref `0x9d`) |
| `PHY_STATUS` | 0 |
| MR5/6/8/12/14 | `0x01 / 0x08 / 0x52 / 0x5d / 0x1f` |

### Tap snapshots at 2400 MT/s

- TX DQ0 per byte after write training, CS0: `33 35 33 36`.
- RX DQ0 per byte after read training, CS0: `18 18 18 17`.
- WL: CS0 `20 22 20 23`, CS1 `19 22 20 23`.

## Results at 2666 MT/s

PLL1 `+0x39c = 0x203c5555`, `+0x3a0 = 0x50dd6538`; `PHY_STATUS` 0 in all four
passes; DRAM usable.

| Pass | Rate | Rx best (margin) | Tx best (margin) | MR14 | UI |
|---|---|---|---|---|---|
| fp0, first | ≈1333 | idx 4 (21) | 29 (17) | `0x1d` | 750 ps |
| fp0, second | ≈1333 | idx 4 (21) | 28 (17) | `0x1c` | 750 ps |
| fp1 | 1066 | idx 5 (27) | 33 (24) | `0x21` | 938 ps |
| fp2 | 2666 | idx 6 (9) | 32 (9) | `0x20` | 375 ps |

Over five runs of the full sequence at 2666: fp0 Rx 4–7 / 21–22,
Tx 26–28 / 17; fp1 Rx 3 / 26, Tx 31–34 / 24; fp2 Rx 5–7 / 8–9,
Tx MR14 23–35 / 8–9. In all of them `PHY_STATUS` was 0, the capacity 8192 MB
and the DRAM pattern tests error-free.

Compared with 2400: RX 9 vs 10, TX 9 vs 10 at fp2. Rx indices 13–15 still have
margin 0. TX/RX taps at fp2 sit in the middle of the UI (TX 32–36, RX 17–18) —
the same tap counts as at 2400.

WL (b0 b1 b2 b3) at 2666: CS0 `19 21 19 23`, CS1 `19 21 20 22`; at 1200:
CS0 `18 21 17 22`, CS1 `17 20 17 21`.

In one of two 2666 runs, RX DQ0 of CS1 byte 1 settled at 20 taps, just above
the 17–19 range; this was not observed at 2400.

The fp0 line (≈1333) is the same slice used for a 1333 final rate; see
[1333 MT/s as the final point](#1333-mts-as-the-final-point).

## fp0 as the final point

This is what a 2133 MT/s request maps to on LPDDR4X ([boot-interface.md](boot-interface.md)).
After fp0/fp1/fp2 are trained, the final DFC from fp2 back to fp0 completes in
29–30 µs and leaves `CTL_FREQ = 0x00800400`, clksel `0x3b50`. Latency confirms
1200 MT/s (seq / rand 32 MiB 333.6–334.1 / 374.0–374.4 ns, next to the
332.2 / 373.0 ns measured when fp0 is selected directly). DRAM pattern tests are
error-free.

### 1333 MT/s as the final point

The same fp0 final point with the PLL1 2666 VCO runs at ≈1333 MT/s (`PLL1 / 2`).
The final DFC from fp2 back to fp0 leaves `CTL_FREQ 0x00800400`, clksel
`0x3b50`. Linux: 4.20 GB/s write / 3.23 GB/s read (1 process), 337 ns; four
processes 4.30/4.55 GB/s, 357 ns. The theoretical peak is 5.33 GB/s (~80 %).
There is no 1333 `data_rate` entry; see [configurations.md](configurations.md).

## Register milestones (2400 MT/s, 2 ranks)

| Point | Values |
|---|---|
| Entry to `ddr_init` | `pmu+0x04 = 0x00019249`, `+0xb0 = 0xa0610c00`, `+0x3b4 = 0x00073340`, `+0x39c = 0x2a200000`, `+0x3a0 = 0x50dd6432`, `+0xe8 = 0x40001000` |
| After PLL + SW-FC | `+0x3b4 = 0x00033b50`, `+0xb0 = 0x40600400`, `+0xe8 = 0x40001003` |
| After Device_Init | STATUS `0x33`, `CTL_FREQ = 0x00800400`, `PI_CFG = 0`, `PHY_INIT = 0x100`, `PI_STATUS = 0x80008000`, COMMON+0xc `0x36e4d800` |
| After MR8 read | MR5/MR8 `0x01 / 0x52` → x8, 8192 MB, ROW_17, map `0x00100001`, geometry `0x05030732` |
| After DFC(1) / DFC(2) | STATUS `0x33`; `CTL_FREQ` `0x50800400` / `0xa0800400`; `PI_CFG` 4 / 8; clksel `0x00033b04` / `0x00033b40` |
| End of sequence | STATUS `0x33`, `CTL_FREQ = 0xa0800400`, `PI_CFG = 8`, clksel `0x00033b40` |

No PI operation took more than 34 µs; no gate timeout and no stale first read
of `PI_STATUS` in any operation.

## DRAM latency (frequency meter)

Pointer chase, 2²⁰ dependent loads, 64 B nodes, `mtime` at 24 MHz
([test-platform.md](test-platform.md)).

Data cache **off**, CPU at 307 MHz (in this state the cache does not filter:
16 KiB costs the same as 32 MiB). ns per load:

| Rate | fp | `pmu+0x39c` | Seq 32 MiB | Rand 32 MiB |
|---|---|---|---|---|
| 2400 | 2 | `0x2a203b00` | 266.10–266.13 | 291.04–291.39 |
| 2666 | 2 | `0x203c5555` | 250.00–250.01 | 277.39–277.40 |
| 1333 | 0 | `0x203c5555` | 310.7 | 350.0 |
| 1200 | 0 | `0x2a203b00` | 332.17–332.20 | 373.01–373.05 |
| 1066 | 1 | `0x2a203b00` | 361.97 | 404.09–404.14 |

The 1333 row is fp0 with the 2666 VCO (same PLL words as 2666); the 1200 row
uses the 2400 VCO. With the full word `0x203b5555` the "2400" line is identical
to the 2666 one, and fp0 measures as ≈1333 (310.7 / 350.0 ns).

fp2 drops 6.0 % (seq) and 4.8 % (rand) from 2400 to 2666.

Data cache **on**, CPU at 1599 MHz, random 32 MiB: see the table in
[clocks-and-pmu.md](clocks-and-pmu.md) (fp0 299.65 / 278.66 ns, fp1
331.31 / 331.35 ns, fp2 213.35 / 198.94 ns for 2400 / 2666).

## Gate delay word `+0x64` (measured)

Only byte 0 / CS0 has this word (the other seven byte/rank blocks read 0). The
low 5 bits are the applied delay; the high byte (`0x43`) is a qualifier.

Sweep of the low byte at 2666 MT/s with the memory-pattern test (0 = the byte
value passes, 1 = fails):

| Byte value | Result |
|---|---|
| 0..9 | fail |
| 10..19 | pass |
| 20..41 | fail |
| 42..52 | pass |
| 53..73 | fail |
| 74..85 | pass |
| 86..105 | fail |
| 106..118 | pass |
| 119..137 | fail |
| 138..147 | pass |
| 148..255 | fail |

The first five windows repeat every 32 codes, which is why the field is read as
5 bits: 10..19, 42..52, 74..85, 106..118, 138..147. Above 148 no value passed
(a single point at 201 excepted); the 2666 link is marginal and a 256-step sweep
may drift, so treat that upper part as inconclusive rather than as a different
mapping.

The window is ≈10–13 codes wide at 2666 MT/s (and exactly 10 codes at 2400,
10..19).
The trained value (e.g. `0x4c`, low 5 bits = 12) sits near the low edge of its
window, not at the centre — consistent with a gate that must open before the
preamble. The window does not move when the Rx data delay is shifted by ±2
codes, so the gate and the data delay are independent axes.

## Memory throughput (measured from Linux)

One 32-bit channel: the theoretical peak is 4.27 GB/s at 1066, 4.8 at 1200,
5.33 at 1333, 9.6 at 2400 and 10.66 GB/s at 2666. Measured on the reference
board with a user-space benchmark (256 MiB buffers, one pass per phase; four
processes of 128 MiB for the aggregate):

| Rate | 1 core write | 1 core read | 1 core copy (R+W) | 4 cores write | 4 cores read | 1-core latency |
|---|---|---|---|---|---|---|
| 1066 | 3.39 GB/s | 3.12 GB/s | 2.86 GB/s | 3.40 GB/s | 3.56 GB/s | 391 ns |
| 1200 | 3.81 GB/s | 3.20 GB/s | 3.31 GB/s | 3.92 GB/s | 4.01 GB/s | 359 ns |
| 1333 | 4.20 GB/s | 3.23 GB/s | 3.72 GB/s | 4.30 GB/s | 4.55 GB/s | 337 ns |
| 2400 | 4.21 GB/s | 3.26 GB/s | 5.26 GB/s | 7.60 GB/s | 7.03 GB/s | 265 ns |
| 2666 | 4.22 GB/s | 3.27 GB/s | 5.56 GB/s | 7.89 GB/s | 7.24 GB/s | 252 ns |

- At 1066/1200/1333 a single process already saturates the channel (~80 % of
  peak); the four-process aggregate adds little.
- At 2400/2666 a single core does not saturate (too few outstanding misses): the
  four-process aggregate reaches ≈79 % of the theoretical peak at 2400 and ≈74 %
  at 2666.
- The 2666 gain is +3.8 % (write) and +3.0 % (read) in the aggregate, and zero on
  one core.
- 1333 gives ~57 % of the 2400 bandwidth and 1.27× its latency; 1066 is the
  widest-eye point (25/32 RX), 1200 the next (22/32).
- User-space benchmark latency (1 process): 391 ns at 1066, 359 at 1200,
  337 at 1333, 265 at 2400 and 252 ns at 2666.
- A user-space pointer chase (32 MiB, huge pages) gives 265 ns at 2400 and
  252 ns at 2666 — the same ≈5 % improvement as the firmware probe above.

Implication for a DDR image: going from 2400 to 2666 buys a few percent of
throughput and ~5 % of latency, at the cost of the eye margin documented in
[eye-training.md](eye-training.md#at-2666-mts).

### 16 GiB Micron board (measured)

Running the same Linux benchmark on the 16 GiB Micron board across all five
final rates (`PHY_STATUS = 0` in all four passes, 16384 MB capacity; 1333 uses
the 2666 VCO with `PLL1_CTRL1 = 0x203c5555`, `PLL1_CTRL2 = 0x50dd6538`, and
`clksel 0x3b50` at fp0):

| Rate | 1 core write | 1 core read | 1 core copy (R+W) | 4 cores write | 4 cores read | 1-core latency |
|---|---|---|---|---|---|---|
| 1066 | 3.39 GB/s | 3.13 GB/s | 2.84 GB/s | 3.45 GB/s | 3.23 GB/s | 389.7 ns |
| 1200 | 3.81 GB/s | 3.19 GB/s | 3.28 GB/s | 4.04 GB/s | 3.86 GB/s | 358.4 ns |
| 1333 | 4.20 GB/s | 3.20 GB/s | 3.68 GB/s | 4.28 GB/s | 4.64 GB/s | 335.8 ns |
| 2400 | 4.22 GB/s | 3.27 GB/s | 5.20 GB/s | 7.66 GB/s | 6.95 GB/s | 266.3 ns |
| 2666 | 4.22 GB/s | 3.25 GB/s | 5.51 GB/s | 7.87 GB/s | 7.26 GB/s | 251.8 ns |

Four-process average latency: 425.8 / 384.5 / 351.8 / 279.6 / 260.7 ns (~81 /
84 / 80 / 80 / 74 % of the 32-bit theoretical peak). Single-process numbers
match the 8 GiB Samsung board within ~1 % at every rate, and four-process
aggregates match within normal 8-core scheduling variance: 1066 3.45/3.23 vs
3.40/3.56 GB/s, 1200 4.04/3.86 vs 3.92/4.01, 1333 4.28/4.64 vs 4.30/4.55, 2400
7.66/6.95 vs 7.60/7.03, and 2666 7.87/7.26 vs 7.89/7.24. There is no difference
in throughput or latency between the 8 GiB and 16 GiB SKUs.

**15-minute memory soak at 2666 MT/s (80 Ω Tx ODT):** running 4 processes ×
512 MiB (2 GiB resident) writing and verifying 64-bit patterns continuously for
900 s completed 1551–1559 iterations per process (≈6.4 TB of traffic) on the
16 GiB Micron board with **0 pattern errors**, and 1258–2290 iterations per
process (≈6.2 TB of traffic) on the 8 GiB Samsung board with **0 pattern
errors**.

## DRAM integrity

With the full sequence completed at 2400 MT/s:

- pattern tests: 0 errors over 64 MiB at `0x10000000` and at `0x200000000`, and
  over 4 MiB at `0x100000000` and at `0x27f000000`;
- address markers written at `0x20000000`, `0x100000000`, `0x180000000`,
  `0x200000000` and `0x27f800000` each read back their own value (no aliasing
  between the low window and the high window).

## Hardware behaviour in failure conditions

| Condition | What the hardware shows |
|---|---|
| PLL enable bits left cleared | CLKSEL reads `0x00003040`; the lock bits never assert |
| RGATE done bit awaited without issuing the command | Bit 1 of `PI_STATUS` never sets; the register stays `0x80008001` |
| HW-DFC kicked without a DFC table | `pmu+0x04` stays `0x00400000` (bit 22 stuck), STATUS `0x77`, `CTL_FREQ = 0x00800400` (unchanged) |

## Per-lane delay and margin map (1200 MT/s, fp0)

Full read-back after one complete pass on fp0 (1200 MT/s), CS0. Delay values are
taps; margins are the harvest values (unit unknown, only comparative).

RX delay words, DQ0..7 | DQ8..15 | DQ16..23 | DQ24..31, `dm` = DM lane:

```
18 17 18 17 19 18 19 18 dm17 | 18 18 18 18 18 17 18 18 dm17 |
18 19 19 18 19 18 19 19 dm18 | 18 17 17 17 17 17 18 17 dm17
```

TX delay words, same layout:

```
32 32 32 31 32 33 33 32 dm32 | 34 35 35 35 35 34 35 35 dm34 |
31 31 31 31 32 32 31 31 dm31 | 35 35 35 35 35 34 35 35 dm35
```

Ranges per byte group (1200 MT/s, fp0, all four passes observed):

| Group | RX taps | TX taps | RX margin | TX margin |
|---|---|---|---|---|
| 0 (DQ0–7) | 17–19 | 31–33 | 24–26 | 21–22 |
| 1 (DQ8–15) | 17–18 | 34–35 | 24–26 | 21–22 |
| 2 (DQ16–23) | 18–19 | 31–34 | 24–26 | 21–22 |
| 3 (DQ24–31) | 17–18 | 34–36 | 24–26 | 21–22 |

The TX taps differ by group by up to 4 (31–33 vs 34–36) while the RX taps stay
in 17–19: the write eye is not centred at the same code in every group.

At 1066 MT/s the same map gives RX 16–18 and margins 26–29 / TX 24–26; at fp2
(2400 MT/s) both margins fall to 9–12 and the TX taps move to 33–36, with one
group saturating at the top code (36).

With the PLL1 VCO at 2666 MT/s the same read-back gives RX 17–18 and TX 32–36
at fp2, margins 9–12 (RX) and 8–11 (TX), `ctl148 = 0x80ac0000` and gate codes
around `0x000125xx` (e.g. `0x00012524 0x00012423 0x00012322 0x00012423`). At
fp0 (≈1333 MT/s) the margins are 21–23 (RX) and 16–19 (TX); at fp1 (1066) they
are 26–29 and 24–26.

### Expected delay windows (acceptance)

Measured with a memory-pattern sweep (same board, CS0, all lanes of a group at
the same code):

| PLL VCO | fp | Rate | RX window | TX window |
|---|---|---|---|---|
| 2400 | 0 | 1200 | 7..28 (22 of 32) | 24..42 (19 of 48) |
| 2400 | 1 | 1066 | 5..29 (25 of 32) | 22..43 (22 of 48) |
| 2400 | 2 | 2400 | 14..21 (8 of 32) | 31..39 (9 of 48) |
| 2666 | 0 | ≈1333 | 9..28 (20 of 32) | 25..41 (17 of 48) |
| 2666 | 1 | 1066 | 5..30 (26 of 32) | 22..44 (23 of 48) |
| 2666 | 2 | 2666 | 15..21 (7 of 32) | 31..36 (6 of 48) |

A new implementation should land its sampling point inside these windows; the
PI leaves RX 17–19 and TX 31–36 at 2400, which is their centre. A result outside
them, with `PHY_STATUS == 0`, means the delay words or the slice were programmed
differently. At 2666 the windows are small enough that a single pattern pass is
not conclusive (see [eye-training.md](eye-training.md#at-2666-mts)).

### Expected WL windows (acceptance)

Sweep of the 5-bit `+0x24` word with the memory-pattern test, CS0, per byte:

| PLL VCO | fp | Rate | byte 0 | byte 1 | byte 2 | byte 3 |
|---|---|---|---|---|---|---|
| 2400 | 0 | 2400 | 8..27 (20) | 11..31 (21) | 6..29 (24) | 8..31 (24) |
| 2400 | 1 | 1066 | 6..29 (24) | 9..31 (23) | 6..30 (25) | 7..31 (25) |
| 2400 | 2 | 2400 | 15..24 (10) | 16..26 (11) | 15..25 (11) | 17..28 (12) |
| 2666 | 0 | ≈1333 | 9..25 (17) | 13..30 (18) | 4..27 (24) | 9..31 (23) |
| 2666 | 1 | 1066 | 6..29 (24) | 9..31 (23) | 5..30 (26) | 6..31 (26) |
| 2666 | 2 | 2666 | 15..22 (8) | 16..24 (9) | 15..23 (9) | 16..27 (12) |

The PI's WL codes at 2400 are 18 / 21 / 17 / 22, inside these windows and within
two codes of their centres. On the final map, Rank 1 (CS1) is reached at CPU
address ≥ 6 GiB on the 8 GiB board and ≥ 10 GiB on the 16 GiB board
([address-map.md](address-map.md#rank-boundary-in-cpu-address-space-measured));
sweeping CS1 at those addresses gives WL windows within ±1 code of CS0 on the
8 GiB board and identical to CS0 on the 16 GiB board
([write-leveling.md](write-leveling.md#independent-measurement-of-the-wl-window-memory-pattern-sweep)).

## Gaps

- No reference data exists for LPDDR4, x16 or one-rank SKUs. Both the 8 GiB
  Samsung and 16 GiB Micron 2-rank x8 LPDDR4X SKUs are fully measured.
- Margins are only compared between runs; their absolute unit is unknown.
- No temperature or voltage variation was characterised.

---

[← Previous: Configurations](configurations.md) | [Index](README.md#index) | [Next: Test Platform →](test-platform.md)
