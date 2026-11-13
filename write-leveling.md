<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: One Training Pass](training-pass.md) | [Index](README.md#index) | [Next: DQS Read Gate →](read-gate.md)

# Write Leveling

## Goal

Align DQS with CK at the DRAM, for each rank and each byte. The software does
not compute the delay: it asks the PI, and the result lands in `byte + 0x24` of
the active slice ([phy.md](phy.md)).

## Recipe

```
PI_WL_CFG (ddr+0x13e8) = 0x0a641e44        /* once, before the first rank */
for cs in 1 .. cs_num:
    PI_CMD (ddr+0x13d0) = 0x10400000 | (cs << 24)
    wait (PI_STATUS & 0x1) == 0x1          /* software timeout */
```

- The issue of the command clears bit 0 of `PI_STATUS` (measured: leftover
  `0x80008067` → `0x80008066` right after the command), so the wait is real
  even when the previous pass left bit 0 set.
- There is no pass/fail code of its own. A failure shows up as bit 1 of
  `PHY_STATUS` (`phy+0x18000`) at the end of the pass.

## Parameters

`PI_WL_CFG` (`ddr+0x13e8`) resets to `0x0a301e44` (byte `[23:16] = 0x30`).
Training needs `0x0a641e44` (byte `[23:16] = 0x64`) written **before** the first
rank. The subfields of this word are sequencer parameters whose exact hardware
meaning is *opaque* ([pi-training-engine.md](pi-training-engine.md)).

## Effects on the PHY

- Only the `+0x24` word of each byte/rank block changes.
- TX DQ (`+0x00`) and RX DQ (`+0x30`) taps do **not** change in this stage;
  they stay `0xf` until the eye training. Do not write them.
- Each rank has its own DQS path: skipping CS1 leaves the second rank with its
  reset WL value.

## Typical results (reference board)

WL taps, bytes b0 b1 b2 b3:

| Rate | CS0 | CS1 |
|---|---|---|
| 1200 MT/s | 18 21 17 22 | 17 20 17 21 |
| 1066 MT/s | 18 20 17 22 | — |
| 2400 MT/s | 19–20 22 20 23 | 19 21–22 20 23 |
| 2666 MT/s | 19 21 19 23 | 19 21 20 22 |
| ≈1333 MT/s (fp0 with the 2666 VCO), cold first pass | 17 20 16 21 | — |

Range 17–23 taps, different per byte and per rank. Acceptance: distinct per
byte and rank, inside ≈17–23 at the final frequency point.

On the 16 GiB Micron board the same stage produces, per pass:

| Pass / rate | CS0 | CS1 |
|---|---|---|
| fp0 cold / 1200 MT/s | 20 23 19 22 | 19 22 19 22 |
| fp0 second / 1200 MT/s | 20 22 19 22 | 19 23 19 22 |
| fp1 / 1066 MT/s | 20 23 19 22 | 20 23 19 22 |
| fp2 / 2400 MT/s | 21 24 20 23 | 21 24 19 23 |

The values, per-byte spread and acceptance range (19–24) match the 8 GiB
reference board.

## Independent measurement of the WL window (memory-pattern sweep)

The WL delay can be characterised without the PI, like the Rx/Tx delay words:

1. save the `byte + 0x24` word;
2. for each code, write `(save & ~0x3f) | code` and run the memory-pattern test
   (write 4 MiB, read back 3 MiB so the checked part is out of cache);
3. the codes with 0 errors form the window; its centre is the sampling point;
4. restore the saved word and re-check (0 errors).

Measured on the 8 GiB reference board (CS0) and on the 16 GiB Micron board (CS1 row), per byte (codes of the 5-bit field):

| Pass | byte 0 | byte 1 | byte 2 | byte 3 | PI left |
|---|---|---|---|---|---|
| 1200 MT/s (fp0) | 8..27 (20) | 11..31 (21) | 6..29 (24) | 8..31 (24) | 18 / 21 / 17 / 22 |
| 1066 MT/s (fp1) | 6..29 (24) | 9..31 (23) | 6..30 (25) | 7..31 (25) | 18 / 21 / 18 / 22 |
| 2666 MT/s (fp2) | 15..22 (8) | 16..24 (9) | 15..23 (9) | 16..27 (12) | 19 / 21 / 20 / 23 |
| 2666 MT/s, CS1 (16 GiB) | 15..23 (9) | 18..25 (8) | 13..24 (12) | 16..27 (12) | 20 / 23 / 19 / 22 |

Notes:

- The window is contiguous within the 5-bit field; writing codes 32..63 reproduces
  the same pattern 32 codes later, which shows that the hardware ignores bit 5.
- The PI's value lands within 2 codes of the measured centre in every byte and
  pass, so WL can be centred by software with the same rule used for Rx/Tx.
- The width differs per byte (byte 2 and 3 are wider than 0 and 1) and halves at
  2666, like the Rx/Tx windows ([eye-training.md](eye-training.md#at-2666-mts)).
- Rank 1 (CS1) is *not* reachable at `0x10000000` or `4 GiB + 256 MiB`, because
  those CPU addresses belong to CS0. The rank boundary in CPU address space
  depends on the active MC map: **6 GiB** on the 8 GiB final map (and on the
  unadjusted default map during the first fp0 pass) and **10 GiB** on the
  16 GiB final map
  ([address-map.md](address-map.md#rank-boundary-in-cpu-address-space-measured)).
- **CS1 WL window (measured on both boards):** using a true CS1 CPU address
  (6 GiB on the default map / 11 GiB on the 16 GiB final map, and 7 GiB on the
  8 GiB final map), the CS1 WL window and the PI's trained taps match CS0:
  - on the **16 GiB Micron** board, CS1 matches CS0 identically: at fp0/1200
    widths are `22/21/23/22` codes (PI `19/22/19/22`) on both ranks, and at
    fp2/2666 widths are `9/8/12/12` codes (PI `20/23/19/22`) on both ranks;
  - on the **8 GiB Samsung** board, CS1 is within ±1 code and ±1 PI tap of CS0:
    at fp0/1200 CS0 is `20/21/25/24` and CS1 is `20/21/23/25`; at fp2/2400 CS0
    is `10/11/11/12` and CS1 is `10/11/12/12`.
  Together with the CS1 Rx/Tx eye sweeps
  ([eye-training.md](eye-training.md#on-the-16-gib-micron-board-measured-1066-final)),
  all three delay stages are symmetric between CS0 and CS1.

## Gaps

- Meaning of the WL CFG fields.
- Unit of the WL code: the window in codes is similar at 1066 and 2400
  (20–25), which suggests a fraction of the UI like the Rx/Tx delay lines, but it
  was not confirmed against a fixed ps reference.
- WL behaviour on x16 dies and on DRAM vendors other than Samsung and Micron is
  not measured.

---

[← Previous: One Training Pass](training-pass.md) | [Index](README.md#index) | [Next: DQS Read Gate →](read-gate.md)
