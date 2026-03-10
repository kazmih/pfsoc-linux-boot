# PolarFire SoC Linux Boot on Renode (Antmicro Pre-built Binaries)

## Branch: renode-antmicro-boot

This branch documents a fully working Linux boot to an interactive shell on the Renode-emulated PolarFire SoC Icicle Kit, using Antmicro's pre-built binary set matched to the Renode platform model. The boot is complete: HSS initializes all cores, U-Boot loads the kernel from the FIT image, Linux boots, and a root shell is accessible on mmuart1.

This approach differs from the `main` branch which attempts a direct fw_payload boot (OpenSBI + kernel, no HSS or U-Boot). That approach boots the kernel but does not reach an interactive shell due to UART interrupt limitations in the single-core Renode setup. This branch uses the full hardware boot flow and reaches a working login prompt.

---

## What Was Achieved

- HSS (Hart Software Services) v0.99 boots on mmuart0, initializes DDR and all five cores, and loads U-Boot.
- U-Boot 2020.01 boots on mmuart1, reads the FIT image from RAM at 0x88300000, verifies sha1 hashes, and loads the kernel, initramfs, and DTB.
- Linux kernel boots fully on mmuart1 with all subsystems initializing.
- BusyBox Buildroot userspace starts: syslogd, klogd, sysctl, mdev, networking (eth0).
- Interactive login prompt appears: `buildroot login:`
- Root shell accessible with credentials `root / root`.

---

## Boot Flow

```
Renode loads hss.elf to eNVM (0x20220000)
    |
    v
HSS (E51, M-mode)
  - Initializes DDR, clocks, PMP
  - Loads U-Boot from eMMC to DDR (0x80200000)
  - Releases U54 application cores
    |
    v
U-Boot (U54_1, S-mode, 0x80200000)
  - Reads fitImage.fit from RAM at 0x88300000
  - Verifies kernel, initramfs, DTB hashes (sha1)
  - Loads kernel to 0x80200000, initramfs to 0x84000000, DTB to 0x82200000
  - Jumps to kernel entry
    |
    v
Linux kernel (U54_1, S-mode)
  - Reads DTB, discovers hardware
  - Mounts initramfs (BusyBox Buildroot rootfs)
  - Starts /init -> login prompt on mmuart1
```

---

## Hardware Platform

- Board: Microchip PolarFire SoC Icicle Kit
- SoC: MPFS095T-FCSG325
- CPU: 5x RISC-V cores (1x E51 monitor + 4x U54 application cores), RV64GC
- Emulated by: Renode 1.13.1 (bundled with SoftConsole)
- Platform description: `F:\SoftConsole\renode\scripts\single-node\polarfire-soc.repl` (Antmicro, bundled)

---

## Required Binary Files

These files are not committed to the repository due to size. Download them before running.

All files are pre-built by Antmicro and are matched to each other and to the Renode platform model bundled in SoftConsole.

**hss.elf** (~3.2 MB) — Hart Software Services bootloader ELF:
```
https://dl.antmicro.com/projects/renode/icicle--hss.elf-s_3297936-bcb7ef60abc78a878995554160eaac1dea962e95
```

**emmc.img** (~25.5 MB) — eMMC flash image containing U-Boot:
```
https://dl.antmicro.com/projects/renode/icicle--emmc.img-s_26746880-3a6ef871bc8eb6fcfbda344e8c23fb534ef89961
```

**u-boot** (~5 MB) — U-Boot ELF for symbol loading only:
```
https://dl.antmicro.com/projects/renode/icicle--u-boot-s_5132448-194bf14572a9bc4b26727567065ede2ffd7f1201
```

**vmlinux** (~8.2 MB) — Linux kernel ELF for symbol loading only:
```
https://dl.antmicro.com/projects/renode/icicle--vmlinux-s_8563992-fa2aad1e61ec38b411f6afb543503cb26601b1e2
```

**fitImage.fit** (~16 MB) — FIT image containing kernel, initramfs, and DTB:
```
https://dl.antmicro.com/projects/renode/icicle--fitImage.fit-s_16976563-1d0e86ed4cc7c24e167ca899fd929d954956b478
```

Place all five files in `F:\Downloads\` before running.

---

## Repository File Structure

```
linux_boot/
    boot.resc       Renode boot script (this branch)
    BINARIES.md     Binary download links (this file summarized)
```

The platform description and .repl file used in this branch are from the SoftConsole installation at:
```
F:\SoftConsole\renode\scripts\single-node\polarfire-soc.resc
F:\SoftConsole\renode\platforms\cpus\polarfire-soc.repl
```

---

## Boot Script

`linux_boot/boot.resc`:

```
i @F:\SoftConsole\renode\scripts\single-node\polarfire-soc.resc

emulation SetGlobalSerialExecution true
emulation SetGlobalQuantum "0.000001"

showAnalyzer mmuart1

machine SdhcCardFromFile @F:\Downloads\emmc.img mmc

macro reset
"""
    sysbus LoadELF @F:\Downloads\hss.elf
    sysbus LoadBinary @F:\Downloads\fitImage.fit 0x88300000
    sysbus LoadSymbolsFrom @F:\Downloads\vmlinux_renode
    sysbus LoadSymbolsFrom @F:\Downloads\u-boot
"""

runMacro $reset
```

---

## How to Run

```
"F:\SoftConsole\renode\bin\Renode.exe" --plain -e "include @F:\FPGA\Task5\linux_boot\boot.resc"
```

In the Renode monitor type:
```
start
```

Watch mmuart0 for HSS output. Watch mmuart1 for U-Boot and Linux output. Boot to login prompt takes approximately 10-15 minutes due to `SetGlobalSerialExecution true` serializing all hart execution for determinism.

Login with `root` / `root`.

---

## Key Design Decisions

**Why SetGlobalSerialExecution true is required.**

The PolarFire SoC has 5 RISC-V harts executing concurrently. In Renode's default parallel execution mode, the timing between harts is non-deterministic. At the point where the Linux kernel initializes SMP and the secondary U54 cores begin executing, a race condition causes one or more cores to jump to a garbage address and abort with `CPU abort: Trying to execute code outside RAM or ROM`. Serial execution forces all harts to take turns in a round-robin, eliminating the race.

**Why SetGlobalQuantum "0.000001" is required.**

The default quantum of `0.0008` (800 microseconds of emulated time per turn) is still too coarse. With a large quantum, a hart can run far ahead of others before yielding, which causes the same SMP race condition even in serial mode. Reducing the quantum to 1 microsecond forces very fine-grained interleaving and produces a deterministic, successful boot. This is the value recommended by Antmicro for this platform.

**Why Antmicro's pre-built binaries are used instead of the Buildroot SDK output.**

The Buildroot SDK produces binaries targeting physical PolarFire SoC hardware with physical DDR at 0x1000000000, sha256 hashes in the FIT image (unsupported by U-Boot 2020.01), and load addresses in the 0x1000000000 range. Antmicro's binaries are pre-adapted for the Renode platform: DDR at 0x80000000, sha1 hashes, and load addresses within the emulated memory map. Using the Buildroot SDK output requires patching the FIT image .its load addresses, replacing sha256 with sha1, replacing the embedded DTB with a Renode-modified version, and rebuilding. These steps are documented on the main branch.

**Why the eMMC image is needed.**

HSS loads U-Boot from the eMMC flash. In Renode, the eMMC is emulated as a file mapped to the sdhc peripheral. The fitImage is loaded directly into RAM at 0x88300000 by the Renode script, bypassing the eMMC for the kernel image. HSS and U-Boot still need the eMMC for the U-Boot binary itself.

**Why fitImage is loaded at 0x88300000.**

U-Boot is configured to search for the FIT image at this address. After HSS releases the U54 cores and U-Boot starts, it reads from 0x88300000, parses the FIT image header, and loads the kernel, initramfs, and DTB to their respective load addresses before booting the kernel.

---

## Tools Used

- Renode 1.13.1 (Antmicro, bundled with SoftConsole): Hardware emulator providing the PolarFire SoC platform model including all 5 RISC-V cores, CLINT, PLIC, NS16550 UARTs, eMMC/SDHC controller, PCIe, Ethernet, and DDR.
- SoftConsole (Microchip): IDE providing the bundled Renode installation and platform scripts.
- Antmicro pre-built binaries: HSS, U-Boot, Linux kernel, initramfs, and FIT image pre-adapted for the Renode PolarFire SoC platform.
- HSS (Hart Software Services, Microchip open source): M-mode bootloader running on the E51 monitor core. Initializes DDR, configures PMP, and loads U-Boot to DDR from eMMC.
- U-Boot 2020.01 (Microchip fork): S-mode bootloader. Reads FIT image, verifies hashes, loads kernel and initramfs to DDR, passes DTB address to kernel.
- Linux kernel (Antmicro build, Buildroot-based): Full Linux kernel with BusyBox userspace and Buildroot rootfs in initramfs.
