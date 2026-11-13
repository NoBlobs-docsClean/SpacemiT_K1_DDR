<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: DQS Read Gate](read-gate.md) | [Index](README.md#index) | [Next: CA Training →](ca-training.md)

# Eye Training: Read and Write Vref Sweeps

Both eye-training stages follow the same pattern:

1. for each of 16 Vref steps, set the Vref and run the PI operation on every
   rank;
2. after each operation, copy the per-DQ margins from the harvest and update
   the minimum of that step;
3. choose the step with the **max-min** rule;
4. set the winning Vref and **run the PI operation again** so the PHY ends up
   with the taps of the winning step.

The difference is which Vref is swept:

| Stage | Vref swept | Range | PI operation | Done mask | Harvest |
|---|---|---|---|---|---|
| Read (Rx) | PHY Rx Vref, COMMON VREF0/VREF1 | index 0..15 | RD `0x10200000` | `0x6` | `phy+0x18200` |
| Write (Tx) | DRAM Vref(DQ), MR14 | 21..36 | WR `0x10080000` | `0x60` | `phy+0x18300` |

## Read training (Rx Vref + RX DQ delays)

With a fixed Vref, the PI leaves:

- margins in harvest RX (0 = eye closed on that DQ);
- RX DQ taps in `byte + 0x30 + 4*lane` (mask 31).

```
VREF0 = slice + 0x3004 ; VREF1 = slice + 0x3204

VREF0 &= ~0x10
PI_RD_CFG (ddr+0x13ec) = 0x00005070
for step in 0 .. 15:
    v = (VREF0 & 0xffff) | PACK_VREF(step)
    VREF0 = v ; VREF1 = v
    min[step] = +inf
    for cs in 1 .. cs_num:
        PI_CMD = 0x10200000 | (cs << 24)
        wait (PI_STATUS & 0x6) == 0x6
        copy harvest RX now            /* the next rank overwrites it */
        min[step] = min(min[step], all 32 DQ margins)
PHY_STATUS &= ~(1 << 2)                /* clear read-leveling error before choosing */
best = max_min(min)
VREF0 = VREF1 = (VREF0 & 0xffff) | PACK_VREF(best)
for cs in 1 .. cs_num:
    RD operation                       /* programs the PHY at the winning Vref */
VREF0 |= 0x10
```

`PACK_VREF(s) = (s<<28) | (s<<24) | (s<<20) | (s<<16)`; for example
`PACK_VREF(5) = 0x55550000`, which is also the init value. The low 16 bits are
preserved by the read-modify-write.

Notes:

- **16 steps** are needed: at 2400/2666 MT/s, indices 13–15 close the eye
  (margin 0 on every DQ), and the "typical" 0.3·VDDQ (index 5) is not the best
  on every board.
- **VREF0 and VREF1** are the two sub-PHYs of the same slice. Writing
  `slice + 0x1000 + 0x3004` would land in the next slice.
- **Bit 4** is cleared only during the sweep: observed `0xa937` → `0xa927`
  → at the end `0xa937` with `[31:16]` = the winner (VREF1 keeps bit 4 clear).
- **Clear `PHY_STATUS` bit 2 before choosing**: the sweep can set the
  read-leveling error bit (seen at 2666) at bad Vref steps even though the
  re-kick at the good Vref is valid. Do not clear the other error bits here.
- **Re-kick**: the last loop step leaves the taps of step 15 in the PHY; the
  final RD operations restore the winner's taps.
- The worst DQ is **not** a fixed pin (on this board it moves among DQ 0, 1, 16,
  17, 19, 20), which is why a per-step minimum over all DQs is used rather than
  a static mask.
- Read training does not touch MR12 / Vref(CA).
- Printing all 16 margin matrices during boot only lengthens the init.

## Write training (Tx Vref = MR14 + TX DQ/DM delays)

Same structure, but the Vref is the **DRAM's** Vref(DQ) (MR14), not a PHY
index. The PI centres TX DQ and DM.

```
seed[i] = 21 + i                       /* 21..36, fits in 6 bits */
PI_WR_CFG (ddr+0x13f8) = 0x00001810
for i in 0 .. 15:
    CTL_TXVREF (ddr+0x30c) = (CTL_TXVREF & 0xffc0ffff) | (seed[i] << 16)
    CA_CMD (ddr+0x24) = 0x1302000e     /* MRW MR14 while in SR (0x1102000e with 1 rank) */
    min[i] = +inf
    for cs in 1 .. cs_num:
        PI_CMD = 0x10080000 | (cs << 24)
        wait (PI_STATUS & 0x60) == 0x60
        copy harvest TX now
        min[i] = min(min[i], all 32 DQ margins)
best = seed[max_min(min)]
CTL_TXVREF[21:16] = best
CA_CMD = 0x1302000e
for cs in 1 .. cs_num:
    WR operation
```

Notes:

- The MR14 command is required at **every** step: the eye being measured is the
  die's. Changing only `CTL_TXVREF` measures the old Vref.
- **Why 21..36:** a 16-point window of the LPDDR4X Vref(DQ) range starting at
  21; the post-init MR14 on this board is `0x19` (25), near the middle. The
  starting MR14 value does not enter the max-min; the sweep in SR writes every
  value itself, so no MR14 seed is needed after a DFC.
- After leaving SR, MR14 read from every (rank, channel) must equal
  `CTL_TXVREF[21:16]` (after a dummy MRR if a DFC happened,
  [mode-registers.md](mode-registers.md)).
- Typical result at 2400 MT/s: the margin-10 plateau starts at index 8–10
  (MR14 29–31) and the first maximum is chosen. At 2666 the eye is 8–9 and the
  first maximum often falls around 23–25.
- Sanity: TX taps ≈30–36 and RX taps ≈17–19 at the final fp on this board. A
  TX tap still `0xf` means the operation did not run. Do not write these
  numbers by hand.
- LPDDR4 (non-X) uses a different MR14 range (range bit 6); 21..36 must not
  be reused there ([configurations.md](configurations.md)).

## Max-min Vref selection

The same rule serves both sweeps (Rx index 0..15, Tx seed 21 + i):

```
for step in 0 .. 15:
    min[step] = min(margin[step][cs][dq] for every cs, every one of the 32 DQ)
best = 0
for step in 1 .. 15:
    if min[step] > min[best]:          /* strict '>' */
        best = step
```

Rules:

- No interpolation between steps.
- DM margins are not weighted: they may be stored, but do not enter the
  minimum.
- No averages; no "best DQ"; not only CS0.
- Copy the harvest **per rank**, not once per step (it latches only the last
  operation).

Why max-min: one bad DQ decides. An average hides a pin at 0 while the others
are at 20, and the worst DQ moves among pins on this board.

Plateau tie-breaking: when several consecutive Vref steps share the same
maximum minimum margin, any step on the plateau places the sampling point well
inside the eye. Keeping the first step that achieves the maximum (`>`) or
picking the middle of the plateau both work; the reference measurements in
[reference-measurements.md](reference-measurements.md) reflect strict `>`
(first maximum).

All minimums 0 (eye closed across the whole band): such a pass is a hardware
training failure (`PHY_STATUS != 0`). If a fallback MR14 is written before
aborting, `0x19` (25, the Device_Init default) is the nominal centre of the
21..36 sweep window.

## Typical results (reference board)

| Pass | Rate | Rx index (min margin) | Tx MR14 (min margin) |
|---|---|---|---|
| fp0, first pass | 1200 | 5–6 (24) | 26–36 (20–21) |
| fp0, second pass | 1200 | 5–6 (24) | 26–36 (20–21) |
| fp0, 2666 VCO | 1333 | 4 (21) | 28–29 (17) |
| fp1 | 1066 | 3–4 (26) | 31–35 (24) |
| fp2 | 2400 | 5–6 (10) | 29–31 (10) |
| fp2 | 2666 | 5–7 (8–9) | 23–35 (8–9) |

Rx per-step minimums at fp2 / 2400 MT/s across six repeated passes:
`7 7 8 8–9 9 9 10 10 9 9 8 7 6–7 0 0 0` (winner always index 6).
More data in [reference-measurements.md](reference-measurements.md).

## Independent verification of the eye (memory-pattern sweep)

The PI reports margins in its own unit; the eye can also be measured directly
from the delay words, without the PI, by moving data:

1. save the delay words of the lanes being measured;
2. for each candidate code, write it to the lane(s), write a pattern large
   enough to defeat the data cache (4 MiB was enough on this board: write 4 MiB,
   read back 3 MiB so the checked part has been evicted), and count mismatching
   words;
3. the codes with 0 mismatches form a contiguous window; its centre is the
   sampling point and its width is the margin in delay codes;
4. restore the saved codes and re-check (0 mismatches expected).

Measured this way at CS0, all lanes of a byte group at the same code:

| fp | Rate | RX window (of 32) | TX window (of 48) |
|---|---|---|---|
| 0 | 1200 MT/s | 7..28 (22) | 24..42 (19) |
| 1 | 1066 MT/s | 5..29 (25) | 22..43 (22) |
| 2 | 2400 MT/s | 14..21 (8) | 31..39 (9) |

The byte-level window is the **minimum over lanes**, not an average. Per byte
group at 1200 MT/s the RX windows are 22–23 (bytes 0), 24–27 (bytes 1), 28–30
(bytes 2) and 29–32 codes (bytes 3); inside one group the lanes differ by at
most 2 codes. The narrowing from group 0 to group 3 is the same at 1066, so it
belongs to the board routing, not to the training.

### At 2666 MT/s

With the PLL1 VCO at 2666 (fp0 ≈1333, fp2 2666) the same sweep gives:

| fp | Rate | RX window | TX window |
|---|---|---|---|
| 0 | ≈1333 MT/s | 9..28 (20 of 32) | 25..41 (17 of 48) |
| 1 | 1066 MT/s | 5..30 (26) | 22..44 (23) |
| 2 | 2666 MT/s | 15..21 (7) | 31..36 (6) |

Per byte group at fp2/2666 the RX windows are 7–9 (group 0), 10–11 (group 1),
15–16 (group 2) and 13–32 codes (group 3). The Vref sweep at fp2 gives widths
`2 3 5 5 5 6 6 7 6 6 5 4 3 0 0 0` for Rx indices 0..15 and
`6 6 6 6 6 6 6 6 5 5 5 5 5 5 5 5` for write codes 21..36: the maximum is one
code above the value the PI chose, i.e. inside the noise.

Two cautions measured at 2666 on this board:

- the PI's write Vref at fp2 was **not stable across boots** (code 21 in one
  run, 35 in another): with a 5–6 code window the max-min ties and the
  tie-break lands elsewhere;
- a window the sweep accepted can still fail later: re-applying the "widest"
  Rx point (index 7, delay 18) produced 46 wrong words out of ≈786 000
  (0.006 %) in the confirm step. At this margin a single pattern pass is not
  conclusive — repeat it or use a larger sample before trusting the edge.

### On the 16 GiB Micron board (measured, 1066 final)

Running the same Vref → window → centre sweep on the 16 GiB Micron board with
the 2400 VCO (1066 MT/s final rate) gives:

| fp | Rate | RX window (of 32) | TX window (of 48) |
|---|---|---|---|
| 0 | 1200 MT/s | 6..29 (24) | 25..46 (22) |
| 1 | 1066 MT/s | 4..30 (27) | 24..47 (24) |
| 2 | 2400 MT/s | 13..22 (10) | 33..40 (8) |

The centres are 17/17/17 (RX) and 35/35/36 (TX), within 0–1 code of the delays
left by the PI (18/17/17 and 34/34/35). The windows are 1–3 codes wider than on
the 8 GiB Samsung board.

Two characteristics observed on the Micron board:

- **Flat Tx Vref plateau:** at 1200 MT/s the Tx window is 21–22 codes across all
  MR14 codes 21..36; at 1066 MT/s it is 24 codes across 21..32 and 23 across
  33..36. Because the curve is a flat plateau rather than a slope, the PI's
  chosen MR14 varies between boots (21, 22, 24 and 26 observed at fp0) due to
  tie-breaking. Selecting the centre of the plateau avoids edge sensitivity.
- **Per-lane centring required at fp2:** at fp2 a single byte-wide RX centre can
  fail confirmation: the widest common Rx point (index 6, window 13..22, common
  centre 17) produced 850 wrong words out of ≈786 000 (0.11 %) on one run and
  370 on a repeat, while restoring the PI's per-lane taps gave 0 errors.
  Sweeping each lane individually shows why: group g0 (DQ0–7 + DM0) is the
  limiting byte group in every pass (24–25 codes at 1200, 26–28 at 1066, 11–12
  at fp2), with g0 < g1 < g2 ≈ g3 as on the 8 GiB board, and the per-lane fp2
  windows (11–16 codes) are wider than the 10-code common intersection.
  **Centring each lane in its own window** (after picking the Vref from the
  common window) eliminates all errors at fp2 at both 2400 and 2666 MT/s,
  confirmed across three distinct data patterns (`0 errors` on all three).
- **Per-lane Tx windows at fp2:** 11–14 codes at 2400 MT/s and 5–6 codes at
  2666 MT/s, with the PI taps at 33–38. The common-window Tx centre still passes
  at 2666 MT/s (6-code window, centre 35), so per-lane Tx centring is optional.

Repeating the per-lane RX centring on the **8 GiB Samsung board** confirmed the
same rule: at fp2/2400 the common centre already passed, and at fp2/2666 the
common centre's 46-word failure above was completely eliminated (`0 errors`
across all three patterns) once each lane was centred in its own window
(per-lane Tx windows there: 11–13 codes at fp2/2400 and 10–11 at fp2/2666).
Thus per-lane RX centring is required on both boards whenever the fp2 eye is
narrow.

With the **2666 VCO** (final rate 1333 or 2666 MT/s; fp0 ≈ 1333, fp2 = 2666) on
the 16 GiB Micron board, the independent sweep gives:

| fp | Rate | RX window (of 32) | RX centre | TX window (of 48) | TX centre |
|---|---|---|---|---|---|
| 0 | ≈1333 MT/s | 8..28 (21) | 18 | 25..44 (20) | 34 |
| 1 | 1066 MT/s | 4..30 (27) | 17 | 24..47 (24) | 35 |
| 2 | 2666 MT/s | 13..21 (9) | 17 (per-lane: 15–18) | 33..38 (6) | 35 |

At fp2/2666 the byte-wide Rx centre fails confirmation (466 wrong words) while
per-lane centring passes with 0 errors; the Tx common-window centre passes with
a 6-code window.

**Per-rank (CS1) symmetry:** earlier software sweeps probed `4 GiB` for CS1,
which is still inside CS0 in CPU address space on both boards. Repeating the
Rx/Tx sweeps at the true CS1 CPU address on the final map (**10 GiB** on the
16 GiB board, **6 GiB** on the 8 GiB board, corresponding to DRAM offsets 8 GiB
and 4 GiB; see
[address-map.md](address-map.md#rank-boundary-in-cpu-address-space-measured))
shows that CS1 matches CS0 within ±1 code in Rx and identically in Tx on both
boards and both VCOs (e.g. at 16 GiB / 2400: Rx `13..21` on both ranks, Tx
`34..40` on both ranks). An independent per-rank sweep can therefore use the
exact same algorithm on CS0 and CS1.

### Agreement with the PI

The centre of the measured window matches the taps the PI leaves: RX 17.5 vs
18 and TX 33 vs 33–36 at 1200 MT/s. The PI centres **per lane** (all trained RX
taps at 1200 are 16–19), so an implementation that centres the whole byte at
once loses the lanes whose own window is wider (groups 2 and 3).

### Rx Vref shifts the window

The window is not independent of the Rx Vref index. At 1200 MT/s (fp0), sweeping the
index from 0 to 14 (step 2) gives:

| index | 0 | 2 | 4 | 6 | 8 | 10 | 12 | 14 |
|---|---|---|---|---|---|---|---|---|
| window | 7..25 | 7..26 | 7..27 | 7..28 | 8..29 | 9..29 | 11..29 | 16..30 |
| width | 19 | 20 | 21 | 22 | 22 | 21 | 19 | 15 |

The window moves right and narrows as the Vref rises, so choosing the Vref
without re-centring the delay leaves margin unused. This also explains why the
window-width criterion and the harvest max-min can pick neighbouring indexes
(the widest window is at 6–8, the PI chose 5–6 on this board) while both end up
with the sampling point inside the eye.

### Tx Vref breadth

The same sweep over the write Vref (MR14 codes 21..36) gives widths that grow
with the code at 1200 and 1066 (18→19 and 20→23), with no interior maximum in
that range on this board; at fp2 there is a shallow maximum at 28–32 (9 codes).

### Cost

Each probe moves 4 MiB in and 3 MiB out of DRAM. A full 16-step sweep over 32
codes and three passes costs ≈8 s of boot time (RX) and ≈12 s (TX with 48 codes),
three orders of magnitude more than the PI, which does the same work in
hardware. Use this method to validate a training result or to characterise a
board, not as the production path.

### Harvest margin vs delay window

The harvest margin is the PI's own eye measure. Sweeping the Rx Vref and
measuring both the margin and the delay window at each step (1200 MT/s, fp0) gives:

| Vref | 0 | 2 | 4 | 6 | 8 | 10 | 12 | 14 | 15 |
|---|---|---|---|---|---|---|---|---|---|
| margin | 22 | 22 | 23 | 24 | 24 | 22 | 21 | 17 | 15 |
| window (codes) | 19 | 20 | 21 | 22 | 22 | 21 | 20 | 16 | 15 |

The two track each other within 0–3 codes across the whole sweep, so the margin
is a **code-unit measure of the same eye**, not a separate physical quantity:
it is comparative (for the max-min rule), not an absolute time. The small offset
comes from the boundary definitions (the window is the byte-lane intersection,
the margin is the PI's per-DQ minimum).

They also disagree at the plateau: the widest window is at Vref 7 and the best
margin at 5/6/8; the PI keeps the first maximum. The sampling point lands inside
the window either way.

## Gaps

- Exact boundary definition of the harvest margin (why it can be 0–3 codes above
  the measured window).
- What the `[15:0]` bits of VREF0/VREF1 control: they do **not** change the delay
  window (measured: `0xa927`, `0xa937`, `0x0000` and `0xffff` give the same window
  within 1 code), so they are control bits (pre/postamble, enables) of unknown
  exact function.
- How the four `PACK_VREF` nibbles map to sub-blocks: they are **not**
  redundant (writing only one of them drops the window from 22 to 19–21 codes),
  but which sub-block each one serves is not established; the sequence always
  writes the same index in all four.
- Absolute voltage of each Rx index and of each MR14 value on this board.
- LPDDR4 write-sweep range.
- At fp2, byte-wide common-window RX centring is too optimistic when the common
  window narrows to ≤10 codes (observed at 2400/2666 MT/s on the 16 GiB board
  and at 2666 MT/s on the 8 GiB board): software sweeps must **centre each lane
  in its own window** after choosing the Vref from the common window.

---

[← Previous: DQS Read Gate](read-gate.md) | [Index](README.md#index) | [Next: CA Training →](ca-training.md)
