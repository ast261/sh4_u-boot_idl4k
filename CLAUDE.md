# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a U-Boot 1.3.1 bootloader fork for the **idl4k** board (STMicroelectronics STx7108 SoC, SH4-300 dual-core CPU).

## Build System

U-Boot uses a two-step build: first configure for the board, then compile.

**Configure** (must run before first build, and after `make distclean`):
```sh
make idl4k_config
```

**Build:**
```sh
make                          # build in-tree
make O=/tmp/build             # build out-of-tree (preferred)
make UBOOT_DEBUG=1            # build with -O0 for debugging
```

The cross-compiler prefix defaults to `sh4-linux-` (set `CROSS_COMPILE` to override). The STSDK environment variables must be set before building.

**Clean:**
```sh
make clean       # remove object files
make distclean   # remove everything including config (run before reconfiguring)
```

Outputs: `u-boot` (ELF), `u-boot.bin`, `u-boot.srec`, `System.map`.

## Debugging on Hardware

Use the debug script with JTAG via STMicroConnect. Requires `AXE_MICROCONNECT_IP_ADDRESS` to be set:

```sh
./debug_idl4k.sh    # uses sh4-linux-gdb + sh4tp to load and run u-boot
```

## Updating the Bootloader on Target

U-Boot runs from SPI serial flash (MX25L8006E), mapped at offset 0 with 512 KiB (`0x80000`) reserved. The environment is stored immediately after at offset `0x80000` and is not affected by a U-Boot update.

The `eeprom` command is the ST40 abstraction for SPI flash access (device address is implicit).

**USB port:** U-Boot's OHCI driver handles only one controller, selected by `CFG_USB_BASE` in `include/configs/idl4k.h`. On this branch (`idl4k-usb-lower-port`) it is USB #1 (`CFG_USB1_BASE`), the **lower** port, because the upper port's socket has a loose contact; `master` uses USB #0, the upper port. Port mapping, confirmed from Linux: upper port = USB #0 (`stm-ehci.0`, `0xfe0ffe00`), lower port = USB #1 (`stm-ehci.1`, `0xfe1ffe00`). USB #2 (`0xfe200000`) is assumed to be the front port but is unverified. Sticks in the other ports are not visible to U-Boot. If `usb start` reports `0 Storage Device(s) found`, read the root-hub port status with `md.l fe1ffc54 1` (USB #0: `fe0ffc54`): bit 8 = port powered, bit 0 = device connected, bit 3 = overcurrent.

**From the U-Boot prompt:**

```
usb start
fatload usb 0 84000000 u-boot.bin
eeprom write 84000000 0 35AC4
reset
```

Replace `35AC4` with the actual file size in hex (`219844` bytes = `0x35AC4` for the current build). Using the exact size avoids erasing sectors beyond the binary.

**Automated update via USB script (`updatefw`):**

On every boot, U-Boot checks for `idl4k.scr` on the USB stick and executes it if found. This is the intended field-update mechanism. Create the script with `mkimage`, place it on a FAT-formatted USB stick as `idl4k.scr`, and the board will run it automatically before booting the firmware.

**NAND flash layout (informational):**

| Partition | Size | Offset | Contents |
|-----------|------|--------|----------|
| env-nand0 | 128 KiB | 0x0 | NAND-side environment (not active) |
| system-nand0 | 4 MiB | 0x20000 | System |
| fw1-nand0 | 32 MiB | 0x420000 | Primary firmware (booted by `bootcmd`) |
| fw2-nand0 | 32 MiB | 0x2420000 | Secondary firmware |
| data-nand0 | ~62 MiB | 0x4420000 | User data |

## Networking

**DHCP / Ethernet does not work on this board.** The on-board PHY is an RTL8211EG (gigabit, RGMII-strapped), but the STx7108 GMAC in this U-Boot port has no RGMII support — only MII/RMII/GMII/GMII-GTX/reverse-MII. The MAC↔PHY interface mode mismatch means the PHY shows link and activity but emits no valid Ethernet frames. There is no software-only fix in this U-Boot tree.

Do not attempt to "fix" `dhcp` / `ping` / `tftpboot`. The `CONFIG_STMAC_IP1001` declaration in `include/configs/idl4k.h` is the historical config; treat it as inert. Use the USB-stick `updatefw` path described above for all field updates.

## Architecture

### Directory Layout

| Path | Purpose |
|------|---------|
| `board/st/idl4k/` | Board-specific init, platform settings (MAC, serial, UUID from flash at `0xFFC00`) |
| `cpu/sh/` | SH4 CPU core (start, interrupts, NAND, SPI FSM, SATA) |
| `cpu/sh/stx7108/` | STx7108 SoC peripheral init |
| `lib_sh/` | SH4 architecture library (board.c, cache ops, SH-specific tests) |
| `include/configs/idl4k.h` | All compile-time options (boot mode, memory map, commands) |
| `include/asm-sh/` | SH4 register definitions and SoC headers |

### Boot Mode Selection (idl4k)

The idl4k config supports three boot modes selected at compile time in `include/configs/idl4k.h`:
- `CFG_BOOT_FROM_SPI` — boot from SPI serial flash (default for idl4k)
- `CFG_BOOT_FROM_NAND` — boot from NAND flash
- Neither defined — boot from NOR flash

Memory base addresses (`CFG_EMI_*`) shift depending on which boot mode is selected, affecting the EMI chip-select mapping.

### Platform Settings (idl4k)

When `CONFIG_PLATFORM_SETTINGS_INIT` is defined, `board/st/idl4k/idl4k.c` reads a `idl4k_platform_settings_t` struct from NOR flash at `0xFFC00` containing: hardware ID, serial number, MAC address, and UUID. These are used to set U-Boot environment variables at runtime.

### `romgen` Files

`board/st/idl4k/idl4k-stx7108.romgen` (and similar for other boards) is an auto-generated GDB initialization script produced by STMicroelectronics `romgen` tool. It programs SoC clocks, DDR, and other hardware registers via the JTAG debug probe before U-Boot loads. Do not edit manually.

### mkconfig

`mkconfig` creates `include/config.mk` (setting `ARCH`, `CPU`, `BOARD`, `VENDOR`, `SOC`) and the `include/asm` symlink. It also conditionally sets `FTA_LIBRARY=yes`. Run via `make <board>_config`, never directly.
