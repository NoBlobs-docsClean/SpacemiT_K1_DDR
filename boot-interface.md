<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright (C) 2026 Thalles Campagnani -->

[← Previous: System Overview](system-overview.md) | [Index](README.md#index) | [Next: MMIO Map and Register Index →](mmio-map.md)

# Boot Interface: Contract with Mainline U-Boot SPL

A DDR image for the K1 meant to be loaded by **mainline U-Boot** has to fit the
slot that U-Boot already provides for the proprietary `ddr_fw.bin`, without any
change to U-Boot: same file name, same load address, same entry point, same
calling convention. This file describes that slot.

## How mainline U-Boot integrates the firmware

### Build time

The board `-u-boot.dtsi` files (for example `k1-bananapi-f3-u-boot.dtsi`) declare
a binman section that concatenates the SPL and the DDR firmware:

```
u-boot-spl-ddr {                /* output: u-boot-spl-ddr.bin */
        type = "section";
        u-boot-spl { };
        ddr-fw {
                type = "blob";
                filename = "ddr_fw.bin";
                align = <64>;
        };
};
```

The file is provided at build time through the `DDR_FW_FILE` environment variable
(`export DDR_FW_FILE=/path/to/ddr_fw.bin`). The resulting `u-boot-spl-ddr.bin`
is what gets signed and placed where the BootROM looks for the first stage (or
pushed over FDL).

### Run time (`ddr_early_init` in `board/spacemit/k1/spl.c`)

1. Look up the `ddr-fw` blob position and size through binman symbols.
2. `memcpy` the raw bytes to `DDR_FIRMWARE_BASE = 0xc082d000` (SRAM).
3. Execute `fence.i` so the copied bytes are visible to instruction fetch. No
   relocation, no `.data` copy and no `.bss` clearing take place.
4. Read the board EEPROM (TLV format) for the rank count, memory type, data
   rate and Tx ODT.
5. Call the firmware:

   ```c
   typedef int (*ddr_init_func_t)(u64 ddr_base, u32 cs_num, u32 data_rate,
                                  void (*puts)(const char *));
   ddr_init(0xc0000000, cfg.cs_num, cfg.data_rate, puts);   /* DEBUG build */
   ddr_init(0xc0000000, cfg.cs_num, cfg.data_rate, NULL);   /* normal build */
   ```

6. **Ignore the return value**, write `0xaa55aa55` to physical address 0,
   flush/invalidate that cache line, read it back, and print
   `DDR is ready` or `DDR is not ready`.

Defaults when the EEPROM cannot be read (common when booting over FDL; the SPL
prints `Fail to detect board:-2`):

| Field | Default |
|---|---|
| `cs_num` | 2 |
| memory type | `"LPDDR4X"` (read but **not passed** to the firmware) |
| Tx ODT | 80 Ω (read but **not passed**) |
| `data_rate` | 2400 |

## The ABI

```c
int ddr_init(u64 ddr_base, u32 cs_num, u32 data_rate, void (*puts)(const char *));
```

Standard RISC-V LP64 calling convention, entered at **offset 0** of the image
(`0xc082d000`):

| Register | Argument | Values / rules |
|---|---|---|
| `a0` | `ddr_base` | MC base, `0xc0000000` on the K1. All MC/PHY addresses are derived from it (`phy = ddr + 0x40000`). |
| `a1` | `cs_num` | 1 or 2: the number of ranks actually populated. Using 1 on a 2-rank package leaves CS1 untrained. |
| `a2` | `data_rate` | Requested final rate in MT/s. See the mapping below. |
| `a3` | `puts` | Optional UART output routine of the SPL. **May be NULL.** If present it is valid RISC-V code in SRAM. The firmware must not require it. |
| `a4`, `a5` | — | **Undefined** (caller garbage). The ABI does not grow; do not read them. |
| return | `int` | 0 on success, negative on failure. The mainline SPL ignores it. |

### `data_rate` mapping

| Requested | Final fp | Notes |
|---|---|---|
| 2400 | 2 | PLL1 VCO for 2400 |
| 2666 | 2 | PLL1 VCO for 2666, same fp2 bank and slice; SoC maximum |
| 1066 | 1 | |
| 1200 | 0 | |
| 1333 | 0 | No table entry. The hardware runs ≈1333 by programming PLL1 with the 2666 VCO and ending at fp0 (`PLL1 / 2`); a plain ABI request for 1333 is treated as 1200 unless the image special-cases it. |
| 2133 | 3 on LPDDR4 | On LPDDR4X there is no fp3: treated as 1200 (final DFC back to fp0, measured) |
| anything else (1600, 2667, 2700, 2933, 3200, 4266 …) | 0 | Treated as 1200. Never round up to 2666. (1333 is the special case above.) |

### What the ABI does not carry

The SPL reads the memory type (LPDDR4 vs LPDDR4X) and the Tx ODT from the
EEPROM but does not pass them. The silicon offers **no way** to recover them:

- no JEDEC mode register reports LPDDR4 vs LPDDR4X;
- no strap, GPIO, PMU bit or eFuse distinguishes them before Device_Init
  (`0xd4282c20[10:9]` is the boot-media strap: eMMC / SPI / NAND / SD);
- registers such as `pmu+0x398`, PHY COMMON or `ddr+0x30c/0x310` echo what the
  firmware programmed, not what the die is.

Consequences:

- **One image per memory type.** An image for the mainline slot has to be
  built for a fixed type; for boards like the reference one (and for the SPL's
  default type) that is LPDDR4X with Tx ODT 80 Ω. An LPDDR4 board needs a
  different `ddr_fw.bin` built for LPDDR4 (Tx ODT 120 Ω). A single image
  supporting both would require changing the SPL and the ABI.
- Tx ODT goes 1:1 with the type (80 Ω for 4X, 120 Ω for 4). EEPROMs may
  specify other values for other 4X SKUs; this ABI cannot honour them.
- The type must not be guessed from MR5 (manufacturer) or from leftover SRAM
  contents.

### The `0xc0800000` block

Some first stages write a 1536-byte DDR information block at `0xc0800000`
(little-endian; it contains a `ddr_type[16]` ASCII field at `+0x24`, no Tx ODT).
It is written **after** `ddr_init` returns. Before the call (and always under
FDL) it holds zeros or leftovers. It is not part of the contract: do not read it,
neither for the algorithm nor to choose the memory type.

## Execution environment

| Property | Value |
|---|---|
| Privilege / harts | M-mode, single hart (the SPL's) |
| Code location | Raw, position-dependent image linked at `0xc082d000` |
| Usable window | `0xc082d000` up to the BootROM data area near `0xc0838000`: 0xb000 bytes (44 KiB) for code and read-only data. The SPL itself uses a BootROM storage-API pointer at `0xc0838498`, which shows that this area belongs to the BootROM. |
| Writable data | **None in the image.** Since nothing relocates the image or initialises `.data`/`.bss`, all mutable state must live on the stack. |
| Stack | The SPL's stack (SRAM). |
| DRAM | Not usable (it is what is being initialised). |
| Caches | The SPL only issues `fence.i` after the copy; the firmware must not depend on cache maintenance instructions that may trap at this point. |
| Time base | `mtime` at `0xe400bff8`, 24 MHz, is usable for delays and timeouts. |

## Output contract

When `ddr_init` returns 0:

- the DRAM is at the final frequency point (fp2 for 2400/2666, fp1 for 1066,
  fp0 for 1200/1333; 1333 uses the 2666 VCO), out of self-refresh, refreshing
  normally;
- every frequency point that the boot may use has been trained;
- the address map covers the full detected capacity (8192 MB on the reference
  board): the low 2 GiB at CPU address `0x0`, the rest from `0x1_0000_0000`
  ([address-map.md](address-map.md));
- MC STATUS reads `0x33` (2 ranks; `0x11` is also acceptable), never `0x77`
  ([self-refresh.md](self-refresh.md));
- the PHY training status `phy+0x18000` is 0.

On failure `ddr_init` must return a negative code and must **not** continue
after a failed training pass (`PHY_STATUS != 0`, read-gate timeout) — running
with a bad DRAM is worse than stopping — but since the mainline SPL ignores the
return value, a clear UART message (when `puts` is available) is the only way
to report it.

## Gaps

- The mainline SPL ignores the return value; a failing `ddr_init` can only be
  detected by the magic-word test at address 0 (which may still pass with a
  marginal DRAM) or by the UART log.
- Memory type and Tx ODT cannot reach the firmware through this ABI (see
  above). Supporting LPDDR4 boards with the same SPL requires either a second
  binary or an SPL/ABI change upstream; this is a software interface question,
  not something more hardware investigation can solve.
- No measurements exist for boards with `cs_num = 1`.

---

[← Previous: System Overview](system-overview.md) | [Index](README.md#index) | [Next: MMIO Map and Register Index →](mmio-map.md)
