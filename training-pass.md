<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: The PI Training Engine](pi-training-engine.md) | [Index](README.md#index) | [Next: Write Leveling →](write-leveling.md)

# One Training Pass

A **training pass** centres the data eye for **one** frequency point that is
already running (`boot_pp`). The boot runs several passes, one per frequency
point (and fp0 twice); the orchestration is in
[boot-sequence.md](boot-sequence.md).

After Device_Init the bus works but the eye is not centred:

- DQS is not levelled with CK;
- the PHY DQS gate is not inside the read burst window;
- the read Vref (PHY) and write Vref (die, MR14) are not at the best step for
  the rate.

At 2400/2666 MT/s the eye is closed without this. Device_Init does not do it.

## Inputs

| Input | Values | Role |
|---|---|---|
| `ddr` | MC base | `0xc0000000` |
| `cs_num` | 1 or 2 | Ranks actually populated |
| `boot_pp` | 0..3 | The frequency point / PHY slice already active |
| log | optional | Text output in SRAM; may be absent |

The pass must not depend on a caller buffer and must not read the
`0xc0800000` block.

## Outputs

For the frequency point `boot_pp`:

- write-leveling taps per byte and rank (`byte + 0x24`);
- read-gate code per byte and rank (`byte + 0x70`);
- RX DQ taps (5 bits) and TX DQ/DM taps (6 bits);
- the PHY Rx Vref index in VREF0/VREF1 of the slice;
- the DRAM Vref(DQ) (MR14), equal to `CTL_TXVREF[21:16]`;
- the return value: `PHY_STATUS` (`phy+0x18000`).

Each slice keeps the taps of **its** rate. Training only the final rate leaves
the other frequency points with reset delays (`0xf`).

## What a pass does not do

- It does not touch the PLL and does not choose the rate.
- It does not program ODT, MR1–MR3, timings or the address map.
- It does not write `CTL_FREQ` or `PI_CFG`.
- It does not use DRAM for code, stack or buffers.
- It does not implement edge search itself (that is the PI's job).
- It does not read the device tree.

## Preconditions

Before entering SR and triggering the PI:

1. `dclk` stable at the rate of `boot_pp` (SW-FC for the first fp0 pass,
   HW-DFC for the others).
2. `CTL_FREQ[31:30] = boot_pp` and `[29:28] = sel` (equal in the normal boot).
3. `PI_CFG[3:2] = boot_pp` (set by init/DFC, never by the pass).
4. The PHY slice `phy + (boot_pp << 14)` is the active one.
5. MC, PHY and mode registers programmed by Device_Init.
6. `cs_num` equals the number of real ranks.
7. The DRAM is out of SR (STATUS SR bits clear).
8. After a HW-DFC, mode-register reads (if any are done before the pass)
   start with a dummy MRR ([mode-registers.md](mode-registers.md)).

If any of these fails, triggering the PI is pointless. The PI does not choose
the frequency. `cs_num = 1` on a two-rank package leaves CS1 with reset delays.

## Mandatory sequence

```
enter SR (both ranks)
PHY_INIT = 512
[CA training only if a product requires it — the LPDDR4X boot does not]
write_leveling        per rank
read_gate             per rank; a timeout fails the boot
read_training         sweep PHY Rx Vref 0..15; max-min; re-apply winner
write_training        sweep MR14 21..36;      max-min; re-apply winner
st = PHY_STATUS
exit SR
return st
```

### Why this order

| Rule | Reason |
|---|---|
| SR first | Without SR a CPU access can collide with the PI. |
| `PHY_INIT = 512` next | Arms the training FSM for this pass. |
| WL before the gate | The gate measures DQS relative to a DQS already aligned with CK. |
| Gate before read training | Without a gate, read training gets margin 0 or times out. |
| Read before write | Rx Vref (PHY) and Tx Vref (die) are independent, but the PI's CFG/STATUS leftovers on this SoC expect this order. |
| Re-apply the winning Vref while still in SR | The last sweep step would otherwise leave the wrong taps in the PHY. |
| Exit SR at the end | The boot needs the DRAM for MR reads, the DFC and the next pass. |

### Forbidden

- Reordering the stages.
- Training two frequency points at once.
- Leaving SR in the middle of a PI-driven sweep.
- Initialising MC/PHY/PLL inside a pass (that belongs to `ddr_init`
  between passes).
- Reading or writing DRAM during a PI-driven pass while in SR.
- Forcing taps to static "nice" values in software without measuring or training
  them (note: while the PI sequencer is the fast ~20 ms production path, all
  four stages can also be swept in software outside SR using memory-pattern
  tests, taking ~20 s, with per-lane RX centring required at fp2; see
  [write-leveling.md](write-leveling.md), [read-gate.md](read-gate.md), and
  [eye-training.md](eye-training.md#independent-verification-of-the-eye-memory-pattern-sweep)).

## Error handling

`PHY_STATUS` (`phy+0x18000`):

| Bit | Meaning |
|---|---|
| 1 | Write leveling failed |
| 2 | Read leveling failed |
| 3 | Write DQ failed |
| 4 | Read gate failed |

- The boot must **fail** if `PHY_STATUS != 0` at the end of the pass, or if the
  gate times out. Continuing with a bad DRAM is worse than stopping.
- `PI_STATUS = 0x80008067` at the end is not an error: those are sticky done
  bits plus the idle bits 15/31.
- Every PI wait needs a software timeout ([pi-training-engine.md](pi-training-engine.md)).
- SR: never wait for STATUS `0x33` after the exit; STATUS `0x77` at the end
  means the pass is still in SR ([self-refresh.md](self-refresh.md)).

Things not to do:

- poll `PI_CMD` (it reads 0);
- wait for `PI_STATUS` to equal an absolute value;
- assume a leftover makes a wait trivially satisfied;
- address the harvest with an absolute `0xc0058xxx`;
- write the Vref of slice N at the offsets of slice N+1;
- train only the final rate;
- skip the second fp0 pass;
- treat PI_STATUS bits 15/31 as errors.

## Working memory

Code, stack and scratch are in SRAM. The minimum scratch is 16 minimum values
for the current sweep and 32 margins per rank for the current step: the whole
sweep need not be stored if the minimum is updated on the fly and the re-kick
uses the winning step. Zero the scratch at the start of the pass.

## Stage-by-stage register specification

Rather than prescribing a specific code structure, the table below summarises
the exact hardware state transitions and MMIO operations required for one pass
at `boot_pp` (`slice = phy + (boot_pp << 14)`). Detailed rules for each stage
are in [self-refresh.md](self-refresh.md), [write-leveling.md](write-leveling.md),
[read-gate.md](read-gate.md), and [eye-training.md](eye-training.md).

| Step | Stage | MMIO writes & operations | Completion / result |
|---|---|---|---|
| 1 | Enter SR & arm FSM | Enter self-refresh on populated ranks ([self-refresh.md](self-refresh.md)); write `phy + 0x10028 = 0x200` (`PHY_INIT = 512`). | `ddr+0x08` SR bits set (`0x44` for 2 ranks, `0x04` for 1 rank); `phy+0x10028 == 0x200`. |
| 2 | Write leveling | Write `ddr + 0x13e8 = 0x0a641e44` (`PI_WL_CFG`). For each rank `cs = 1..cs_num`, write `ddr + 0x13d0 = 0x10400000 \| (cs << 24)`. | Poll `(ddr + 0x13fc & 0x1) == 0x1` per rank. Trained WL delays in `byte_base + 0x24`. |
| 3 | DQS read gate | For each rank `cs = 1..cs_num`, write `ddr + 0x13d0 = 0x10100000 \| (cs << 24)`. | Poll `(ddr + 0x13fc & 0x2) == 0x2` and check gate timeout ([read-gate.md](read-gate.md)). Gate code in `byte_base + 0x70`, applied delay in `slice + 0x64[4:0]`. |
| 4 | Read eye sweep (RX Vref) | Clear bit 4 of `VREF0` (`slice + 0x3004 &= ~0x10`); write `ddr + 0x13ec = 0x00005070` (`PI_RD_CFG`). Sweep `step = 0..15`: set `[31:16]` of `VREF0` (`slice + 0x3004`) and `VREF1` (`slice + 0x3204`) to `(step << 28) \| (step << 24) \| (step << 20) \| (step << 16)` (preserving `[15:0]`), trigger `ddr + 0x13d0 = 0x10200000 \| (cs << 24)` for each `cs = 1..cs_num`, wait for `(PI_STATUS & 0x6) == 0x6`, and immediately read the 32 DQ margins (8 words) from `phy + 0x18200` to record the minimum margin across all DQs and ranks for `step`. | Clear transient read-leveling error bit 2 (`phy + 0x18000 &= ~0x4`), select the `step` with the largest minimum margin, write that winning `step` to `VREF0` and `VREF1`, re-run `0x10200000 \| (cs << 24)` for all ranks so the PHY latches the winning RX DQ/DM taps (`byte_base + 0x30..0x50`), and restore `VREF0 \|= 0x10`. |
| 5 | Write eye sweep (TX MR14) | Write `ddr + 0x13f8 = 0x00001810` (`PI_WR_CFG`). Sweep 16 candidate MR14 values `vdq` (e.g. `21..36` on LPDDR4X): update `ddr + 0x30c[21:16] = vdq`, issue MRW MR14 (`ddr + 0x24 = 0x1302000e` for 2 ranks or `0x1102000e` for 1 rank), trigger `ddr + 0x13d0 = 0x10080000 \| (cs << 24)` for each `cs = 1..cs_num`, wait for `(PI_STATUS & 0x60) == 0x60`, and immediately read the 32 DQ margins (8 words) from `phy + 0x18300` to record the minimum margin across all DQs and ranks for `vdq`. | Select the `vdq` with the largest minimum margin, write it to `ddr + 0x30c[21:16]`, issue MRW MR14 (`ddr + 0x24`), and re-run `0x10080000 \| (cs << 24)` for all ranks so the PHY latches the winning TX DQ/DM taps (`byte_base + 0x00..0x20`). |
| 6 | Status check & exit SR | Read `st = readl(phy + 0x18000)` (`PHY_STATUS`), exit self-refresh ([self-refresh.md](self-refresh.md)), and return `st`. | `st == 0` indicates success; any nonzero value must abort the boot. |

After the pass, outside SR: optionally read MR14 on every (rank, channel) and
check it equals `CTL_TXVREF[21:16]` (after a dummy MRR if a DFC happened since
the last read).

## Gaps

- The order constraint "read before write" is required by observed engine
  state, not explained.
- Whether a pass can safely skip SR (the engine works without it in some
  states, [self-refresh.md](self-refresh.md)) is not established; the
  boot always uses SR.

---

[← Previous: The PI Training Engine](pi-training-engine.md) | [Index](README.md#index) | [Next: Write Leveling →](write-leveling.md)
