# PolarFire SoC Linux Boot on Renode

## Overview

This repository documents the process of building a Linux boot image for the Microchip PolarFire SoC Icicle Kit and booting it on the Renode hardware emulator. The project uses the Buildroot SDK provided by Microchip to compile OpenSBI, Linux kernel 5.15.68, and a BusyBox-based initramfs, then boots the resulting firmware on a Renode-emulated PolarFire SoC platform.

---

## Hardware Target

- Board: Microchip PolarFire SoC Icicle Kit
- SoC: MPFS095T-FCSG325
- CPU: 5x RISC-V cores (1x E51 monitor core + 4x U54 application cores), RV64GC ISA
- Relevant memory map:
  - DDR: 0x80000000 (1 GB)
  - CLINT: 0x02000000
  - PLIC: 0x0C000000
  - UART0: 0x20000000
  - UART1: 0x20100000

---

## Boot Sequence Theory

On real PolarFire SoC hardware, the boot sequence is:

1. Power on: The E51 monitor core executes the Hart Software Services (HSS) bootloader from eNVM at 0x20220000. HSS initializes DDR, clocks, and peripheral configuration, then loads subsequent boot stages and releases the U54 application cores.
2. OpenSBI (M-mode, 0x80000000): Provides the Supervisor Binary Interface (SBI) used by the OS to perform privileged operations such as timer management, IPI, and system reset.
3. U-Boot (S-mode, 0x80200000): Locates and loads the Linux kernel and device tree blob from storage, then transfers control to the kernel.
4. Linux kernel (S-mode, 0x80400000): Reads the device tree blob (DTB) to discover hardware, loads drivers, mounts the root filesystem, and starts the init process.

For Renode emulation, the HSS and U-Boot stages are bypassed. OpenSBI is built with the Linux kernel embedded as its payload (fw_payload.bin), and the modified DTB is embedded into OpenSBI at build time. This reduces the boot chain to: OpenSBI -> Linux kernel directly.

---

## What Was Achieved

- Full Buildroot SDK compilation on Ubuntu 22.04 in VMware, producing all necessary boot artifacts.
- OpenSBI v0.9 boots successfully on the emulated PolarFire SoC platform in Renode.
- Linux kernel 5.15.68 (linux4microchip+fpga-2022.09) fully boots and initializes on the emulated platform.
- Initramfs is successfully unpacked by the kernel (~19 MB BusyBox-based root filesystem).
- All major kernel subsystems initialize: memory management, SMP (single core), RCU, networking stack, SCSI, USB, PCI, and serial drivers.
- UART1 (0x20100000) is probed and registered as ttyS1 by the 8250/16550 driver.
- The earlycon (ns16550a at 0x20100000) provides console output throughout the entire boot sequence via the keep_bootcon mechanism.
- The BusyBox init process runs from the initramfs.

## What Remains Incomplete

Interactive shell access is not functional in this Renode setup. The BusyBox shell runs after init but cannot be used interactively. The 8250/16550 driver in Linux uses interrupt-driven TX and RX. Renode's NS16550 peripheral model requires working PLIC interrupt delivery for the UART to function interactively. The interrupt path from the UART through the PLIC to the running hart does not fully support the character-by-character I/O needed by a terminal session. As a result, the shell starts but produces no output and accepts no input.

This limitation is specific to Renode emulation. On real PolarFire SoC hardware, a full interactive login prompt would appear on UART1 at 115200 baud after the boot sequence completes.

---

## Repository File Structure

```
linux_boot/
    icicle.repl               Renode platform description for PolarFire SoC Icicle Kit
    boot.resc                 Renode boot script
    riscvpc.dts               Modified device tree source for Renode
    riscvpc_renode.dtb        Compiled device tree blob for Renode
    fw_payload.bin            OpenSBI + Linux kernel payload (load at 0x80000000)
    initramfs_renode.cpio.gz  Modified BusyBox initramfs (load at 0x88000000)
```

Original Buildroot SDK output artifacts (unmodified, for reference or real hardware use):
```
fitImage.fit          FIT image containing kernel, DTB, and initramfs for real hardware boot
payload.bin           HSS payload binary for real hardware
vmlinux.bin           Linux kernel binary (PE32+ RISC-V format, 17 MB)
initramfs.cpio.gz     Original BusyBox root filesystem (20 MB)
riscvpc.dtb           Original device tree blob targeting real hardware
```

---

## Build Environment

- Host OS: Ubuntu 22.04 LTS running in VMware Workstation on Windows 11
- SDK: polarfire-soc/polarfire-soc-buildroot-sdk (Microchip, archived November 2022)
- Toolchain: riscv64-unknown-linux-gnu- (built internally by Buildroot)
- Renode: bundled with SoftConsole (Microchip IDE)

---

## Step-by-Step Build Process

### Step 1: Clone the Buildroot SDK

```bash
git clone https://github.com/polarfire-soc/polarfire-soc-buildroot-sdk.git
cd polarfire-soc-buildroot-sdk
git submodule update --init --recursive --depth=1
```

The Linux kernel and riscv-gnu-toolchain submodules are large. Use --depth=1 for a shallow clone to reduce download size. Allow significant time for cloning.

### Step 2: Install Build Dependencies

```bash
sudo apt install -y gawk texinfo u-boot-tools mtools
```

These packages are not present by default on Ubuntu 22.04 but are required by the Buildroot build system.

### Step 3: Expand VM Disk Space

The full build requires approximately 35-40 GB of disk space. Expand the VMware virtual disk to at least 50 GB before building, then extend the partition from within Ubuntu:

```bash
sudo growpart /dev/sda 3
sudo resize2fs /dev/sda3
```

### Step 4: Build the Buildroot SDK

```bash
make DEVKIT=icicle-kit-es
```

This builds the full software stack including the RISC-V cross-compiler toolchain, Linux kernel, OpenSBI, U-Boot, BusyBox, and all userspace packages. Build time is approximately 1-3 hours.

Key output files produced under `work/`:
- `fitImage.fit` (37 MB): FIT image for real hardware
- `payload.bin` (546 KB): HSS payload for real hardware
- `vmlinux.bin` (17 MB): Linux kernel binary
- `initramfs.cpio.gz` (20 MB): BusyBox root filesystem
- `riscvpc.dtb` (17 KB): Device tree blob

### Step 5: Modify the Device Tree for Renode

The original DTB targets physical PolarFire SoC hardware and contains values incompatible with Renode. The following modifications are required.

Decompile the DTB to DTS source:

```bash
cd work
dtc -I dtb -O dts riscvpc.dtb -o riscvpc.dts
```

Apply the following modifications. Each is applied using Python for reliable multi-line text replacement:

**a. Fix the DDR memory node address.**

The original DTB places DDR at physical address 0x1000000000 (64 GB offset). Renode maps DDR starting at 0x80000000:

```bash
sed -i 's/reg = <0x10 0x00 0x00 0x76000000>/reg = <0x00 0x80000000 0x00 0x40000000>/' riscvpc.dts
```

**b. Disable cpu@2, cpu@3, and cpu@4.**

Only hartid=1 (cpu@1) runs in Renode. Disabling the other U54 cores prevents the kernel SMP bringup from hanging while waiting for secondary CPUs:

```python
import re
with open('riscvpc.dts', 'r') as f:
    content = f.read()
for cpu in ['cpu@2', 'cpu@3', 'cpu@4']:
    pattern = r'(' + cpu + r'.*?status = )"okay"'
    content = re.sub(pattern, r'\1"disabled"', content, count=1, flags=re.DOTALL)
with open('riscvpc.dts', 'w') as f:
    f.write(content)
```

**c. Fix the UART1 serial node.**

The original serial@20100000 node references a clock provider (clkcfg) that does not exist in Renode. This causes the 8250 driver to return EINVAL and fail to probe. Remove the clock reference, remove the wide-register properties that are incompatible with Renode's NS16550 model, and add a fixed clock-frequency:

```python
import re
with open('riscvpc.dts', 'r') as f:
    content = f.read()
old = re.search(r'(serial@20100000 \{.*?\});', content, re.DOTALL).group(0)
new = old
new = re.sub(r'compatible = "[^"]+";', 'compatible = "ns16550";', new)
new = re.sub(r'\t+reg-io-width = <[^>]+>;\n', '', new)
new = re.sub(r'\t+reg-shift = <[^>]+>;\n', '', new)
new = re.sub(r'\t+clocks = <[^>]+>;\n', '', new)
content = content.replace(old, new)
with open('riscvpc.dts', 'w') as f:
    f.write(content)
```

Then insert `clock-frequency = <150000000>;` into the serial@20100000 node (before the phandle line).

**d. Update the chosen node.**

Calculate the initramfs end address:

```bash
INITRD_SIZE=$(stat -c%s initramfs_renode.cpio.gz)
INITRD_END=$(python3 -c "print(hex(0x88000000 + $INITRD_SIZE))")
echo $INITRD_END
```

Then update the chosen node with correct bootargs and initrd location:

```python
import re
with open('riscvpc.dts', 'r') as f:
    content = f.read()
content = re.sub(
    r'(chosen \{[^}]*bootargs = "[^"]*";)',
    r'\1\n\t\tlinux,initrd-start = <0x88000000>;\n\t\tlinux,initrd-end = <INITRD_END_VALUE>;',
    content
)
# Update bootargs to correct console and enable keep_bootcon
content = re.sub(
    r'bootargs = "[^"]*"',
    'bootargs = "earlycon=ns16550a,mmio32,0x20100000,115200n8 console=ttyS1,115200 keep_bootcon init=/bin/sh"',
    content
)
with open('riscvpc.dts', 'w') as f:
    f.write(content)
```

Replace `INITRD_END_VALUE` with the hex value from the calculation above.

Recompile the DTB:

```bash
dtc -I dts -O dtb riscvpc.dts -o riscvpc_renode.dtb 2>/dev/null
```

Verify the changes compiled correctly:

```bash
strings riscvpc_renode.dtb | grep "keep_bootcon\|initrd\|bootargs"
```

### Step 6: Modify the Initramfs

Extract the original initramfs:

```bash
mkdir -p /tmp/initramfs_extract
cd /tmp/initramfs_extract
zcat ~/polarfire-soc-buildroot-sdk/work/initramfs.cpio.gz | cpio -idmv 2>/dev/null
```

Edit `etc/inittab` to run a shell on ttyS1. Replace the default getty line with:

```
ttyS1::respawn:-/bin/sh
```

Repack the initramfs:

```bash
cd /tmp/initramfs_extract
find . | cpio -o -H newc 2>/dev/null | gzip -9 > ~/polarfire-soc-buildroot-sdk/work/initramfs_renode.cpio.gz
```

### Step 7: Build OpenSBI with Linux Kernel as Payload

Build OpenSBI for the generic RISC-V platform. The Linux kernel binary is embedded as the payload at offset 0x200000, and the Renode-modified DTB is embedded so OpenSBI passes the correct DTB address to the kernel via register a1:

```bash
cd ~/polarfire-soc-buildroot-sdk/opensbi
make PLATFORM=generic \
  CROSS_COMPILE=~/polarfire-soc-buildroot-sdk/toolchain/bin/riscv64-unknown-linux-gnu- \
  FW_PAYLOAD_PATH=~/polarfire-soc-buildroot-sdk/work/vmlinux.bin \
  FW_PAYLOAD_OFFSET=0x200000 \
  FW_FDT_PATH=~/polarfire-soc-buildroot-sdk/work/riscvpc_renode.dtb \
  clean all
```

Output: `build/platform/generic/firmware/fw_payload.bin` (approximately 19 MB)

This single binary contains OpenSBI firmware, the Linux kernel at offset 0x200000, and the embedded DTB. Loading it at 0x80000000 in Renode is sufficient for OpenSBI to boot Linux directly without U-Boot.

### Step 8: Transfer Files to Windows

Serve build outputs from Ubuntu over HTTP:

```bash
cd ~/polarfire-soc-buildroot-sdk/opensbi/build/platform/generic/firmware
cp ~/polarfire-soc-buildroot-sdk/work/initramfs_renode.cpio.gz .
cp ~/polarfire-soc-buildroot-sdk/work/riscvpc_renode.dtb .
python3 -m http.server 8000
```

Access `http://<VM_IP>:8000` from the Windows browser and download:
- `fw_payload.bin` to `F:\Downloads\fw_payload.bin`
- `initramfs_renode.cpio.gz` to `F:\Downloads\initramfs_renode.cpio.gz`
- `riscvpc_renode.dtb` to `F:\Downloads\riscvpc_renode.dtb`

### Step 9: Create the Renode Platform Description

Save as `F:\FPGA\Task5\linux_boot\icicle.repl`:

```
cpu1: CPU.RiscV64 @ sysbus
    cpuType: "rv64gc"
    hartId: 1
    timeProvider: clint

cpu2: CPU.RiscV64 @ sysbus
    cpuType: "rv64gc"
    hartId: 2
    timeProvider: clint

cpu3: CPU.RiscV64 @ sysbus
    cpuType: "rv64gc"
    hartId: 3
    timeProvider: clint

cpu4: CPU.RiscV64 @ sysbus
    cpuType: "rv64gc"
    hartId: 4
    timeProvider: clint

clint: IRQControllers.CoreLevelInterruptor @ sysbus 0x02000000
    frequency: 1000000
    [0, 1] -> cpu1@[3, 7]

plic: IRQControllers.PlatformLevelInterruptController @ sysbus 0x0C000000
    0 -> cpu1@11
    numberOfSources: 186
    numberOfContexts: 2
    prioritiesEnabled: false

uart0: UART.NS16550 @ sysbus 0x20000000
    wideRegisters: true
    -> plic@90

uart1: UART.NS16550 @ sysbus 0x20100000
    wideRegisters: true
    -> plic@91

ddr: Memory.MappedMemory @ sysbus 0x80000000
    size: 0x40000000
```

Platform design notes:
- The E51 core (hartid=0) is not instantiated. The PolarFire SoC E51 runs HSS which is not needed for this emulation. Only the U54 application cores (hartid=1 through hartid=4) are present.
- Only cpu1 executes. cpu2, cpu3, cpu4 are present so OpenSBI can reference their hart IDs without crashing, but they are halted in the boot script.
- CLINT frequency is set to 1 MHz, matching the hardware specification. The CLINT timer interrupt and software interrupt lines are connected to cpu1.
- The PLIC has 186 interrupt sources and 2 contexts (M-mode and S-mode for hartid=1), matching the kernel's expectation for a single-hart system.
- DDR is mapped at 0x80000000 with 1 GB size, matching the corrected memory node in the DTB.

### Step 10: Create the Renode Boot Script

Save as `F:\FPGA\Task5\linux_boot\boot.resc`:

```
mach create "icicle-kit"
machine LoadPlatformDescription @F:\FPGA\Task5\linux_boot\icicle.repl

showAnalyzer sysbus.uart0
showAnalyzer sysbus.uart1

sysbus.cpu2 IsHalted true
sysbus.cpu3 IsHalted true
sysbus.cpu4 IsHalted true

sysbus LoadBinary @F:\Downloads\fw_payload.bin 0x80000000
sysbus LoadFdt @F:\Downloads\riscvpc_renode.dtb 0x82200000 "" true
sysbus LoadBinary @F:\Downloads\initramfs_renode.cpio.gz 0x88000000

sysbus.cpu1 PC 0x80000000

start
```

Boot script notes:
- `fw_payload.bin` is loaded at 0x80000000, which is OpenSBI's base address.
- The DTB is loaded at 0x82200000 using Renode's LoadFdt command. This address matches the `Domain0 Next Arg1` value that OpenSBI reports, which is the address OpenSBI passes to the kernel in register a1.
- The initramfs is loaded at 0x88000000, matching the `linux,initrd-start` value in the chosen node of the DTB.
- cpu1's PC is set to 0x80000000 (OpenSBI entry point). cpu1 has hartid=1 and becomes the boot hart.
- cpu2, cpu3, cpu4 are halted so they do not compete to become the boot hart or interfere with OpenSBI cold boot initialization.

### Step 11: Run Renode

```
"F:\SoftConsole\renode\bin\Renode.exe" --plain -e "include @F:\FPGA\Task5\linux_boot\boot.resc"
```

Two UART terminal windows open automatically. OpenSBI output appears on uart0. The Linux kernel boot log appears on uart1.

---

## Expected Boot Output

**uart0 (OpenSBI):**

```
OpenSBI v0.9
Platform Name             : Microchip PolarFire-SoC Icicle Kit
Platform Features         : timer,mfdeleg
Platform HART Count       : 5
Firmware Base             : 0x80000000
Firmware Size             : 148 KB
Runtime SBI Version       : 0.2
Domain0 Boot HART         : 1
Domain0 Next Address      : 0x0000000080200000
Domain0 Next Arg1         : 0x0000000082200000
Domain0 Next Mode         : S-mode
Boot HART ID              : 1
```

**uart1 (Linux kernel, abbreviated):**

```
Linux version 5.15.68-linux4microchip+fpga-2022.09
Machine model: Microchip PolarFire-SoC Icicle Kit
earlycon: ns16550a0 at MMIO32 0x0000000020100000
printk: bootconsole [ns16550a0] enabled
printk: debug: skip boot console de-registration.
Memory: ~980MB available
smp: Brought up 1 node, 1 CPU
devtmpfs: initialized
Unpacking initramfs...
Serial: 8250/16550 driver, 4 ports, IRQ sharing disabled
20100000.serial: ttyS1 at MMIO 0x20100000 (irq = 6, base_baud = 9375000) is a 16550
printk: console [ttyS1] enabled
mpfs_dma_proxy mpfs-dma-proxy: proxy dma 4 channels initialized
```

After the final driver initialization line, the kernel enters the idle loop and the BusyBox init process runs. The CPU program counter can be verified via the Renode monitor:

```
sysbus.cpu1 PC
```

A value in the range 0xffffffff80000000 confirms the kernel is running normally in the idle loop.

---

## Key Design Decisions and Lessons

**Why fw_payload embeds the DTB rather than relying on LoadFdt alone.**

OpenSBI passes the DTB address to the kernel in register a1. When FW_FDT_PATH is specified at build time, OpenSBI embeds the DTB and guarantees the correct address is used. If only LoadFdt is used in Renode without the DTB embedded in fw_payload, OpenSBI reads address 0x0 looking for the device tree magic number, finds nothing, and the kernel panics early in boot.

**Why hartid=1 is the boot hart.**

The PolarFire SoC E51 core is hartid=0. The four U54 application cores are hartid=1 through hartid=4. Linux is designed to run on the U54 cores. By not instantiating hartid=0 in Renode and setting only cpu1's PC before starting, OpenSBI elects hartid=1 as the cold boot hart, which is the correct behavior for this platform.

**Why cpu@2 through cpu@4 are disabled in the DTB.**

With a single running hart in Renode, Linux SMP initialization hangs indefinitely if those CPUs are marked as available (status = "okay") in the DTB. The kernel issues SBI HSM (Hart State Management) calls to start secondary CPUs, but the halted Renode cores never respond. Marking cpu@2 through cpu@4 as disabled tells the kernel they do not exist, allowing single-CPU boot to complete without hanging.

**Why keep_bootcon is required in the bootargs.**

The Linux 8250/16550 driver probes successfully and registers ttyS1, but interactive use requires interrupt-driven I/O which does not work in this Renode configuration. Without keep_bootcon, the kernel disables the earlycon bootconsole when it switches to ttyS1, eliminating all visible output. With keep_bootcon, the earlycon remains active in parallel, preserving the kernel boot log in the uart1 window.

**Why interactive shell input and output do not work.**

The 8250/16550 kernel driver uses interrupts for both transmit and receive. For transmit, the driver waits for the Transmit Holding Register Empty (THRE) interrupt before writing the next character. For receive, characters are delivered via the Received Data Available interrupt. Both require the PLIC to correctly deliver interrupt source 91 (UART1) to the running hart. While the PLIC is wired in the platform description, Renode's NS16550 model does not generate these interrupts in a way that the 8250 driver can consume for terminal I/O. This is a limitation of the emulation, not of the Linux image itself.

---

## Tools Used

- Buildroot: Meta-build system used to cross-compile the complete software stack from source, including the RISC-V toolchain, Linux kernel, OpenSBI, U-Boot, BusyBox, and userspace libraries.
- OpenSBI (open source): RISC-V Supervisor Binary Interface reference firmware. Provides M-mode runtime services to S-mode OS.
- Linux kernel 5.15.68 (Microchip fork): The operating system kernel, configured for the PolarFire SoC.
- BusyBox: Provides the init process, shell, and core userspace utilities within the initramfs.
- dtc (Device Tree Compiler): Used to decompile the Buildroot-generated DTB into editable DTS source and recompile it after modifications.
- cpio and gzip: Used to extract and repack the BusyBox initramfs archive.
- Python 3: Used for reliable multi-line and regex-based DTS text manipulation.
- Renode (Antmicro): Open source hardware emulator. Provides RISC-V CPU cores, CLINT, PLIC, NS16550 UART, and memory peripheral models.
- SoftConsole (Microchip): IDE for PolarFire SoC development, bundled with Renode.
- GCC RISC-V cross-toolchain (riscv64-unknown-linux-gnu): Built by Buildroot, used by OpenSBI Makefile for compilation.
- VMware Workstation: Virtualization platform running the Ubuntu 22.04 build environment on Windows 11.
