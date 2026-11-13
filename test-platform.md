<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: Acceptance Criteria and Reference Measurements](reference-measurements.md) | [Index](README.md#index)

# Test Platform: FDL, Software Resets, BootROM Media, Probes

This file describes the hardware facts behind the experimental setup used to
discover everything else: how images reach the SoC, how the board can be reset
by software, where the BootROM looks for the first stage, and how the real DRAM
clock can be measured. None of it is needed by `ddr_init` itself.

## FDL: BootROM USB download

The K1 BootROM tries the boot media and, when they fail (or when the FDL button
is held at reset), enters a USB download mode that offers a fastboot-style
interface. A signed first-stage image is pushed and started with:

```
fastboot stage <signed-first-stage.bin>
fastboot continue
```

The image runs from SRAM exactly as if it had been loaded from a boot medium.
An experiment image is the mainline SPL with a DDR image in the `ddr_fw.bin`
slot (the `u-boot-spl-ddr.bin` of [boot-interface.md](boot-interface.md)), signed with the
vendor signing tool.

Two practical notes from 2026-11-07, when the same board was used with a
different host:

- In that session the ROM enumerated as `361c:0008` with interface class 255 /
  subclass 66 / protocol 1 (ADB), and the stock `fastboot` client did not bind
  (`fastboot devices` stayed empty). The vendor flashing tool is what spoke to
  it. Whether a given host sees the fastboot or the ADB face of the ROM depends
  on the tool/host, not on the board state.
- A signed image can also be delivered without USB: write it to the medium's
  first-stage slot and reboot (see "Where the BootROM looks on eMMC"). That is
  how the later register/eye experiments were run, with the same image and the
  same boot path; it costs one reboot per run instead of one USB transfer.

Observations useful for automation:

- The BootROM prints `usb download handler` on the UART when it is in FDL;
  that string is a reliable "ready for the next run" signal.
- `fastboot devices` may still list a stale gadget after the SPL starts, so it
  is not a reliable readiness signal.
- UART: 115200 baud.
- Without a board EEPROM read (common in FDL), the mainline SPL prints
  `Fail to detect board:-2` and uses 2 ranks / 2400 MT/s.
- If the strap selects SPI NOR and none is fitted, the mainline SPL stops with
  `Unsupported Boot Device` after DDR init — harmless for DDR experiments.

### Loading without FDL: writing the boot medium

FDL is a *download* mode: the image runs from SRAM and the storage medium is not
touched. The production boot path writes the signed first-stage image into the
medium's `fsbl` slot and lets the BootROM load it at reset. Two hardware traps
were measured:

- The BootROM's fastboot `flash fsbl` **reports `OKAY` but writes nothing**:
  the `fsbl` partition it exposes has size 0, so `flash` only downloads to
  SRAM, and `continue` then runs that SPL in download mode, where it tries to
  fetch the next stage over the same fastboot gadget and stops
  (`SPL load failed`, `bootdev=12, loader=FASTBOOT`). `continue` is only useful
  with an image staged to SRAM via `fastboot stage`; it does not reboot from
  the storage medium.
- The FDL gadget enumerates as `361c:1001` (`DFU USB download gadget`), whereas
  the ROM's ADB interface enumerates as `361c:0008`; which interface appears
  depends on the strap and the host (see the FDL notes above).

On the BPI-F3, the SD card's first-stage slot is `mmcblk1p1` (256 KiB `fsbl`
partition, header magic `AIHD` at offset `0x100`). A running Linux system can
write it directly (`dd if=image of=/dev/mmcblk1p1 bs=4096 conv=fsync`).

**Recovery when the SD card image fails to boot.** If a written SD image hangs
during DDR training (or if a watchdog-reset test hook is left enabled in an
SD-booted image, causing an endless boot loop), the board cannot boot from SD.
To recover without an external card reader: power off, remove the SD card, boot
Linux from the onboard **eMMC**, and hot-plug the SD card once Linux is up. The
SD host controller uses an active-low card-detect GPIO (GPIO 80 on the BPI-F3)
and detects the insertion immediately, allowing the partition to be rewritten
with `dd`.

### Boot sequence printed by the BootROM

| Reset source | `sys` value printed | Path taken |
|---|---|---|
| RST button | `0x0` | `bm:3` (SD, fails) → `bm:0` (eMMC; `invalid bootinfo magic code:0x0` when empty) → FDL |
| FDL button | `0x1000` | `bm:2` → download mode directly |
| PMIC P1 soft reset | `0x0` (8 of 8) | same as the RST button |
| SoC watchdog | `0x0` (1 of 1) | same as the RST button |

`sys` reflects the boot flag register `0xd4282d10`. The mainline SPL writes its
own boot mode there (`0x55a` USB … `0x55e` SD); after an image loaded through
FDL it reads `0x0000055a`. A PMIC reset clears it.

## Software reset of the SoC (watchdog, MMIO only)

Works without I2C.

| Block | Address | Role |
|---|---|---|
| MPMU | `0xd4050000` | Clocks / reset of the watchdog block |
| WDT | `0xd4080000` | Watchdog timer |
| Start | `0xd4051020` | bit 4 starts the counter |

`MPMU+0x200` (WDTPCR): bit 0 = APB clock of the WDT, bit 1 = timer clock
(25.6 MHz / 100 → 256 Hz count), bit 2 = block reset (1 = held in reset; clear
it for the MMIO to respond).

Every write to the WDT must be **immediately** preceded by the unlock pair:

```
WDT+0xB0 = 0xbaba
WDT+0xB4 = 0xeb10
WDT+off  = value
```

| Offset | Value | Role |
|---|---|---|
| `+0xC0` | 0 | Clear status |
| `+0xBC` | N | Timeout in ticks of the 256 Hz clock (1 ≈ 4 ms; 0 fires at once) |
| `+0xB8` | 3 | Enable |
| `+0xC8` | 1 | Timeout causes a SoC reset |

Sequence (does not return):

```
MPMU+0x200 |= 0x3            /* clocks */
MPMU+0x200 &= ~0x4           /* release block reset */
unlock; WDT+0xC0 = 0
unlock; WDT+0xBC = 1
unlock; WDT+0xB8 = 3
unlock; WDT+0xC8 = 1
*(0xd4051020) |= (1 << 4)    /* start */
loop forever
```

If the WDT registers always read 0, its clocks/reset are not set correctly.
Clearing `0xd4282d10` before arming makes the next `sys` value reflect only the
watchdog.

## Board reset through the PMIC (closest to the RST button)

SpacemiT P1 on **I2C bus 8**, address **`0x41`** (BPI-F3 / `k1-x_deb1`). Register
numbers from the public P1 map (the same `P1_*` names as in U-Boot):

| Register | Bit | Role |
|---|---|---|
| `0x44` | 0 | WDT_CTRL: W1C status |
| `0x44` | 1–2 | Timeout (0 = 1 s … 3 = 16 s) |
| `0x44` | 3 | WDT enable |
| `0x7C` | 7 | PWR_CTRL0: the PMIC watchdog causes a reset |
| `0x7E` | 1 | PWR_CTRL2: **SW_RST** (mask `0x02`) |
| `0xAB` | 2 | ALIVE: reboot flag (mask `0x04`) |

Immediate reset:

```
P1[0xAB] |= 0x04      /* reboot flag, not "charging" */
P1[0x7E] |= 0x02      /* SW_RST; does not return, ≈100 ms to the BootROM */
```

The rails go down and up: this matches the RST button (`sys: 0x0`). The FDL
button (`sys: 0x1000`) is a strap and cannot be reproduced by software.

Measured: `P1[0xAB]` already reads `0xf4` (bit 2 set), `P1[0x7E]` reads 0. At
the point where the SPL runs, I2C8 and its pin mux are often not configured
yet; the SoC watchdog works without them.

## Where the BootROM looks on eMMC

- The BootROM reads `bootinfo` (80 bytes, magic `f01407b0`) at LBA 0 of the
  eMMC `boot0` partition. On the board: `spl0_offset = 0x200`,
  `spl1_offset = 0` (backup copy at LBA 0 of `boot1`),
  `spl_size_limit = 0x36000`.
- The first stage must therefore sit at **LBA 1 of `boot0`** (offset `0x200`).
  An image at 64 KiB or 128 KiB is rejected (`Failed to verify spl0/spl1`) and
  the BootROM falls back to FDL.
- A signed first-stage image at that offset passes the BootROM signature check
  (TPKS) and runs from SRAM exactly as an FDL-loaded one.

## DRAM latency probe (frequency meter)

Because the PLL block has no frequency counter, the real DRAM clock is measured
by latency:

- build a cyclic chain of 64-byte nodes at `0x10000000`: random order
  (Sattolo's algorithm) over 16 KiB, 256 KiB and 32 MiB, and one 32 MiB chain
  in address order;
- run 2²⁰ dependent loads per chain and time them with `mtime`
  (`0xe400bff8`, 24 MHz) and the cycle counter;
- report ns per load and the CPU frequency (`cycle / mtime`).

With the data cache off the CPU runs at 307 MHz in the SPL and the cache does
not filter (all sizes cost the same); with the data cache on (1599 MHz) use the
random 32 MiB chain (sequential is hidden by the prefetcher, 16 KiB fits in L1).
A PLL2-based point (fp1) is the control: it must not change between PLL1
recipes. Results in [reference-measurements.md](reference-measurements.md).

## User-space probes (Linux)

Once the boot reaches Linux, two user-space workloads characterise the trained
DRAM:

- **Throughput and latency benchmark:** 256 MiB buffers (one pass each of
  sequential 64-bit write, read, and copy) and a 32 MiB random pointer chase on
  huge pages (`MADV_HUGEPAGE`) for latency, run with 1 process (single-core) and
  4 concurrent processes (128 MiB each, multi-core aggregate).
- **Multi-process memory soak:** 4 concurrent processes (512 MiB each, 2 GiB
  resident) continuously writing and verifying 64-bit patterns in a loop for
  120–900 s to test long-term data integrity under load at 2666 MT/s.

Neither replaces the register-level probes; they measure end-to-end stability
and performance. Results are in
[reference-measurements.md](reference-measurements.md).

## Gaps

- The BootROM's FDL protocol beyond `stage` / `continue` and the image
  signature format were not studied (the vendor tool was only executed).
- **USB identity of the download mode (observed 2026-11-07):** the BootROM
  enumerates as `361c:0008` with interface class 255 / subclass 66 /
  protocol 1 (ADB), so the stock `fastboot` client does not bind to it
  (`fastboot devices` stays empty). The vendor TitanFlasher tool is what
  talks to this ROM. A host-side loader for the ROM would have to speak
  that ADB-shaped transport, not plain fastboot.
- **Card-detect GPIO number and hot-plug behaviour** were measured on the
  BPI-F3 (GPIO 80, active low); other K1 boards may wire card-detect
  differently.
- The PMIC register map is taken from the public P1 description; only the bits
  above were exercised.

---

[← Previous: Acceptance Criteria and Reference Measurements](reference-measurements.md) | [Index](README.md#index)
