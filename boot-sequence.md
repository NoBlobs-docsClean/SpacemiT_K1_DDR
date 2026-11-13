<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: CA Training](ca-training.md) | [Index](README.md#index) | [Next: Configurations →](configurations.md)

# The Complete `ddr_init` Sequence

A training pass sees **one** frequency point. Deciding how many passes to run,
in which order, and what happens between them is the job of `ddr_init`.

## LPDDR4X, 2 ranks, x8 dies (reference board)

```
 1  check arguments; data_rate -> PLL1 recipe and final fp     (04)
 2  PLL1 + PLL2, lock, SW-FC to fp0, MC clock                   (06)
 3  Device_Init(fp0): MC, banks, PHY analog, DFI, JEDEC, MR, ZQ (08, 09, 10)
 4  train(pp0)                    pass 1: cold PHY, default map (15)
 5  read MR5 / MR8 (out of SR)    width, density, size
 6  if MR8[7:6] != 0  (x8):
        MC reset (pmu+0xb0 bit 21)                              (06)
        Device_Init(fp0) with the x8 overlays                   (21)
 7  adjust the address map                                      (12)
 8  fill the DFC table (once)                                   (13)
 9  train(pp0)                    pass 2
10  DFC(1); dummy MRR; train(pp1) pass 3: 1066 MT/s
11  DFC(2); dummy MRR; train(pp2) pass 4: 2400 or 2666 MT/s
12  [16 GiB only: MRW MR21]                                     (21)
13  DFC(final)                    already-trained point, no new training
14  return 0
```

LPDDR4 (non-X) inserts `DFC(3); train(pp3)` (2133 MT/s) **after** pp1 and
**before** pp2 — not between the two fp0 passes
([configurations.md](configurations.md)).

For a 1333 final rate the final fp is 0 with the 2666 VCO; there is no 1333
`data_rate` entry ([boot-interface.md](boot-interface.md)). The requested final
rate also selects the PLL1 VCO, which sets the rates at which the *other*
passes run: 1066/1200/2400 use the 2400 VCO (fp0 = 1200, fp2 = 2400), whereas
1333/2666 use the 2666 VCO (fp0 ≈ 1333, fp2 = 2666); fp1 is always 1066
([clocks-and-pmu.md](clocks-and-pmu.md#the-final-rate-chooses-the-vco-and-the-vco-sets-fp0fp2)).

Mode registers are read only between passes, never inside a pass's
self-refresh.

## Why each step exists

1. **First fp0 pass.** The analog is cold and Device_Init used x16 timings
   (the die width is not known yet). The gate lands around 105–118 taps at
   1200 MT/s. This pass exists so that the bus is alive enough to **read MR8**
   reliably before deciding byte mode and the address map.
2. **x8 re-initialisation.** x8 dies need byte-mode parameters in the MC
   (`+0x13e4`, `+0x3b4` of every bank). Without the MC reset and the second
   Device_Init the gate stays at the cold value; afterwards it is ≈32–40 taps.
   The reset also wipes the taps of the first pass.
3. **Address map.** Programmed from MR8 after the re-initialisation. The DFC
   table snapshots the map, so it must come first.
4. **DFC table.** Loaded once, after the map, before the first HW-DFC.
5. **Second fp0 pass.** Records the fp0 slice (1200, or ≈1333 with the 2666
   VCO) with byte mode and the final map. Always present: on x16 there is no
   second Device_Init, but there is still a second fp0 pass (after the map and
   the table). Never skip it on x8. A final rate of 1333 uses this slice and
   ends at fp0.
6. **fp1 and fp2 passes.** Independent slices. Without them 1066 and
   2400/2666 would run with reset delays (`0xf`).
7. **Final DFC.** Selects the trained final point: fp2 for 2400/2666, fp1 for
   1066, fp0 for 1200/1333 (where 1333 uses the 2666 VCO on `PLL1 / 2`). It
   does not train again. For 2400/2666 this is a second DFC(2), which is
   idempotent.

## What must hold between steps

| Before | Condition |
|---|---|
| Device_Init | PLLs locked; `dclk` at fp0; `pmu+0xe8[1:0] = 3`; `ddr+0x44` holds writes |
| First pass | Device_Init verified: STATUS `0x33` (2 ranks), `PHY_INIT = 0x100`, `CTL_FREQ = 0x00800400`, `PI_STATUS = 0x80008000` |
| MR8 read | Pass 1 exited SR (`STATUS & 0x44 == 0`) |
| Map adjustment | MR8 decoded; x8 re-init done if needed |
| DFC table | Map adjusted (the snapshot captures it) |
| Each HW-DFC | Previous pass exited SR; table complete |
| Each pass after a DFC | DFC verified (`CTL_FREQ`, `PI_CFG`, clksel, STATUS `0x33`) |
| Return | Final DFC verified; STATUS `0x33` (or `0x11`), never `0x77` |

## State at the end of the sequence (reference board, 2400 MT/s)

| Item | Value |
|---|---|
| STATUS | `0x33` |
| `CTL_FREQ` | `0xa0800400` (fp2) |
| `PI_CFG` | `0x8` |
| `pmu+0x3b4` | `0x00033b40` |
| `PHY_STATUS` | 0 |
| MR5/6/8/12 | `0x01 / 0x08 / 0x52 / 0x5d` on all four (rank, channel) pairs |
| MR14 | Trained value (`0x1e`–`0x1f` at 2400) on all four pairs |
| Capacity | 8192 MB |

Each HW-DFC completes in 25–30 µs; each PI operation in ≤17 µs.

On the 16 GiB Micron board the end state has the same structure with
16 GiB-specific register values: `PHY_STATUS = 0`, capacity 16384 MB,
MR5/6/8 = `0xff / 0x07 / 0x58` (x8, 16 Gb) on all four (rank, channel) pairs,
MC address map `0x00110001` / `0x05030832` (`+0x20c = 2`), and MR21 written
after the training passes (`CA_CMD = 0x13020095`).

For a 1333 final rate the end state is the fp0 state with the 2666 PLL1 words:
`CTL_FREQ 0x00800400`, clksel `0x00033b50`, `PI_CFG 0`.

## Failure points

| Step | Detection | Measured signature |
|---|---|---|
| PLL lock | `pmu+0x3b4 & 0x30000` not set within 1 ms | CLKSEL `0x00003040` (PLL enables cleared) |
| SW-FC | bit 26 of `pmu+0x04` not clearing within 10 ms | — |
| MC clock | `ddr+0x44` does not hold a written value | — |
| DFI init | PI_STATUS bit 31 not set within 10 ms | — |
| JEDEC init | STATUS bits 0/4 not set within 10 ms | — |
| Training | PI timeout, gate timeout, `PHY_STATUS != 0` | — (never observed on the reference board) |
| MR read | MR_DATA bit 31 not set within 10 µs | — |
| Map | Unsupported MR8 width/density | — |
| DFC | bit 22 of `pmu+0x04` not clearing within 10 ms | `pmu+0x04 = 0x00400000`, STATUS `0x77`, `CTL_FREQ` unchanged |

On any failure, return a negative value and do not continue. The SPL must not
use the DRAM (the mainline SPL cannot tell, see
[boot-interface.md](boot-interface.md)).

## Gaps

- Whether the first fp0 pass could be replaced by a cheaper way of reading
  MR8 reliably on a cold PHY is not explored.
- The LPDDR4 ordering (pp3 after pp1) is derived but not measured on a
  board.

---

[← Previous: CA Training](ca-training.md) | [Index](README.md#index) | [Next: Configurations →](configurations.md)
