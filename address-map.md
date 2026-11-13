<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Self-Refresh and MC STATUS](self-refresh.md) | [Index](README.md#index) | [Next: The Hardware DFC Engine →](dfc-engine.md)

# Address Map and Geometry

Two different things translate addresses:

1. The **MC** maps its linear DRAM space onto ranks, rows, columns and banks.
   This is programmed by software (the DDR image).
2. The **SoC interconnect** maps CPU physical addresses onto the MC: a 2 GiB
   window at `0x0` and the rest above 4 GiB. This is fixed in the SoC.

**`0x80000000`–`0xFFFFFFFF` is not DRAM.** That range is the SoC's MMIO hole:
any CPU access to it locks the SoC (measured three times on the reference board,
with valid delay taps). Valid DRAM in CPU space is `0x0`–`0x7FFFFFFF` and
`0x100000000` onwards.

## MC registers

| Offset | Name | Content |
|---|---|---|
| `ddr+0x200` | CS0 map | bit 0 = present; `[20:16]` = size code |
| `ddr+0x204` | CS0 start high | 0 (CS0 always starts at 0) |
| `ddr+0x208` | CS1 map | bit 0 = present; `[20:16]` = size code; `[31:23]` = CS1 start, low 9 bits, in 8 MB units |
| `ddr+0x20c` | CS1 start high | CS1 start in 8 MB units, `>> 9` |
| `ddr+0x220` | CS0 geometry | rows / columns / banks |
| `ddr+0x224` | CS1 geometry | same encoding; always equal to CS0 (same die on both ranks) |

### Size code (`[20:16]`)

The code is **not** a simple shift. Confirmed on the silicon:

| Code | Size of the rank |
|---|---|
| `0x0d` | 512 MB |
| `0x01` | 768 MB |
| `0x0e` | 1024 MB |
| `0x02` | 1536 MB |
| `0x0f` | 2048 MB |
| `0x03` | 3072 MB |
| `0x10` | 4096 MB |
| `0x11` | 8192 MB |
| `0x12` | 16384 MB |

*Pattern (inferred):* codes `0x0d`…`0x12` are powers of two
(`512 MB << (code − 0x0d)`); codes `0x01`…`0x03` are the 1.5× sizes
(768, 1536, 3072 MB) used by 3 Gb / 6 Gb / 12 Gb densities.

When inserting the code, preserve bits `[14:13]`: mask `0x0060FFFF`.

### CS1 start

Unit: **8 MB**.

```
area_mb   = size_mb / cs_num        /* size of ONE rank */
start_8mb = area_mb >> 3
ddr+0x208[31:23] = start_8mb & 0x1ff
ddr+0x20c        = start_8mb >> 9
ddr+0x208[20:16] = size code of area_mb
ddr+0x208[0]     = 1
```

Example, 2 ranks × 4096 MB: `start_8mb = 512 = 0x200` → `[31:23] = 0`,
`ddr+0x20c = 1`, code `0x10` → `ddr+0x208 = 0x00100001`. The 4 GiB start lives
entirely in `ddr+0x20c`.

CS1 is always written, even with one rank: start = `area_mb`, same code.
Device_Init's default already sets bit 0 of CS1 on one-rank systems until the
map adjustment; the adjustment must leave CS1 coherent (start = `area_mb`,
same code) or clear its present bit.

### Geometry (`ddr+0x220` / `+0x224`)

```
[1:0]   bank_num   0 = 2, 1 = 4, 2 = 8 banks
[7:4]   col_num    code = columns − 7     (1 = COL_8 … 3 = COL_10 … 5 = COL_12)
[11:8]  row_num    code = rows − 10       (1 = ROW_11 … 3 = ROW_13 … 7 = ROW_17, 8 = ROW_18)
other   preserve   (0x05030000 and bits [3:2] on the reference board)
```

Insert with mask `0xFFFFF00C` (keep `[31:12]` and `[3:2]`). Reference board:
ROW_17 (7), COL_10 (3), BANK_8 (2) → `0x05030732`.

## Deriving capacity and geometry from MR8

Read MR8 outside self-refresh, after the first training pass:

- `io = MR8[7:6]`: 0 = x16, 1 = x8.
- `dens = MR8[5:2]`: JEDEC code 0 = 2 Gb, 1 = 3 Gb, 2 = 4 Gb, 3 = 6 Gb,
  4 = 8 Gb, 5 = 12 Gb, 6 = 16 Gb; 12 = 1 Gb (rare).

Size **per JEDEC channel**:

| Density code | x16 | x8 (two dies per 16-bit channel) |
|---|---|---|
| 1 Gb | 128 MB | 256 MB |
| 2 Gb | 256 MB | 512 MB |
| 3 Gb | 384 MB | 768 MB |
| 4 Gb | 512 MB | 1024 MB |
| 6 Gb | 768 MB | 1536 MB |
| 8 Gb | 1024 MB | 2048 MB |
| 12 Gb | 1536 MB | 3072 MB |
| 16 Gb | 2048 MB | 4096 MB |

Size of a rank = 2 channels × the value above. Total = rank size × `cs_num`.

Geometry (bank count is always 8 and columns always 10 for every supported density):

| I/O, density | ROW | COL | BANK |
|---|---|---|---|
| x16, 1 Gb | 13 | 10 | 8 |
| x16, 2 Gb | 14 | 10 | 8 |
| x16, 3/4 Gb | 15 | 10 | 8 |
| x16, 6/8 Gb | 16 | 10 | 8 |
| x16, 12/16 Gb | 17 | 10 | 8 |
| x8, 1 Gb | 14 | 10 | 8 |
| x8, 2 Gb | 15 | 10 | 8 |
| x8, 3/4 Gb | 16 | 10 | 8 |
| x8, 6/8 Gb | 17 | 10 | 8 |
| x8, 12/16 Gb | 18 | 10 | 8 |

Reference board (8 GiB): x8, 8 Gb → 2048 MB per channel, 4096 MB per rank (code
`0x10`), 8192 MB total, ROW_17, `+0x220 = 0x05030732`.

16 GiB SKU (*measured*): x8, 16 Gb → 4096 MB per channel, 8192 MB per
rank (code `0x11`), **16384 MB** total, ROW_18, `+0x220 = 0x05030832`
(MR8 = `0x58`). Two ranks, still no interleave; the CS1 start field
(`+0x20c = 2`, `+0x208[31:23] = 0`) points to **8 GiB of DRAM offset**. Because
of the 2 GiB low window and the 2 GiB MMIO hole, that corresponds to **CPU
address 10 GiB** (`0x2_8000_0000`). The rank-boundary table below uses CPU
addresses; do not confuse the two spaces. On the 8 GiB board the CS1 start is
4 GiB of DRAM offset, which appears at CPU address 6 GiB (`0x1_8000_0000`).

## Default map and the adjustment

Device_Init writes the default: both ranks `0x00100001` (present, 4096 MB),
`ddr+0x20c = 1`, geometry `0x05030732`. The first training pass runs with it;
training tolerates both the default and the adjusted map.

The adjustment runs after MR8 is read (and after the x8 re-initialisation if
any), before filling the DFC table:

1. `area_mb` and its size code in CS0 and CS1;
2. the CS1 start (`[31:23]` and `+0x20c`);
3. rows / columns / banks in `+0x220` and `+0x224`.

It does not touch `+0x204` (stays 0) and does not configure any interleave.
The DFC table captures these registers in its snapshot
([dfc-engine.md](dfc-engine.md)), so the table must be filled **after**
the adjustment.

## Interleave / hash

No bank/rank hash is programmed in this sequence. CS1 is **contiguous**
after CS0 in the MC address space.

## CPU address map (fixed in the SoC)

Observed with 8 GiB:

```
0x0_0000_0000 – 0x0_7fff_ffff   2 GiB   low DRAM window
0x0_8000_0000 – 0x0_ffff_ffff   2 GiB   MMIO hole (MC, PMU, SRAM, ...)
0x1_0000_0000 – 0x2_7fff_ffff   6 GiB   rest of the DRAM
```

None of this is programmed in the MC: the MC sees linear DRAM from 0 (CS0 then
CS1). The low window is why the SPL can test `0xaa55aa55` at address 0. Markers
written at `0x2000_0000`, `0x1_0000_0000`, `0x1_8000_0000`, `0x2_0000_0000` and
`0x2_7f80_0000` all read back their own value (no aliasing). The board device tree declares only the low window; the
real size comes from the density.

*Inferred:* DRAM offsets `[2 GiB, 8 GiB)` appear at CPU `0x1_0000_0000`
onward (6 GiB = total − low window), and on the 16 GiB board DRAM offsets
`[2 GiB, 16 GiB)` appear at CPU `0x1_0000_0000`–`0x4_7fff_ffff` (14 GiB = total
− low window).

## Rank boundary in CPU address space (measured)

Intentionally corrupting the PHY delay taps of one rank and running a 4 MiB
memory pattern across base addresses locates the exact CPU address where CS1
begins (a single-word probe stays in cache and is inconclusive). The boundary
depends on **which MC address map is currently programmed**:

| Board / active map | CS0 CPU range (excluding `2–4 GiB` MMIO hole) | CS1 starts at (CPU address) |
|---|---|---|
| 16 GiB, unadjusted default map during first fp0 pass (`0x00100001` / `0x05030732`) | below 6 GiB | **6 GiB** (`0x1_8000_0000`, DRAM offset 4 GiB) |
| 16 GiB, final adjusted map (`0x00110001` / `0x05030832`) | **0–9.75 GiB** | **10 GiB** (`0x2_8000_0000`, DRAM offset 8 GiB) |
| 8 GiB, final map (`0x00100001` / `0x05030732`) | **0–5.75 GiB** | **6 GiB** (`0x1_8000_0000`, DRAM offset 4 GiB) |

Measured on the 16 GiB final map at fp2 in 256 MiB steps:

| CPU Address | CS1 with corrupted delays | CS0 with corrupted delays | Serving rank |
|---|---|---|---|
| 6 – 9.75 GiB | passes | **fails** | CS0 |
| 10 GiB and above | **fails** | passes | CS1 |

The ranks are **not** fine-interleaved: each address range belongs to a single
rank, and corrupting a rank's delays breaks that entire contiguous region. Any
per-rank software training sweep must use a **CPU address ≥ 10 GiB** for CS1 on
the 16 GiB final map and **≥ 6 GiB** on the 8 GiB final map (and below those
thresholds for CS0), which are the CPU aliases of DRAM offsets 8 GiB and 4 GiB.
Probing above 8 GiB is only safe **after** the MC address map has been adjusted
for 16 GiB: during the first fp0 pass the default map covers only 8 GiB of DRAM,
so writing 4 MiB above 8 GiB hangs the controller.

## Gaps

- Fields of `+0x208`/`+0x200` outside bit 0, `[14:13]`, `[20:16]` and
  `[31:23]`, and the upper `0x0503` of the geometry word.
- Whether the CPU-side window layout is fixed or configurable somewhere in the
  interconnect.
- 1 Gb density and x16 configurations were not measured on hardware. The 16 GiB
  configuration (code `0x11` per rank, ROW_18, 2 ranks) is measured on the
  Micron `MT53E4G32D8` board.

---

[← Previous: Self-Refresh and MC STATUS](self-refresh.md) | [Index](README.md#index) | [Next: The Hardware DFC Engine →](dfc-engine.md)
