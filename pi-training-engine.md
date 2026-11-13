<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: The Hardware DFC Engine](dfc-engine.md) | [Index](README.md#index) | [Next: One Training Pass →](training-pass.md)

# The PI Training Engine

The PI is a training sequencer inside the MC. It performs write leveling, DQS
gate search, read and write eye centring (and CA training) on command, and
writes its results into the PHY: delay taps into the per-byte blocks and margins
into the harvest registers. Its internal algorithm (edge search) is a **black
box**; the firmware triggers it, waits for it and reads its results.

The firmware's job per operation:

1. write the engine parameters (CFG) for the stage — once per stage, not per
   rank;
2. write `PI_CMD`;
3. wait for the operation's done bits in `PI_STATUS` (under a mask, never by
   equality);
4. read the taps from the PHY and, for read/write training, copy the margins
   from the harvest immediately.

## What the engine guarantees

- It centres **each lane** in its own window, not the byte as a whole: at
  2400 MT/s on the reference board all 32 trained RX taps land in 16..19 while
  the measured windows of the byte groups differ by up to 7 codes
  ([eye-training.md](eye-training.md#agreement-with-the-pi)).
- Its Vref choice comes from the harvest minimum (max-min over the 32 DQ and
  all ranks), which can differ by one or two steps from the index that
  maximises the measured delay window. Both choices put the sampling point
  inside the eye.
- The DQS gate **code** it reports (`+0x70`) cannot be overridden, but the
  applied delay lives in `+0x64` (byte 0 / CS0, low 6 bits) and *is*
  software-settable, with a measured window
  ([read-gate.md](read-gate.md#where-the-gate-actually-lives-0x70-is-the-code-0x64-is-the-delay-measured)).
- At 2666 MT/s the write Vref it picks at fp2 is **not stable across boots**
  (code 21 in one run, 35 in another): the margin plateau is 5–6 codes wide and
  the max-min ties, so the tie-break lands elsewhere. Do not treat the value as
  reproducible at that rate; treat the plateau as the result. The same tie-break
  explains the varying Tx Vref code on the 16 GiB Micron board, whose Tx Vref
  curve is a flat plateau at 1200/1066 MT/s.
- Independently of the engine, the same centring result is reproducible by a
  software memory-pattern sweep: choosing the Vref by the common window and then
  centring **each lane** in its own window reproduces the PI's sampling point to
  0–1 code on both ranks and is required at fp2, where a single byte-wide centre
  fails confirmation
  ([eye-training.md](eye-training.md#on-the-16-gib-micron-board-measured-1066-final)).

## Registers

| Address | Name | Role |
|---|---|---|
| `ddr+0x13d0` | PI_CMD | Command port. Also used for the DFI `init_start` command. |
| `ddr+0x13e0` | PI_CFG | `[3:2]` = active fp. Written by Device_Init and the DFC, **never** by the training. |
| `ddr+0x13e8` | PI_WL_CFG | Write-leveling parameters |
| `ddr+0x13ec` | PI_RD_CFG | Read-training parameters (also listed among the per-bank registers) |
| `ddr+0x13f0` | PI_CA0 | CA-training parameters ([ca-training.md](ca-training.md)) |
| `ddr+0x13f4` | PI_CA1 | CA-training parameters |
| `ddr+0x13f8` | PI_WR_CFG | Write-training parameters |
| `ddr+0x13fc` | PI_STATUS | Done bits (sticky), idle bits, PHY-init-done |

## PI_CMD

```
PI_CMD = PI_OP | (cs << 24)        /* cs = 1 .. cs_num */
```

The rank field is **1-based**: `cs = 1` trains CS0, `cs = 2` trains CS1;
`cs = 0` trains nothing. PHY and buffer indices remain 0-based (CS1 registers at
`byte_base + 0x100`, harvest copy buffer `cs - 1`).

**`cs = 3` is not "both ranks": it hangs the sequencer** (*measured*: issuing
`PI_CMD` with `cs = 3` locks the sequencer after the `cs = 2` command and never
completes). Do not use it; always issue `cs = 1` and `cs = 2` separately. The
values 1 and 2 coincide with the rank mask used by the other command ports
([memory-controller.md](memory-controller.md)).

`PI_CMD` reads 0 right after the write. Do not poll it as a busy flag and do not
rewrite it while waiting.

## Operations

| Name | PI_OP | Done bits | Result |
|---|---|---|---|
| WL | `0x10400000` | bit 0 (`0x1`) | PHY `byte + 0x24` |
| RGATE | `0x10100000` | bit 1 (`0x2`) | PHY `byte + 0x70` (and `+0x64` of byte 0 / CS0) |
| RD | `0x10200000` | bits 1 + 2 (`0x6`) | Harvest RX + RX DQ taps |
| WR | `0x10080000` | bits 5 + 6 (`0x60`) | Harvest TX + TX DQ/DM taps |
| CA | `0x10800000` | bit 24 | CA training (unused by the LPDDR4X boot) |
| PROG | `0x13000002` | bit 29 | Applies a frequency slice (CA path frequency change) |
| KICK | `0x13000004` | bit 30 | After a PLL change on the CA path |

Other commands seen on the same port: `0x10000001 | (cs_sel << 24)` = DFI
`init_start` (done = bit 31), `0x10000100 | (cs_sel << 24)` = clear
`init_start`; the DFC flows also use `0x13000010`.

## PI_STATUS semantics

| Property | Detail |
|---|---|
| Idle value | `0x80008000`. Bits 15 and 31 are **always 1** on this SoC. They are not errors; do not wait for them to become 0. |
| Bit 31 | Also "PHY init done" after the DFI `init_start` in Device_Init. |
| Sticky done bits | Done bits accumulate across stages. After a full pass the register reads `0x80008067` — leftovers, not an error. |
| Clear on issue | Writing `PI_CMD` **clears the done bits of that operation**; the sequencer sets them again when it finishes. Measured on WL: leftover `0x80008067` + WL command → immediately `0x80008066` (only bit 0 cleared). The wait is therefore never trivially satisfied. |
| Reset | After Device_Init and after an MC reset the register reads idle `0x80008000`. Writing `PHY_INIT` (512) at the start of a pass does **not** clear leftover done bits. |

Bits each operation clears at issue and waits for:

| Operation | Clears | Waits for |
|---|---|---|
| WL | bit 0 | bit 0 |
| RGATE | bits 1 + 2 | bit 1 (completion sets bits 1 + 2) |
| RD | bits 1 + 2 | bits 1 + 2 |
| WR | bits 5 + 6 | bits 5 + 6 |

All four operations were measured directly on both the 16 GiB Micron and 8 GiB
Samsung boards by reading `PI_STATUS` immediately after writing `PI_CMD` (starting
from `0x80008067`):

- WL clears bit 0 (`0x80008066`) and sets bit 0 at completion;
- RGATE clears **bits 1 and 2** (`0x80008061`) at issue and sets **both** at
  completion (`0x80008067`);
- RD clears bits 1 and 2 (`0x80008061`) and sets both at completion;
- WR clears bits 5 and 6 (`0x80008007`) and sets both at completion.

Thus after RGATE a PASS reads `(PI_STATUS & 6) == 6`, and a timeout reads `0`.
See [read-gate.md](read-gate.md).

Completion test:

```
while ((PI_STATUS & DONE) != DONE)
        ;   /* with a software timeout */
```

Never `PI_STATUS == DONE`: after WR the register reads `0x80008067`, not `0x60`.

Measured per-operation behaviour: the first read after `PI_CMD` shows the
operation's done bit at 0 and a later read shows it at 1, in 72 operations ×
5 repeated training passes.

## Timing

The engine has **no hardware timeout**; software must impose one.

One PI operation completes in ≤17 µs (measured including the polling loop);
the slowest operation ever observed took 34 µs.

Per-stage upper bounds that are safe as timeouts (whole stage, all ranks and
Vref steps, including software polling and harvest copies): WL 10 ms, gate
39 ms, read 114 ms, write 122 ms. A done bit that does not appear within such a
bound is a failure.

## CFG: engine parameters

These are not taps but windows/timeouts of the sequencer. Their bit meanings are
*opaque*; training needs the **written** values, not the reset values.

| Register | Reset | Boot needs | Difference from reset (*opaque* subfields) |
|---|---|---|---|
| PI_WL_CFG (`+0x13e8`) | `0x0a301e44` | `0x0a641e44` | Byte `[23:16]` changes from `0x30` (48) to `0x64` (100); bytes `[30:24] = 0x0a`, `[15:8] = 0x1e`, `[7:0] = 0x44`, bit 31 = 0 |
| PI_RD_CFG (`+0x13ec`) | `0x00000480` | `0x00005070` | Byte `[15:8] = 0x50` (80), byte `[7:0] = 0x70` (112), `[31:16] = 0` |
| PI_WR_CFG (`+0x13f8`) | `0x00000f10` | `0x00001810` | Bits `[15:6]` change from `0x3c` (60) to `0x60` (96); bits `[5:0] = 0x10` (16) |
| PI_CA0 (`+0x13f0`) | `0x055d74ff` (as left by the MC) | not written by the LPDDR4X boot | see [ca-training.md](ca-training.md) |
| PI_CA1 (`+0x13f4`) | 0 | not written | see [ca-training.md](ca-training.md) |

Write each CFG **once**, before the first rank of its stage. Re-writing the same
value later is idempotent; writing it at the start of every pass is the simplest
correct choice.

`PI_RD_CFG` (`+0x13ec`) also appears among the per-bank MC registers, with
Device_Init values `0x480` (fp0, fp1, fp3) and `0x450` (fp2). The training
overwrites it with `0x5070` in the active bank at the start of the read stage.

## Results

- Write leveling: `byte + 0x24` per byte and rank. No pass/fail code of its
  own; a failure appears as bit 1 of `phy+0x18000`.
- Gate: `byte + 0x70`.
- Read: RX DQ taps at `byte + 0x30 + 4*lane` (mask 31) and harvest RX.
- Write: TX DQ taps at `byte + 0x00 + 4*lane`, TX DM at `byte + 0x20`
  (mask 63) and harvest TX.
- The PI writes all four bytes (sub-PHY A and B).
- The harvest is a latch of the **last** operation; copy it right after each
  wait, before the next rank's operation.

## What the PI does not do

- It does not choose the frequency: it works at the one that is running.
- It does not program MR14 (the firmware does, per step,
  [mode-registers.md](mode-registers.md)).
- It does not select the Vref: it measures one Vref per operation; the firmware
  sweeps and chooses ([eye-training.md](eye-training.md)).

## Gaps

- The internal sequencer algorithm (edge search, step sizes, pass/fail
  criteria) — would need RTL or documentation. It is **not needed** for the
  Rx/Tx delay and Vref stages: those can be replaced by a memory-pattern sweep
  ([eye-training.md](eye-training.md#independent-verification-of-the-eye-memory-pattern-sweep)).
  Write leveling also has a software replacement: sweeping the 5-bit `+0x24`
  word with the same pattern test reproduces the PI's centring
  ([write-leveling.md](write-leveling.md#independent-measurement-of-the-wl-window-memory-pattern-sweep)).
  The DQS gate also has a software control — not the code at `+0x70`, which is
  a result, but the applied delay in `+0x64`
  ([read-gate.md](read-gate.md#where-the-gate-actually-lives-0x70-is-the-code-0x64-is-the-delay-measured)).
- Bit meanings of the CFG parameters (window vs timeout) and of PI_CA0/CA1.
- Whether the CFG registers other than `+0x13ec` are banked per fp. Writing
  them at the start of every pass sidesteps the question.
- A real RGATE timeout after an actual command has never been observed on the
  tested boards (the operation always completed within 34 µs).

---

[← Previous: The Hardware DFC Engine](dfc-engine.md) | [Index](README.md#index) | [Next: One Training Pass →](training-pass.md)
