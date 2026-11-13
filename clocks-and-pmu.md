<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: MMIO Map and Register Index](mmio-map.md) | [Index](README.md#index) | [Next: Memory Controller →](memory-controller.md)

# Clocks, PLLs and the PMU

The DDR clock is the responsibility of the DDR image. The mainline SPL does not
program any DDR PLL; it only copies the image to SRAM and calls it. All registers below are
in the APMU, `pmu = 0xd4282800`.

## Registers

| Offset | Name | Function in the DDR boot |
|---|---|---|
| `+0x04` | `PMU_CC_AP` | Frequency-change kicks and handshake bits (bit 31 `AP_RD_ST_CLEAR`, bit 26 SW-FC req, bit 22 `TOP_DDR_FREQ_CHG_REQ`, bit 18 `AP_ALLOW_FREQ_CHG`) |
| `+0x98` | `PMU_AP_IMR` | Interrupt mask: bit 15 = `AP_DCLK_FC_DONE_INT_MSK`, bit 4 = `DCLK_FC_DONE_INT_MSK`, bit 1 = `AP_FC_INTR_MSK` |
| `+0xa0` | `PMU_AP_ISR` | Interrupt status: bit 15 = `AP_DCLK_FC_DONE_INT`, bit 4 = `DCLK_FC_DONE_INT`, bit 1 = `AP_FC_STS` (cleared by writing 0 to the bit, RMW) |
| `+0xb0` | `MC_HW_SLP_TYPE` | PHY clock enable, DCLK bypass, MC/PHY reset, DFC mode, MC table number |
| `+0xe8` | `MCK_CTRL` | MC clock enable, bits `[1:0]` |
| `+0x180` | `DFC_AP` | Runtime AP-triggered DFC control (bit 0 = `DFC_REQ`, bits `[3:1]` = frequency level `0..7`, bit 15 = `DFC_INT_CLR`, bit 14 = `DFC_INT_MSK`) — used by OS/U-Boot runtime DVFS, not by `ddr_init` |
| `+0x188` | `DFC_STATUS` | Runtime DFC status (bit 0 = `DFC_STS` in progress, bits `[3:1]` = current frequency level, bits `[6:4]` = target level, bit 7 = `DFC_CAUSE`, bit 8 = `DFC_INT`) |
| `+0x190`..`+0x1ac` | `DFC_LEVEL_0`..`7` | 8 runtime DFC level descriptors (`pmu + 0x190 + 4*n`): bits `[3:0]` `vol_lv`, `[5:4]` `mc_timing_num`, `[6]` `high_freq`, `[7]` `pll_off`, `[8]` `pll_sel`, `[11:9]` `kvco`, `[12]` `pll_bypass`, `[13]` `dclk_pdiv`, `[15:14]` `pll_div`, `[22:16]` `pll_fbdiv`, `[23]` `band_sel`, `[31:24]` `pll_frcdiv` |
| `+0x398` | `PHY_LDO` | PHY supply mode (LPDDR4 vs LPDDR4X), written by the PHY init |
| `+0x39c` | `PLL1_CTRL1` | PLL1 VCO word |
| `+0x3a0` | `PLL1_CTRL2` | PLL1 analog word |
| `+0x3a4` | (PHY bias) | `[15:8] = 0x0F`, written by the PHY init |
| `+0x3a8` | `PLL2_CTRL1` | PLL2 word |
| `+0x3ac` | `PLL2_CTRL2` | PLL2 analog word |
| `+0x3b4` | `CLKSEL` | DDR clock mux, PLL dividers, PLL enables, PLL lock status |

### Values at entry to `ddr_init`

Power-on-reset values are not observable (the BootROM and SPL run first). On the
reference board, at entry to `ddr_init` (measured, identical on every run,
including after a run with the 2666 MT/s PLL1 words followed by a PMIC reset,
which restores them):

| Register | At entry | After PLL + SW-FC to fp0 |
|---|---|---|
| `+0x04` | `0x00019249` | — |
| `+0xb0` | `0xa0610c00` | `0x40600400` |
| `+0xe8` | `0x40001000` | `0x40001003` |
| `+0x39c` | `0x2a200000` | `0x2a203b00` (2400) / `0x203c5555` (2666) |
| `+0x3a0` | `0x50dd6432` | `0x50dd6432` (2400) / `0x50dd6538` (2666) |
| `+0x3b4` | `0x00073340` (PLL1 already locked) | `0x00033b50` |

## CLKSEL (`pmu+0x3b4`)

The register is written as a whole word. Measured per-fp values:

| fp / sel | Word | Source | Rate |
|---|---|---|---|
| 0 | `0x3b50` | PLL1 / 2 | 1200 MT/s (≈1333 with the 2666 VCO) |
| 1 | `0x3b04` | PLL2 / 2 | 1066 MT/s |
| 2 | `0x3b40` | PLL1 / 1 | 2400 or 2666 MT/s, depending on the VCO |
| 3 | `0x3b00` | PLL2 / 1 | 2133 MT/s (LPDDR4 only) |
| 4 | `0x3b02` | external clock | CA-training path only |

Bitfields inside `pmu+0x3b4`:

| Bits | Name | Function |
|---|---|---|
| `0` | `TOP_EXT_CLK_DIV` | External bypass clock divider |
| `1` | `TOP_EXT_CLK_SEL` | Select external clock (`1` in selector 4, `0x3b02`) |
| `[3:2]` | `TOP_PLL2_DIV` | PLL2 post-divider (`0` = /1 in `0x3b00`, `1` = /2 in `0x3b04`) |
| `[5:4]` | `TOP_PLL1_DIV` | PLL1 post-divider (`0` = /1 in `0x3b40`, `1` = /2 in `0x3b50`) |
| `[7:6]` | `TOP_PLL1_2_SEL` | Clock source select (`0` = PLL2, `1` = PLL1 in bit 6) |
| `8` | `TOP_PLL2_EN` | PLL2 enable |
| `9` | `TOP_PLL1_EN` | PLL1 enable |
| `11` | — | Additional enable required by this SoC |
| `[13:12]` | — | Preserved as `0x3` (`0x3b00` upper control byte) |
| `16` | `PLL2_LOCK` | PLL2 lock status (read-only) |
| `17` | `PLL1_LOCK` | PLL1 lock status (read-only) |

Thus the per-fp words share `0x3b00` (bits 8, 9, 11 plus bits 12–13) and
differ in the low byte, which selects source and divider (`0x50` = PLL1/2,
`0x04` = PLL2/2, `0x40` = PLL1/1, `0x00` = PLL2/1, `0x02` = external). After a
frequency change the register reads back the requested word plus the lock bits
(e.g. `0x00033b04`, `0x00033b40`).

Clearing bits 8/9 makes the lock bits drop (`0x00073340` → `0x00003040`,
measured). There is no separate "update" bit: re-enabling and waiting for
lock is enough.

## PLL1: 2400 vs 2666 MT/s

PLL1 has exactly **two** usable VCO settings on this SoC. There is no general
encoding: 2400/59 ≠ 2666/60, and values such as `0x3D` or neighbours do not
exist. No setting exists for 2700, 2800, 2933, 3000 or 3200 MT/s.

**2666 MT/s** — write both words completely:

```
pmu+0x39c = 0x203c5555
pmu+0x3a0 = 0x50dd6538
```

**2400 MT/s** — read-modify-write of byte `[15:8]` over the value the BootROM
left, **without** writing `+0x3a0`:

```
pmu+0x39c = (pmu+0x39c & 0xFFFF00FF) | (0x3B << 8)
/* +0x3a0 untouched */
```

On the reference board this gives `+0x39c = 0x2a203b00`, `+0x3a0 = 0x50dd6432`.

Pitfalls (*measured*):

- The full word `0x203b5555` (i.e. `0x3B` in bits `[23:16]` of the 2666
  template, with the 2666 `+0x3a0`) is **not** 2400: it locks the **same**
  clock as `0x203c5555`: programming it gives 2666 MT/s, and fp2 training
  then yields the 2666 margins (8–9) instead of the 2400 ones (10).
- Do not zero `+0x39c` / `+0x3a0`; the 2400 recipe depends on the analog
  word left by the BootROM.
- Reading `+0x39c` back only shows what was written; it does **not** prove
  the clock (see "Verifying the real DRAM clock").

Effect on the other frequency points: fp0 is PLL1/2, so with the 2666 VCO fp0
runs at ≈1333 MT/s instead of 1200. PLL2 (fp1 = 1066, fp3 = 2133) is not
affected.

**1333 MT/s as the final rate.** There is no 1333 `data_rate` table entry; the
hardware reaches ≈1333 by using the 2666 VCO and finishing at fp0 (clksel
`0x3b50`, `CTL_FREQ 0x00800400`). Because fp0 is clocked by `PLL1 / 2`, switching
PLL1 from the 2400 VCO to the 2666 VCO raises fp0 from 1200 MT/s to ≈1333 MT/s
with the exact same MC bank 0 and DFC table entry. See
[configurations.md](configurations.md) and
[reference-measurements.md](reference-measurements.md).

### The final rate chooses the VCO, and the VCO sets fp0/fp2

The requested final data rate selects the PLL1 VCO, and therefore determines the
data rates at which the *other* passes run during training (*measured*):

| Requested final rate | PLL1 VCO | fp0 (`PLL1 / 2`) | fp1 (`PLL2 / 2`) | fp2 (`PLL1 / 1`) |
|---|---|---|---|---|
| 1066, 1200, 2400 | 2400 | 1200 MT/s | 1066 MT/s | 2400 MT/s |
| 1333, 2666 | 2666 | ≈1333 MT/s | 1066 MT/s | 2666 MT/s |

fp1 (clocked from PLL2) is always 1066 MT/s. Thus a boot configured to finish at
1066 MT/s still trains fp0 at 1200 MT/s and fp2 at 2400 MT/s; the requested
final rate is only entered by the final DFC step. This is why the same silicon
exhibits two distinct sets of fp0/fp2 training margins depending on the
requested final rate.

## PLL2

Always programmed, also for LPDDR4X (fp1 uses PLL2/2):

```
pmu+0x3a8 = 0x20238000
pmu+0x3ac = 0x50dd632c
```

## Enable and lock

1. In `+0x3b4`, clear bits 8 and 9, then set bits 8, 9 and 11.
2. Wait for `(pmu+0x3b4 & 0x30000) == 0x30000` (PLL1 **and** PLL2 locked).
   Lock is reached well within 1 ms. A PLL that does not lock is a fatal
   condition for the initialisation.
3. There is no other lock register for these PLLs.

With the PLL enable bits left cleared (*measured*), CLKSEL reads
`0x00003040` and the lock bits never assert.

## Frequency-change kicks (`pmu+0x04`)

| Bit | Mask | Name | When |
|---|---|---|---|
| 26 | `0x04000000` | SW-FC | DRAM not initialised yet (first switch to fp0), and the CA-training path |
| 22 | `0x00400000` | HW-DFC | DRAM alive: every frequency change between training passes ([dfc-engine.md](dfc-engine.md)) |

Each kick is its own busy flag: write the bit, then wait for the **same** bit
to read 0. Writing the whole word (`1 << n`) and a read-modify-write both work.
Never mix the two mechanisms (do not wait for bit 26 after a bit-22 kick).

## `pmu+0xb0` (`PMUA_MC_HW_SLP_TYPE`): bitfields and encodings

Full bitfield layout of `pmu+0xb0`:

| Bit(s) | Name | Meaning |
|---|---|---|
| `31` | `TOP_DDRPHY1_EN` | PHY1 clock enable (set at entry, `0xa0610c00`; cleared to `0` during K1 single-channel operation) |
| `30` | `TOP_DDRPHY0_EN` | PHY0 clock enable (`1` in all active encodings) |
| `23` | `TOP_DCLK_BYPASS_FC_REQ` | DCLK bypass frequency-change request |
| `22` | `TOP_DCLK_BYPASS_CLK_EN` | DCLK bypass clock enable (`1` in SW-FC, `0` in HW-DFC) |
| `21` | `TOP_DCLK_BYPASS_RST` | MC/PHY reset, active low (`1` = out of reset, `0` = in reset) |
| `[20:19]` | `TOP_DCLK_BYPASS_SEL` | Bypass clock source (`0` = 312 MHz, `1` = 416 MHz, `2` = 26 MHz) |
| `[18:16]` | `TOP_DCLK_BYPASS_DIV` | Bypass clock divider |
| `10` | `TOP_MC_REG_TABLE_EN` | Enable MC register table (`1` in SW-FC `0x40600400`) |
| `9` | `TOP_FREQ_PLL_CHG_MODE` | Change PLL through the DFC table (`1` in HW-DFC `0x402002xx`, `0` in default SW-FC) |
| `[4:3]` | `TOP_MC_REQ_TABLE_NUM` | Target MC timing bank / frequency point `0..3` |

**SW-FC encoding** (before Device_Init, and the CA path):

| Value | Use |
|---|---|
| `0x40600400` | Default SW-FC setting (bits 30, 22, 21, 10) |
| `0x40600600` | CA-training path only, when the selected source is sel 1 (adds bit 9); never used by the normal boot |

**HW-DFC encoding** (between training passes, bits 30, 21, 9, and `fp << 3` in `[4:3]`):

Values: fp0 `0x40200200`, fp1 `0x40200208`, fp2 `0x40200210`,
fp3 `0x40200218`. `pmu+0xb0` can go directly from the SW-FC value
`0x40600400` to the HW-DFC value; no intermediate step is needed.

## MC/PHY reset (bit 21 of `pmu+0xb0`)

Clear bit 21, wait ≈100 µs, set it again, wait ≈100 µs.

- Resets the MC and the PHY reset path: `PHY_INIT` reads `0x100` again, all
  taps return to their defaults, `PI_STATUS` returns to idle `0x80008000`.
- Does **not** turn off the PLLs: the lock bits in `+0x3b4` stay set. Do not
  repeat the PLL programming or the enable/lock sequence.
- Used only for x8 byte-mode dies, followed by a complete Device_Init
  ([configurations.md](configurations.md)). Also the last-resort recovery
  of a stuck DFC ([dfc-engine.md](dfc-engine.md)).

## MC clock (`pmu+0xe8`)

After the first SW-FC to fp0:

```
pmu+0xe8 = (pmu+0xe8 & ~3) | 3
```

Bits `[1:0] = 3` enable the MC clock. Without this, writes to the MC do not
stick. The AXI port of the DDR (decode at `0xc0000000`) is always present in the
SoC map; there is no separate AXI enable.

## PHY supply and bias

Written by the PHY init ([phy.md](phy.md)), not by the PLL sequence:

- `pmu+0x3a4`: `[15:8] = 0x0F`
- `pmu+0x398`, LPDDR4X: `[11:10] = 3`
- `pmu+0x398`, LPDDR4: `[11:10] = 2` and `[13:12] = 3` (LDO ≈0.88 V)

## From SoC reset to a writable MC

The BootROM/SPL leave the APMU accessible. From there, this sequence makes the
MC writable:

1. Program PLL1 (2400 RMW or 2666 full words) and PLL2.
2. Enable PLL1 + PLL2 and wait for lock (`0x30000` in `+0x3b4`, 1 ms).
3. SW-FC to fp0:
   1. `pmu+0x3b4 = 0x3b50`
   2. `pmu+0xb0 = 0x40600400`
   3. `pmu+0x04` bit 26; wait for bit 26 to clear.

   `dclk` now runs at fp0 (≈1200 MT/s).
4. `pmu+0xe8 [1:0] = 3`.
5. Device_Init ([device-init.md](device-init.md)).

Check: if `ddr+0x44` does not hold a value written to it, step 3 or 4 failed.

## Verifying the real DRAM clock

The PLL block has **no frequency counter**. Register read-back proves nothing
about the actual clock. The method that works is **memory latency**: a chain of
dependent loads (pointer chasing through 64-byte nodes in a random cyclic
permutation, 2²⁰ loads), timed with `mtime` (24 MHz). Details of the probe are
in [test-platform.md](test-platform.md).

With the data cache enabled, CPU at 1599 MHz, random accesses over 32 MiB
(sequential access is hidden by the prefetcher, ≈16 ns; a 16 KiB set fits in L1,
≈1.25 ns):

| fp | Source | PLL1 for 2400 | PLL1 for 2666 |
|---|---|---|---|
| 0 | PLL1 / 2 | 299.65 ns | 278.66 ns |
| 1 | PLL2 / 2 | 331.31 ns | 331.35 ns (control: unchanged) |
| 2 | PLL1 / 1 | 213.35 ns | 198.94 ns |

fp1 does not change, which validates the meter; fp0 and fp2 drop ≈7 % with the
2666 VCO (11 % more clock, the rest is fixed cost). With the full word
`0x203b5555` the latency is identical to the 2666 recipe. Cache-off latencies are listed in
[reference-measurements.md](reference-measurements.md).

## Gaps

- Power-on-reset values of `+0x04`, `+0xb0`, `+0x39c`, `+0x3a0`, `+0x3b4`.
  Only the values at entry to `ddr_init` are measured, and the 2400 recipe
  relies on them; a different BootROM or first stage leaving other values would
  need re-measurement.
- The field layout of the PLL words: only two VCO points are known and the
  2400 point depends on BootROM leftovers. A PLL formula would be needed to
  support other rates (none exist on this SoC anyway: rates above 2666 MT/s are
  out of scope by design).
- The meaning of bit 18 of CLKSEL (set at entry, `0x7` in `[18:16]`; clear
  after the sequence above), of bits 12–13, and of the "extra enable"
  bit 11.
- The other bits of `pmu+0xb0` (bits 31 and 29 are set at entry,
  `0xa0610c00`).
- Whether the interrupt-status bits of `pmu+0xa0` are write-1-to-clear (not
  needed by the boot).

---

[← Previous: MMIO Map and Register Index](mmio-map.md) | [Index](README.md#index) | [Next: Memory Controller →](memory-controller.md)
