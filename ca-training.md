<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Eye Training](eye-training.md) | [Index](README.md#index) | [Next: The Complete `ddr_init` Sequence →](boot-sequence.md)

# CA Training (Engine Present, Unused by the LPDDR4X Boot)

The LPDDR4X boot on this SoC does **not** run CA training. Vref(CA) comes from
the init (`ddr+0x30c[31:24] = 0x9D`, MR12 = `0x5d` on the reference board).
`PI_CA0` arrives from the MC as `0x055d74ff`; the LPDDR4X pass does not write it.

In the LPDDR4X boot: do not turn the CA port on through MISC, do not issue the
CA operation and do not change the PLL inside a training pass.

## Static Vref(CA) sensitivity on LPDDR4X boards (measured)

Even without CA training, the static Vref(CA) programmed through
`ddr+0x30c[31:24]` (`0x9D` on standard LPDDR4X SKUs, corresponding to MR12 =
`0x5d`) can be varied to test command-bus sensitivity at 2666 MT/s:

- On the **16 GiB Micron** board, setting `ddr+0x30c[31:24]` to `0x8D`, `0xAD`
  or `0xBD` boots with `PHY_STATUS = 0`, leaves the DQ training margins
  unchanged (since CA is a separate signal path), and passes a 120 s
  multi-process memory soak with zero data errors.
- On the **8 GiB Samsung** board, `0x8D` and `0xAD` also boot with
  `PHY_STATUS = 0` and pass the 120 s memory soak, but **`0xBD` fails** the
  fp1 (1066 MT/s) write training (all Tx margins 0, `PHY_STATUS = 0x8`) and
  hangs at fp2. Thus the usable static Vref(CA) range on the 8 GiB Samsung board
  extends only up to `0xAD`.

This is a coarse functional check: it shows that the CA bus carries commands
without error around `0x9D` (±`0x10` on both boards), not a measurement of the
CA timing eye, and does not indicate that CA training is needed on LPDDR4X.

## Hardware registers and commands

The CA training hardware exists in the MC/PHY (`PI_CA0`, `PI_CA1`, `CA_STAT`,
and `PI_CMD` operations `CA`, `PROG`, `KICK`), although it has **not been
exercised** on the LPDDR4X reference board and its internal bitfields are
*opaque*.

| Register / Command | Address / Value | Role |
|---|---|---|
| `MISC` CA enable / disable | `ddr+0x28` = `0x13010000` / `0x13008000` | Enables (`0x13010000`) or disables (`0x13008000`) the CA training mode before/after MRW updates |
| `PI_CA0` | `ddr+0x13f0` = `0x05dc74ff` (base) | CA sequencer config 0: bits `[31:28]` = 1-based rank (`cs`), bit 27 toggled during external-clock switch, bits `[26:0] = 0x05dc74ff` (*opaque*) |
| `PI_CA1` | `ddr+0x13f4` = `0xec000f99` | CA sequencer config 1 (*opaque*) |
| `CA_STAT` | `phy+0x18004` = `0x01008870 \| subpass` | Sub-pass select (`subpass` = 0 or 1 in bit 0) |
| `PI_CMD` CA op | `ddr+0x13d0` = `0x10800000 \| (cs << 24)` | Starts CA training for rank `cs`; completion indicated by `PI_STATUS` bit 24 |
| `PI_CMD` PROG op | `ddr+0x13d0` = `0x13000002` | Prepares the PI/PHY for an in-training clock switch; completion on `PI_STATUS` bit 29 |
| `PI_CMD` KICK op | `ddr+0x13d0` = `0x13000004` | Finalizes an in-training clock switch after SW-FC; completion on `PI_STATUS` bit 30 |

### Hardware sequence (unexercised on LPDDR4X)

1. Select the active bank in `CTL_FREQ[31:30] = boot_pp`, enable CA mode via
   `MISC = 0x13010000`, and issue `CA_CMD = 0x13020000 | mr` for MRs
   `13, 1, 2, 3, 11, 12, 14, 22`.
2. Write `PI_CA0 = 0x05dc74ff` and `PI_CA1 = 0xec000f99`.
3. For each rank `cs` (`1 .. cs_num`), set `PI_CA0[31:28] = cs` and run two
   sub-passes (`subpass = 0, 1`):
   - write `phy+0x18004 = 0x01008870 | subpass`;
   - trigger `PI_CMD = 0x10800000 | (cs << 24)`;
   - perform an in-training clock switch to `boot_pp`, toggle `PI_CA0` bit 27,
     switch to selector `4` (external clock, `pmu+0x3b4 = 0x3b02`), toggle
     `PI_CA0` bit 27 again, and wait for `PI_STATUS` bit 24.
4. Disable CA mode (`MISC = 0x13008000`), write MR13 (`CA_CMD = 0x1302000d`),
   and switch the clock back to `boot_pp`.

## In-training clock switch (SW-FC + PI PROG/KICK)

Unlike the HW-DFC engine used between training passes ([dfc-engine.md](dfc-engine.md)),
the CA training hardware switches clocks while inside a pass by combining the
PI `PROG`/`KICK` operations with the PMU SW-FC bit:

1. Set `CTL_FREQ[29:28]` to `(sel == 4) ? 0 : sel` (preserving `[31:30] = boot_pp`)
   and set `PI_CFG[3:2] = sel`.
2. Issue `PI_CMD = 0x13000002` (`PROG`) and wait for `PI_STATUS` bit 29.
3. Write `pmu+0x3b4 = clksel[sel]` (`0x3b02` for selector 4 / external clock)
   and `pmu+0xb0 = (sel == 1) ? 0x40600600 : 0x40600400`.
4. Trigger SW-FC (`pmu+0x04 |= (1 << 26)`) and wait for bit 26 to clear.
5. Issue `PI_CMD = 0x13000004` (`KICK`) and wait for `PI_STATUS` bit 30.

## Gaps

- Bit fields of `PI_CA0` and `PI_CA1`.
- The CA-training sequence has not been exercised on silicon; its success
  criteria (CA_STAT bits, resulting CAPHY delays) are not measured.
- Whether LPDDR4 boards on the K1 require CA training.
- Meaning of the `PI_CA0` bit 27 toggling between the two frequency changes.

---

[← Previous: Eye Training](eye-training.md) | [Index](README.md#index) | [Next: The Complete `ddr_init` Sequence →](boot-sequence.md)
