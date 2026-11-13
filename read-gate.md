<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Write Leveling](write-leveling.md) | [Index](README.md#index) | [Next: Eye Training →](eye-training.md)

# DQS Read Gate

## Goal

Find, per byte and per rank, the window during which the PHY opens the DQS
gate on reads. The result is the gate code at `byte + 0x70`
([phy.md](phy.md)).

## Recipe

```
for cs in 1 .. cs_num:
    PI_CMD = 0x10100000 | (cs << 24)
    wait (PI_STATUS & 0x2) == 0x2            /* software timeout -> fail */
    st = PI_STATUS
    if (st & 6) == 6:  PASS                  /* bits 1+2 set on completion */
    else:              fail                  /* timeout leaves (st & 6) == 0 */
    if PHY_STATUS & (1 << 4): fail           /* defensive check on gate error bit */
```

There is no separate CFG register for the gate stage.

## Issue and completion status (measured)

Measured on both the 16 GiB Micron and 8 GiB Samsung boards by reading
`PI_STATUS` immediately after issuing each command (starting from `0x80008067`):

| Command | `PI_STATUS` right after issue | `PI_STATUS` after completion |
|---|---|---|
| WL | `0x80008066` (bit 0 cleared) | `0x80008067` |
| RGATE | `0x80008061` (**bits 1 and 2 cleared**) | `0x80008067` (both set) |
| RD | `0x80008061` (bits 1 and 2 cleared) | `0x80008067` |
| WR | `0x80008007` (bits 5 and 6 cleared) | `0x80008067` |

Issuing `RGATE` clears **both bits 1 and 2**, and completion sets **both**.
Polling waits on bit 1, and bit 2 is set alongside it on success:

- a PASS leaves `(st & 6) == 6`;
- a timeout leaves `(st & 6) == 0` (both bits were cleared at issue and neither
  asserted); any value other than `(st & 6) == 6` is a failure;
- checking `phy+0x18000` bit 4 (read-gate error) remains a useful defensive
  safeguard.

The wait itself must have a software timeout. Measured: when the done bit is
awaited without issuing the RGATE command, bit 1 never sets and `PI_STATUS`
stays at `0x80008001` (only the WL done bit from the previous stage). The done
bit is only produced by an issued operation. A timeout of an RGATE that was
actually issued has never been observed: on this board the operation always
completes in microseconds.

Ignoring a gate timeout leaves the gate in the wrong place.

## Gate code acceptance

```
GATE = (1 << 16) | (g << 8) | t
```

| When | Acceptance |
|---|---|
| Before the PI | `0x00000a00`: untrained default, not a valid result |
| Right after RGATE | bit 16 = 1 and the two low bytes equal (`g == t`) |
| End of the pass | bit 16 = 1 and both bytes in the trained range; `t` may differ from `g` by up to 2 (read training adjusts it, e.g. `0x00012625`, `0x00012423`) |

The PI writes the gate of all four bytes (sub-PHY A and B). Do not copy A to B.

## Typical values (reference board)

| Pass | Gate (byte value `g`) |
|---|---|
| First fp0 pass (cold PHY), 1200 MT/s | ≈105–118 taps (111–118 measured at a true 1200) |
| First fp0 pass with the 2666 VCO (fp0 ≈1333) | ≈104–110 |
| Second fp0 pass, after the x8 re-initialisation | ≈32–40 (32 = `0x20` is accepted) |
| fp1, 1066 MT/s | ≈32–38 |
| fp2, 2400 MT/s | ≈34–40 |

The cold first-pass value is not a criterion: it reflects the x16 timings
programmed on x8 dies before the byte-mode re-init. Do not compare the first
pass with the second or with fp2.

Measured directly on both boards: running a second fp0 training pass *without*
performing the x8 MC reset and `Device_Init` keeps the cold gate code (`110–113`
on the 16 GiB Micron board, `113–115` on the 8 GiB Samsung board), whereas the
pass after the x8 `Device_Init` and MC address-map adjustment drops to `29–33`
(Micron) and `33–35` (Samsung). Thus the drop comes from the x8 PHY/MC
re-initialisation, not from "warming up" the PHY by training twice; the Vref and
minimum margins are identical in all three passes.

On the 16 GiB Micron board the gate codes match the reference board's pattern:
cold fp0 `110–114` (`109–114` across 15 runs, or `101–105` with the 2666 VCO),
fp0 after the x8 re-init `29–33`, fp1 `29–34`, and fp2 `32–36`, all reading
`0x0001ggtg` with equal bytes immediately after RGATE.

## Where the gate actually lives: `+0x70` is the code, `+0x64` is the delay (measured)

The `+0x70` word cannot be used to move the gate:

- only the low byte (`t`) accepts a write; the high byte keeps the value the PI
  left. Writing `0x00010000` reads back `0x00017300` / `0x00017100`;
- writing `(1 << 16) | (d << 8) | d` into `+0x70` and `+0x1070` for 64 values
  of `d` from 0 to 252, and separately forcing `0x00010000` into all eight
  byte/rank gate words (`+0x70`, `+0x170`, `+0x270`, `+0x370` and the `+0x1000`
  group), both leave the memory-pattern test with **0 errors** — unlike the
  RX/TX delay words, where moving a code by a few taps immediately breaks
  reads.

So `+0x70` is the **reported code**, not the control. The applied value is the
low 6 bits of the byte-0/CS0 word `+0x64` (the same word the gate stage
modifies):

- period 32: writing codes that differ by 32 gives the same result, so the field
  is 5 bits (bit 5 of the byte is ignored);
- sweeping that byte around its trained value gives a contiguous passing
  window — measured at 2666 MT/s: 10..19, 42..52, 74..85, 106..118, 138..147
  (zero-error windows of the pattern test, ≈10–13 codes wide);
- the trained value (e.g. `0x4c`) sits near the **low edge** of its window, not
  at the centre, which is consistent with a DQS gate that must open before the
  preamble (asymmetric optimum, unlike a data tap);
- the high byte (`0x43`) is a qualifier: writing the whole word as 0 breaks
  reads **and locks the SoC**.

So the gate position **is** software-settable — through `+0x64`, not `+0x70`.

**Independence (measured).** With the Rx delay fixed at three points inside the
eye (15, 17 and 19 of a 7..28 window at 2400 MT/s), the `+0x64` window is
identical in all three: **10..19**. A shift of ±2 codes in the data delay does
not move it, so the two are orthogonal axes: `+0x64` is the gate position, not
another data delay.

Related words in the same block, measured the same way (writing four values and
running the pattern test, byte 0 / CS0):

| Word | Effect |
|---|---|
| `+0x28`, `+0x2c` | none: 0 errors with any value |
| `+0x50` | in the read path: writing 0 breaks reads |
| `+0x60` | in the read path: writing 0 breaks reads; `0xffffffff` restores, `0x1` does not |
| `+0x64` | low 6 bits = applied delay (window ≈10–13 codes at 2666); high byte = qualifier; writing 0 locks the SoC |

None of them produced a controllable gate window in that probe (the probe wrote
whole words, so field-level tests remain to be done). Treat `+0x64` as read-only
until its fields are understood.

## Gaps

- Meaning of the low byte `t` and why read training moves it.
- What the `0x43` high byte of `+0x64` is: the low 5 bits are the applied gate
  delay (measured), the high byte is a qualifier that must be preserved, but its
  meaning is not established.
- No example of a genuine RGATE timeout after a real command; the detection
  logic above covers it by construction, not by observation.

---

[← Previous: Write Leveling](write-leveling.md) | [Index](README.md#index) | [Next: Eye Training →](eye-training.md)
