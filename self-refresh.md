<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Mode Registers](mode-registers.md) | [Index](README.md#index) | [Next: Address Map and Geometry →](address-map.md)

# Self-Refresh and MC STATUS

Training runs **inside** self-refresh (SR). The PI uses the same CA and DQS
paths as the controller; parking the ranks in SR makes the PI the only owner of
the bus and keeps CPU/AXI traffic from colliding with it.

## Protocol

Through USER_0 (`ddr+0x20`), waiting on STATUS (`ddr+0x08`):

```
mode = (cs_num == 2) ? 3 : 1          /* rank mask */
mask = (cs_num == 2) ? 0x44 : 0x04     /* SR bits 2 and 6 */

enter:  USER_0 = 0x10000000 | (1 << 6) | (mode << 24)
        wait (STATUS & mask) == mask
exit:   USER_0 = 0x10000000 | (2 << 6) | (mode << 24)
        wait (STATUS & mask) == 0
```

With two ranks: enter `0x13000040`, exit `0x13000080`. One rank: `0x11000040`,
`0x11000080`, mask `0x04`. Budget 10 ms for each wait.

Using `mode = 1` on a two-rank package leaves CS1 refreshing on its own while
the PI trains it, and the CS1 gate fails.

## STATUS (`ddr+0x08`) bit semantics

STATUS is the general MC status, not a self-refresh latch. Bits `[6:0]` come in
CS0/CS1 pairs:

| Bit | Rank | Meaning |
|---|---|---|
| 0 | CS0 | DRAM init (Device_Init) of the rank complete. Device_Init waits for `0x1` (1 rank) or `0x11` (2 ranks). |
| 1 | CS0 | CKE / active (*inferred*). Set after Device_Init; cleared by the training SR exit on this board. |
| 2 | CS0 | **Self-refresh.** Enter/exit mask `0x04` (1 rank) or `0x44` (2 ranks). |
| 4 | CS1 | Like bit 0 |
| 5 | CS1 | Like bit 1 |
| 6 | CS1 | Like bit 2 |

The bits are **not** write-1-to-clear: writing `+0x08` does nothing to them.
They change only with USER_0 commands and with the MC reset. Device_Init sets
bits 0/4; SR enter sets 2/6; SR exit clears 2/6.

## Observed STATUS through the full sequence (2 ranks)

| Point | Command | STATUS | Notes |
|---|---|---|---|
| After Device_Init | — | `0x33` | Bits 0, 1, 4, 5; **not** in SR |
| After a HW-DFC | — | `0x33` | The DFC engine exits its own SR |
| Before SR enter | — | `0x33` | Start of a training pass |
| After SR enter | `0x13000040` | `0x77` | Bits 2 and 6 set |
| After SR exit | `0x13000080` | `0x11` | Bits 2/6 (and 1/5) cleared |
| Handoff to the SPL | — | `0x33` | Final DFC restores bits 1/5; `0x11` is also valid |

Rules derived from this:

- **Do not wait for `0x33` after the training SR exit**: the value is `0x11`
  and such a wait never completes. Wait only for `(STATUS & mask) == 0`.
- **`0x77` at handoff is a failure**: bits 2/6 still set means the pass never
  left SR (exit wait timed out, enter/exit not issued, or Device_Init never
  ran). The SPL's test at address 0 then fails or the memory is inaccessible.
- If STATUS is already `0x77` at the start of a pass, the enter is idempotent
  and its wait returns at once; the exit **must** still clear the bits. If
  `(STATUS & 0x44) == 0` does not happen within 10 ms, the MC is not honouring
  USER_0 — typically because Device_Init did not run.
- The HW-DFC engine **requires** bits 2/6 to be 0 at the kick. STATUS `0x77`
  when kicking the first DFC leaves the engine stuck in SR with
  `pmu+0x04` bit 22 set ([dfc-engine.md](dfc-engine.md)).

## Does the PI really need SR?

Not strictly at the hardware level: in a state where STATUS already read
`0x77` and the enter/exit commands did not change it, the exit wait expired
after 10 ms — and still every PI operation completed and the DRAM contents
survived (6 regions of 64 KiB checked, 0 errors).

The requirement nevertheless stays: train **in** SR (mask bits set) and
**leave** SR at the end (mask bits clear). Outside SR the PI races with AXI
traffic.

## Restrictions while in SR

- No mode-register reads; only the training MR14 write
  ([mode-registers.md](mode-registers.md)).
- No DRAM access by the CPU.
- Do not leave SR in the middle of a Vref sweep.

## Gaps

- The meaning of bits 1/5 (CKE/active) is inferred, and why the training SR
  exit clears them while the DFC restores them is not explained.
- Bits 3, 7 and above are not characterised.
- How the controller reaches the state in which STATUS stays `0x77` with
  enter/exit having no effect is not understood; the PI still worked in that
  state.

---

[← Previous: Mode Registers](mode-registers.md) | [Index](README.md#index) | [Next: Address Map and Geometry →](address-map.md)
