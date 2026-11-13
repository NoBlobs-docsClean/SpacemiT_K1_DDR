<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Clocks, PLLs and the PMU](clocks-and-pmu.md) | [Index](README.md#index) | [Next: Device_Init →](device-init.md)

# Memory Controller (MC)

Base: `ddr = 0xc0000000`. The MC does **not** compute delays. For the boot it
is a set of command ports plus configuration storage:

- command ports that sequence the DRAM (init, ZQ, self-refresh, FSP, mode
  registers, CA on/off);
- four **timing banks**, one per frequency point;
- the address map (CS sizes, CS1 start, geometry);
- the **PI** training sequencer ([pi-training-engine.md](pi-training-engine.md));
- the **DFC table** consumed by the hardware frequency-change engine
  ([dfc-engine.md](dfc-engine.md)).

## Command word format

The command registers share one layout (*inferred* from every command used by
the boot; all observed commands fit it):

```
 31   28 27 26 25 24 23                                 0
+-------+-----+-------+------------------------------------+
| 0001  |  0  |CS mask|            command payload         |
+-------+-----+-------+------------------------------------+
```

- `[31:28] = 1` on every command.
- `[25:24]` = rank mask: bit 0 = CS0, bit 1 = CS1. `cs_sel = 1` with one
  rank, `3` with two; a single rank `c` (0-based) is `1 << c`, i.e. `c + 1`.
- The payload depends on the port.

So `0x13…` addresses both ranks, `0x11…` CS0 only and `0x12…` CS1 only.

## USER_0 (`ddr+0x20`): DRAM sequencing

| Command | Word (both ranks) | Effect |
|---|---|---|
| DRAM init | `0x10000001 \| (cs_sel << 24)` | JEDEC power-up sequence (RESET_n, CKE, tINIT1..5) performed by the MC |
| Self-refresh enter | `0x10000040 \| (mask << 24)` → `0x13000040` | Puts the selected ranks in SR |
| Self-refresh exit | `0x10000080 \| (mask << 24)` → `0x13000080` | Takes them out |
| ZQCAL START, CS0 | `0x11002000` | MPC ZQCAL start |
| ZQCAL LATCH, CS0 | `0x11001000` | MPC ZQCAL latch |
| ZQCAL START / LATCH, CS1 | `0x12002000` / `0x12001000` | |
| FSP-WR request | `0x13000008` | Toggles the FSP being written, driven from the PMU side; only used inside the DFC flow |
| FSP-OP request | `0x13000004` | Switches the operating FSP; only used inside the DFC flow |
| (DFC flow) | `0x13000010` | Used by the DFC flows; meaning *opaque* |

ZQCAL is an MPC that goes through USER_0, **not** through CA_CMD. No
completion bit exists for ZQCAL or for the FSP requests; see
[device-init.md](device-init.md) for the delays.

Do not issue the FSP requests by hand after a DFC: `0x13000008` after a
frequency change flips FSP-WR away from the set that mode-register reads
observe.

## CA_CMD / USER_1 (`ddr+0x24`): mode registers

```
CA_CMD = 0x10000000 | (cs_mask << 24) | (ch << 18) | (op << 16) | mr
         op = 1: MRR (read)    op = 2: MRW (write)
         ch = JEDEC channel 0/1 (read only)
         mr = mode-register number, [7:0]
```

- **MRR**: `0x10010000 | ((cs + 1) << 24) | (ch << 18) | mr`, one rank and
  one channel at a time; the result appears in `MR_DATA` (`ddr+0x370`),
  bit 31 = valid, `[7:0]` = value. Budget 10 µs.
- **MRW**: `0x10020000 | (cs_sel << 24) | mr` (`0x13020000 | mr` with two
  ranks). The **data is not in the command**: the MC serialises it from the
  bank of the active frequency point. MR14 comes from `ddr+0x30c[21:16]`, MR12
  from `ddr+0x30c[31:24]`, the others from the rest of the bank.

Full protocol in [mode-registers.md](mode-registers.md).

## MISC (`ddr+0x28`): CA port on/off

Not a static configuration register but a command port:

| Word | Effect |
|---|---|
| `0x13010000` | CA on |
| `0x13008000` | CA off |

The LPDDR4X training does not write it. The DFC flows write both values (DRAM
out of self-refresh). After Device_Init it reads 0 on the reference board.

## STATUS (`ddr+0x08`)

Per-rank status bits, in pairs CS0/CS1:

| Bit | Rank | Meaning |
|---|---|---|
| 0 / 4 | CS0 / CS1 | DRAM init done |
| 1 / 5 | CS0 / CS1 | CKE / active (*inferred*) |
| 2 / 6 | CS0 / CS1 | In self-refresh |

Typical values: `0x33` after Device_Init (ranks up, not in SR), `0x77` in
training SR, `0x11` after the training SR exit. The bits are **not** W1C; they
change only with USER_0 commands and with the MC reset. Full semantics in
[self-refresh.md](self-refresh.md).

## CTL0 (`ddr+0x44`)

Global control word. Device_Init writes `0x00040300`. The DFC flows write other
variants (`0x00040303`, `0x00040380`, `0x00040b43`, `0x00040b00`), and the DFC
snapshot header uses `CTL0 | (1 << 2) | (1 << 12)`. Bit meanings are *opaque*.
`ddr+0x44` holding a written value is the simplest check that the MC clock is
running.

## CTL_FREQ (`ddr+0x304`): frequency point and bank select

```
[31:30]  fp: active frequency point AND the timing bank that register
         accesses go to
[29:28]  sel: frequency select (equal to fp in the normal boot)
[27:0]   0x0800400 on the reference board (opaque, preserve)
```

| fp | CTL_FREQ |
|---|---|
| 0 | `0x00800400` |
| 1 | `0x50800400` |
| 2 | `0xa0800400` |
| 3 | `0xf0800400` |

Software writes `CTL_FREQ` only while filling the banks during Device_Init
(and while building the DFC snapshot). In the normal boot the HW-DFC engine
updates it; the training never writes it. The CA-training path is the exception:
there software writes `[29:28]` itself ([ca-training.md](ca-training.md)).

## PI_CFG (`ddr+0x13e0`)

`[3:2]` = active fp (observed values 0, 4, 8 for fp0, fp1, fp2). Written by
Device_Init (`|= fp << 2` with fp = 0) and maintained by the DFC. The training
must **not** write it: doing so races with the DFC.

## Timing banks

The MC keeps one copy of every frequency-dependent register per frequency
point. Writing `CTL_FREQ[31:30] = fp` selects the copy that subsequent register
reads and writes access. Device_Init fills all four banks (including fp3 on
LPDDR4X). Frequency changes only **select** a bank; they never rewrite it.

The list of banked and global registers follows from the DFC snapshot layout
([dfc-engine.md](dfc-engine.md)):

- **Per bank (24):** `0x300, 0x30c, 0x310, 0x314, 0x38c, 0x390, 0x394, 0x398,
  0x39c, 0x3a0, 0x3a4, 0x3a8, 0x3ac, 0x3b0, 0x3b4, 0x3b8, 0x3bc, 0x3c0,
  0x3c4, 0x400, 0x3d8, 0x34c, 0x13e4, 0x13ec` (plus `CTL_FREQ` itself).
- **Global (22):** `0x048, 0x054, 0x058, 0x060, 0x064, 0x148, 0x14c, 0x200,
  0x204, 0x208, 0x20c, 0x220, 0x224, 0x2c4, 0x2c0, 0x380, 0x384, 0x388,
  0x080, 0xa00, 0xac0, 0xacc`.

Values are in [device-init.md](device-init.md).

## Other registers

| Offset | Role |
|---|---|
| `0x70` / `0x74` / `0x78` | DFC table port: INDEX / DATA / ADDR. Filled through the `0xf0000000` window ([dfc-engine.md](dfc-engine.md)). |
| `0x148` | Written `0x80ac0000` before every HW-DFC. Not touched by Device_Init. *Opaque.* |
| `0x200`–`0x224` | Address map ([address-map.md](address-map.md)) |
| `0x370` | MR_DATA ([mode-registers.md](mode-registers.md)) |
| `0x13d0` | PI_CMD; also the DFI `init_start` command port ([device-init.md](device-init.md), [pi-training-engine.md](pi-training-engine.md)) |
| `0x13e8`–`0x13fc` | PI configuration and status ([pi-training-engine.md](pi-training-engine.md)) |

## Gaps

- The command-word layout is inferred from all observed commands; other
  payload bits of USER_0, CA_CMD and MISC are unknown.
- Meaning of USER_0 payload `0x10` (used by the DFC flows).
- Meaning of the CTL0 bits, of `CTL_FREQ[27:0]`, of `ddr+0x148` and of the
  globals marked *opaque*.
- STATUS bits 1/5 are only inferred to be CKE/active; bits 3 and 7 and above
  are not characterised.
- Reset values of the MC registers are not established; write complete words.

---

[← Previous: Clocks, PLLs and the PMU](clocks-and-pmu.md) | [Index](README.md#index) | [Next: Device_Init →](device-init.md)
