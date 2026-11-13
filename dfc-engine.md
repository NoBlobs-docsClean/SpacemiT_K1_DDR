<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Address Map and Geometry](address-map.md) | [Index](README.md#index) | [Next: The PI Training Engine →](pi-training-engine.md)

# The Hardware DFC Engine

The K1 has two ways of changing the DDR clock:

| Mechanism | Kick | Used for |
|---|---|---|
| **SW-FC** | `pmu+0x04` bit 26 | The first switch to fp0, before the DRAM is initialised ([clocks-and-pmu.md](clocks-and-pmu.md)); and the CA-training path ([ca-training.md](ca-training.md)) |
| **HW-DFC** | `pmu+0x04` bit 22 | Every frequency change with live DRAM: between training passes and the final switch |

This file describes the HW-DFC. It is a table-driven engine: when kicked it
executes command flows stored in a table inside the MC, parks the DRAM in
self-refresh, hands over to the PMU to switch the clock, restores the MC from a
register snapshot, reprograms the dies' mode registers for the new frequency and
resumes. The firmware only loads the table (once), kicks and waits.

## Who parks the DRAM

The **engine**, not the firmware. Its flows contain the USER_0, MISC, DFI and MR
commands. The firmware:

1. makes sure the previous training pass has left self-refresh;
2. kicks the HW-DFC in the PMU;
3. waits for the busy bit.

Do not issue `0x13000040` (SR enter) by hand around a HW-DFC.

## Kick recipe

Preconditions:

- the four MC timing banks are programmed (Device_Init);
- the address map has been adjusted;
- the table has been loaded completely (all regions below), **after** the map
  adjustment;
- the DRAM is **not** in training self-refresh: `(ddr+0x08 & 0x44) == 0`
  (2 ranks; typical STATUS `0x33` or `0x11`).

```
pmu+0x98  |= 0x10                  /* mask the DCLK frequency-change IRQ */
ddr+0x148  = 0x80ac0000            /* before EVERY HW-DFC, before the PLL change */
pmu+0x3b4  = clksel[fp]            /* 0x3b50 / 0x3b04 / 0x3b40 / 0x3b00 */
pmu+0xb0   = 0x40200200 | (fp << 3)/* table / bank number in [4:3] */
pmu+0x04   = 1 << 22               /* kick (whole word or RMW, both work) */
wait (pmu+0x04 & 0x00400000) == 0  /* budget 10 ms; measured 25–30 µs */
```

From the SW-FC value `0x40600400`, `pmu+0xb0` goes directly to the HW-DFC
value; no intermediate step (bit 22 / bit 10) is needed. Clearing `pmu+0xa0`
before the first DFC is not needed; if desired, write the bits 1, 4 and 15
(W1C *inferred*).

The firmware does **not** write `CTL_FREQ` or `PI_CFG` for a HW-DFC: the engine
sets `CTL_FREQ[31:30] = [29:28] = fp` and `PI_CFG[3:2] = fp` itself.

### Verifying success

- Bit 22 of `pmu+0x04` back to 0.
- `CTL_FREQ[31:30] == CTL_FREQ[29:28] == fp`, rest `0x00800400`
  (fp1 → `0x50800400`, fp2 → `0xa0800400`).
- `PI_CFG[3:2] == fp`.
- `pmu+0x3b4` shows the requested clksel with the lock bits
  (e.g. `0x00033b04`, `0x00033b40`).
- STATUS `0x33` (ranks up, out of SR).
- After the DFC, one **dummy MRR** before any meaningful mode-register read
  (the first MRR after the first DFC to fp2 returns 0,
  [mode-registers.md](mode-registers.md)). After the next write
  training, MR14 must equal `CTL_TXVREF[21:16]`.

`PI_STATUS = 0x80008067` (leftover done bits of the previous pass) does **not**
prevent the DFC; it reads the same before each kick and after the final DFC.
`PHY_INIT` reading `0x200` (left by the previous pass) is also expected; do not
reset it to `0x100`.

## The table window: `0xF0000000`

The table port (`+0x70/+0x74/+0x78`) and the PHY marker registers
(`+0x40000 + 0x10100…`) are filled through the window `T = 0xF0000000`,
**not** through `ddr`.

On the reference board the window **aliases the MC**:
`read(0xF0000000 + 0x44) == read(ddr + 0x44) == 0x00040300`. Writing through
`ddr+0x70` would therefore also work there, but filling through the window is
correct whether or not it aliases. A quick check:

```
alias = (read(0xF0000000 + 0x44) == read(ddr + 0x44))
```

If it does not alias (0 or garbage at `0xF0000000+0x44`), the table RAM exists
only behind the window: values for the snapshot are still **read from `ddr`**,
but DATA/ADDR/INDEX and the markers must be **written to `T`**.

## Entry format

One entry:

```
T+0x74 = DATA
T+0x78 = ADDR
T+0x70 = INDEX       /* writing the index commits the entry */
```

ADDR fields:

| Bits | Name | Effect |
|---|---|---|
| `[12:0]` | offset | Target MC register (`0x13fc`, PI_STATUS, fits) |
| 13 | POLL | 1 = the engine reads the offset until DATA is satisfied; 0 = it writes DATA |
| 16 | REQ_PMU | Handshake: the PMU changes the PLL only after this entry |
| 17 | EOP | Last entry of this flow |

So `0x00002008` = poll STATUS (`+0x08`), `0x000033fc` = poll PI_STATUS
(`+0x13fc`), `0x00010044` = write CTL0 + REQ_PMU, `0x00020044` = write CTL0 +
EOP. There are **no** MMIO registers at `ddr+0x2008` or `ddr+0x33fc`; these are
only ADDR encodings.

POLL semantics are the engine's business; the firmware does not reproduce them.
Two consecutive polls on the same offset describe a transition (e.g. PI_STATUS
`0x8000` then `0`). DATA = 2 on STATUS is bit 1 (not the SR mask `0x44`).
DATA = 0 on PI_STATUS does **not** require the whole register to read 0: the
boot works with the leftover `0x80008067`. Whether POLL compares under a mask or
waits for a pulse of bit 15 is not established.

## Table regions

All of them are mandatory before the first kick:

1. **Entry flow** at indices `0x00`–`0x1b` (plus a copy of the last entry at
   `0x12e`).
2. **Exit / idle flow** at `0x180`–`0x18e`.
3. **Snapshot** of the MC registers at `0x200`–`0x28a`.
4. **PHY analog markers**.

### Entry flow (`0x00`–`0x1b`)

`(DATA, ADDR, INDEX)`. ADDR `0x24` = CA_CMD, `0x20` = USER_0, `0x28` = MISC,
`0x44` = CTL0, `0x13d0` = PI/DFI command. Annotations are *inferred* from the
command formats; the values are *opaque* and must be written as listed.

```
0x00040303  0x00000044  0x00    CTL0
0x13000008  0x00000020  0x01    USER_0: FSP-WR request
0x13010000  0x00000028  0x02    MISC: CA on
0x1302000d  0x00000024  0x03    MRW MR13
0x13020001  0x00000024  0x04    MRW MR1
0x13020002  0x00000024  0x05    MRW MR2
0x13020003  0x00000024  0x06    MRW MR3
0x1302000b  0x00000024  0x07    MRW MR11
0x1302000c  0x00000024  0x08    MRW MR12
0x1302000e  0x00000024  0x09    MRW MR14   (do not change this entry)
0x13020016  0x00000024  0x0a    MRW MR22
0x13008000  0x00000028  0x0b    MISC: CA off
0x1302000d  0x00000024  0x0c    MRW MR13 again (FSP-OP)
0x13000010  0x00000020  0x0d    USER_0 (opaque command 0x10)
0x00000002  0x00002008  0x0e    POLL STATUS
0x00000002  0x00002008  0x0f    POLL STATUS
0x13000001  0x000013d0  0x10    DFI init_start
0x00008000  0x000033fc  0x11    POLL PI_STATUS
0x00000000  0x000033fc  0x12    POLL PI_STATUS
0x00040303  0x00010044  0x13    CTL0 + REQ_PMU   <- PLL switch happens here
0x10000100  0x000013d0  0x14    DFI init_start clear
0x00008000  0x000033fc  0x15    POLL PI_STATUS
0x00008000  0x000033fc  0x16    POLL PI_STATUS
0x13000004  0x00000020  0x17    USER_0: FSP-OP request
0x1302000d  0x00000024  0x18    MRW MR13
            (16 GiB: insert 0x13020095 / 0x24 here at 0x19 and shift the rest)
0x00000002  0x00002008  0x19    POLL STATUS
0x00000000  0x00002008  0x1a    POLL STATUS
0x00040380  0x00020044  0x1b    CTL0 + EOP
0x00040380  0x00020044  0x12e   copy of the EOP entry
```

### Exit / idle flow (`0x180`–`0x18e`)

```
0x00040b43  0x00000044  0x180   CTL0
0x13000010  0x00000020  0x181   USER_0 (opaque command 0x10)
0x00000002  0x00002008  0x182   POLL STATUS
0x00000002  0x00002008  0x183   POLL STATUS
0x13000001  0x000013d0  0x184   DFI init_start
0x00008000  0x000033fc  0x185   POLL PI_STATUS
0x00000000  0x000033fc  0x186   POLL PI_STATUS
0x00040b43  0x00010044  0x187   CTL0 + REQ_PMU
0x10000100  0x000013d0  0x188   DFI init_start clear
0x00008000  0x000033fc  0x189   POLL PI_STATUS
0x00008000  0x000033fc  0x18a   POLL PI_STATUS
0x1302000d  0x00000024  0x18b   MRW MR13
            (16 GiB: MR21 again here)
0x00000002  0x00002008  0x18c   POLL STATUS
0x00000000  0x00002008  0x18d   POLL STATUS
0x00040b00  0x00020044  0x18e   CTL0 + EOP
```

### One rank

The listed flows address both ranks (`0x13` in the command byte). With one rank
the USER_0, CA_CMD and PI/DFI entries get `0x11` in that byte instead
(*inferred* from the command format; not measured without a one-rank board).

### Snapshot (`0x200`–`0x28a`)

The snapshot lets the engine restore the MC for the target frequency. Values are
**read from `ddr`** (the live MC); entries are **written to `T`**. 22 global
registers, then one block of 25 entries per fp (`CTL_FREQ` + 24 banked
registers).

Global offsets (once):

```
0x048, 0x054, 0x058, 0x060, 0x064, 0x148, 0x14c,
0x200, 0x204, 0x208, 0x20c, 0x220, 0x224,
0x2c4, 0x2c0, 0x380, 0x384, 0x388,
0x080, 0xa00, 0xac0, 0xacc
```

Banked offsets (24, per fp):

```
0x300, 0x30c, 0x310, 0x314,
0x38c, 0x390, 0x394, 0x398, 0x39c, 0x3a0, 0x3a4,
0x3a8, 0x3ac, 0x3b0, 0x3b4, 0x3b8, 0x3bc, 0x3c0,
0x3c4, 0x400, 0x3d8, 0x34c, 0x13e4, 0x13ec
```

Layout:

| Index | DATA | ADDR |
|---|---|---|
| `0x200` | `read(ddr+0x44) \| (1 << 2) \| (1 << 12)` | `0x10044` (CTL0 + REQ_PMU) |
| `0x201`–`0x216` | `read(ddr+off)` for the 22 globals | `off` (no flags) |
| `0x217`–`0x22f` | fp0 block | |
| `0x230`–`0x248` | fp1 block | |
| `0x249`–`0x261` | fp2 block | |
| `0x262`–`0x27a` | fp3 block | |
| `0x27b`–`0x289` | fixed tail (below) | |
| `0x28a` | `read(ddr+0x44)` as read at the start (without the extra bits) | `0x20044` (CTL0 + EOP) |

Each fp block: select the bank in the **live** MC by writing `CTL_FREQ[31:28]`
= 0, 5, 0xA, 0xF (as in Device_Init), then write one `+0x304` entry followed by
the 24 banked registers read from that bank. The `+0x304` value **stored in the
table** has `[29:28]` cleared (`DATA &= ~0x30000000` after reading the live
value):

| fp | Live `[31:28]` | Stored `[31:28]` | Block index | Stored `CTL_FREQ` (typical) |
|---|---|---|---|---|
| 0 | 0 | 0 | `0x217` | `0x00800400` |
| 1 | 5 | 4 | `0x230` | `0x40800400` |
| 2 | 0xA | 8 | `0x249` | `0x80800400` |
| 3 | 0xF | 0xC | `0x262` | `0xC0800400` |

Fixed tail (*opaque*):

| Index | DATA | ADDR | (*inferred* role) |
|---|---|---|---|
| `0x27b` | `0x00020200` | `0x13e0` | PI_CFG |
| `0x27c` | `0x13000010` | `0x13d0` | PI/DFI command |
| `0x27d` | `0x00010000` | `0x33fc` | POLL PI_STATUS |
| `0x27e` | `0x00010000` | `0x33fc` | POLL PI_STATUS |
| `0x27f` | `0x13000008` | `0x0020` | USER_0: FSP-WR |
| `0x280` | `0x13000004` | `0x0020` | USER_0: FSP-OP |
| `0x281` | `0x13020000` | `0x0028` | MISC |
| `0x282` | `0x13000001` | `0x13d0` | DFI init_start |
| `0x283` | `0x00008000` | `0x33fc` | POLL PI_STATUS |
| `0x284` | `0x00008000` | `0x33fc` | POLL PI_STATUS |
| `0x285` | `0x10000100` | `0x13d0` | DFI init_start clear |
| `0x286` | `0x00008000` | `0x33fc` | POLL PI_STATUS |
| `0x287` | `0x00008000` | `0x33fc` | POLL PI_STATUS |
| `0x288` | `0x1302000d` | `0x0024` | MRW MR13 |
| `0x289` | `0x13020003` | `0x0024` | MRW MR3 |

After the snapshot, restore `CTL_FREQ` (`ddr+0x304`) to the fp0 value.

### PHY analog markers

Written through the window as well (`T + 0x40000 + …`, i.e. the PHY OTHER block
seen through `T`), in this order:

```
T+0x40000+0x10104 = 0x00001100
T+0x40000+0x10108 = 0x00010000
T+0x40000+0x10100 = 0x00000020
T+0x40000+0x10104 = 0x000000ff
T+0x40000+0x10108 = 0x0001001c
T+0x40000+0x10100 = 0x00000021
T+0x40000+0x10104 = 0x00000000
T+0x40000+0x10108 = 0x0005001c
T+0x40000+0x10100 = 0x00000022
```

*Inferred:* these have the same shape as the MC table port (DATA at `+0x104`,
ADDR at `+0x108`, then INDEX at `+0x100` committing), i.e. three entries
(indices `0x20`–`0x22`) of a PHY-side table; bit 16 of the ADDR words matches
REQ_PMU. Their meaning is *opaque*.

### Pre-kick checklist

- `(ddr+0x08 & 0x44) == 0`;
- the alias check of the window has been done (or the window is used anyway);
- the last snapshot entry is at index **`0x28a`**, not `0x27a`;
- ADDR of `0x200` is `0x10044` and ADDR of `0x28a` is `0x20044`;
- the stored `CTL_FREQ` at `0x230` / `0x249` / `0x262` has `[29:28] == 0`;
- `CTL_FREQ` is still fp0 (`0x00800400`) before the first DFC.

## Final DFC

Identical to `DFC(fp_final)`. For 2400 and 2666, `fp_final = 2`: the same
operation as the training DFC(2), issued again after the last pass (idempotent).
For 1066, `fp_final = 1`. For 1200 and 1333, `fp_final = 0` (`PLL1 / 2`, so 1200
uses the 2400 VCO and 1333 uses the 2666 VCO with the exact same DFC table
entry). The `data_rate` → fp mapping is in [boot-interface.md](boot-interface.md).
A LPDDR4X boot asked for 2133 trains fp0/fp1/fp2 and the final DFC goes from fp2
**back to fp0** (`CTL_FREQ 0x00800400`, clksel `0x3b50`); this path works and
takes 29–30 µs.

## Failure modes (measured)

| Situation | What the hardware shows |
|---|---|
| Incomplete table (snapshot ending at `0x27a`, no header/EOP flags, live `CTL_FREQ` values stored, no tail, no markers) | Bit 22 still set 110 ms after the kick; STATUS `0x77` (the engine entered SR); `CTL_FREQ = 0x00800400` and `PI_CFG = 0` unchanged; the DRAM stays in SR and the SPL hangs at its `0xaa55aa55` test. Kick form (word vs RMW) is not the cause. |
| No table loaded at all | `pmu+0x04 = 0x00400000` (bit 22 stuck), STATUS `0x77`, `CTL_FREQ = 0x00800400`; the busy bit never clears. |
| STATUS `0x77` at the kick (previous pass did not leave SR) | The engine starts (enters/keeps SR) but never changes `CTL_FREQ`: the same hang. Fix the SR exit; the DFC is not the place to resolve it. |

A bit 22 still set after 10 ms is a stuck engine, not "still running".

### Recovering from a stuck engine

Bit 22 stays set in hardware until the flow reaches REQ_PMU and EOP. Writing 0 to
`pmu+0x04` does **not** release it.

1. `USER_0 = 0x13000080` (SR exit, 2 ranks) and wait for
   `(STATUS & 0x44) == 0`. The DRAM becomes accessible again; bit 22 may stay
   set.
2. If bit 22 is still set: MC reset (`pmu+0xb0` bit 21, ≈100 µs low and high),
   Device_Init again, reload the table through `0xF0000000`, train fp0 again
   if the taps were lost. Never kick another DFC on top of a stuck bit 22.

### Useful diagnostics

Right before the kick and on timeout: STATUS, CTL0, MISC, `ddr+0x148`,
`CTL_FREQ`, `PI_CFG`, `PI_STATUS`, `phy+0x10028`,
`pmu+0x04/0xa0/0xb0/0x3b4`, and `read(0xF0000000+0x44)` vs `ddr+0x44`.

## Gaps

- POLL semantics: comparison under a mask vs waiting for a pulse of bit 15 of
  PI_STATUS. Full-register equality is excluded.
- Meaning of `ddr+0x148 = 0x80ac0000`, of the CTL0 variants in the flows, of
  USER_0 command `0x10` and of the tail entries (`PI_CFG = 0x00020200`,
  `MISC = 0x13020000`, PI_STATUS polls on bit 16). All are required exactly as
  listed.
- Whether the three PHY marker writes really are entries of a PHY-side table
  (they only share the DATA/ADDR/INDEX shape), and what they do.
- Whether the `0xF0000000` window is a separate RAM on other boards or silicon
  revisions. Always filling the table through the window is safe either way.
- The table size limits (highest usable index), the handling of unused indices
  and whether gaps between regions matter.
- The one-rank variant of the flows is not measured on hardware; the 16 GiB
  variant (with MR21 inserted) is confirmed on the 16 GiB Micron board.

---

[← Previous: Address Map and Geometry](address-map.md) | [Index](README.md#index) | [Next: The PI Training Engine →](pi-training-engine.md)
