# Yocto Kernel & Bootloader Recipes — Complete Reference Guide

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [linux-yocto — The Reference Kernel Recipe Approach](#2-linux-yocto--the-reference-kernel-recipe-approach)
3. [kernel.bbclass and kernel-yocto.bbclass](#3-kernelbbclass-and-kernel-yoctobbclass)
4. [Kernel Configuration — defconfig vs Fragments](#4-kernel-configuration--defconfig-vs-fragments)
5. [The kernel-yocto "Features" and SCC Files](#5-the-kernel-yocto-features-and-scc-files)
6. [Writing a Kernel bbappend](#6-writing-a-kernel-bbappend)
7. [Using a Vendor/Custom Kernel Tree Instead of linux-yocto](#7-using-a-vendorcustom-kernel-tree-instead-of-linux-yocto)
8. [Kernel Modules — Out-of-Tree Module Recipes](#8-kernel-modules--out-of-tree-module-recipes)
9. [Device Tree Handling](#9-device-tree-handling)
10. [Kernel Debugging Tools](#10-kernel-debugging-tools)
11. [virtual/kernel and Kernel Provider Selection](#11-virtualkernel-and-kernel-provider-selection)
12. [U-Boot Recipe Structure](#12-u-boot-recipe-structure)
13. [U-Boot Configuration and defconfig](#13-u-boot-configuration-and-defconfig)
14. [U-Boot Environment and Boot Scripts](#14-u-boot-environment-and-boot-scripts)
15. [virtual/bootloader and Alternative Bootloaders](#15-virtualbootloader-and-alternative-bootloaders)
16. [Kernel and Bootloader Deployment (do_deploy)](#16-kernel-and-bootloader-deployment-do_deploy)
17. [Complete Worked Example — Custom BSP Kernel Setup](#17-complete-worked-example--custom-bsp-kernel-setup)
18. [Common Pitfalls and Debugging](#18-common-pitfalls-and-debugging)
19. [Interview Questions and Answers](#19-interview-questions-and-answers)

---

## 1. Introduction

Kernel and bootloader recipes are architecturally similar to normal recipes (they're still `.bb` files producing packages/deployable artifacts through the standard fetch/configure/compile/install pipeline) but carry substantial **domain-specific tooling** layered on top, because kernel and bootloader configuration management has unique requirements that don't map cleanly onto generic Autotools/CMake patterns: kernel `.config` files have tens of thousands of interdependent options, device trees need board-specific source files and compilation, and bootloaders need to produce raw, specially-formatted binary images (not standard ELF executables installed to `/usr/bin`) that get written to specific flash/disk offsets.

---

## 2. linux-yocto — The Reference Kernel Recipe Approach

`linux-yocto` is the Yocto Project's own maintained kernel recipe family, built around a key design principle: rather than maintaining one full kernel source tree per board/vendor (which would mean re-merging upstream security patches across dozens of divergent trees), `linux-yocto` uses a **single shared kernel git repository with a tree of merged feature branches**, and individual machine/BSP configurations are expressed as small, layered **configuration fragments** and **patch sets** applied on top of a common base — managed by a tool called `kernel-yocto` (the `kgit`/`kgit-meta` tooling, anciently referred to as "kern-tools").

```bitbake
# A linux-yocto recipe is typically very short, since most of the actual
# logic lives in kernel.bbclass/kernel-yocto.bbclass:

KBRANCH = "v6.6/standard/base"
SRCREV_machine = "abc123..."
SRC_URI = "git://git.yoctoproject.org/linux-yocto.git;branch=${KBRANCH};protocol=https"

LINUX_VERSION = "6.6.21"
LINUX_VERSION_EXTENSION = "-myproduct"

inherit kernel
require recipes-kernel/linux/linux-yocto.inc

COMPATIBLE_MACHINE = "qemux86-64|my-board"
```

### Why This Architecture Exists

A traditional approach (one full forked kernel source tree per board) means every upstream kernel CVE fix or stability patch has to be manually cherry-picked into N separate trees. The linux-yocto approach keeps ONE shared upstream-tracking tree, with board/feature differences expressed as small, independently-maintainable fragments (config fragments, small patch sets) that get combined at build time — meaning a single upstream kernel update (`SRCREV` bump) can propagate a security fix to every board's build simultaneously, with board-specific customizations re-applied automatically on top.

---

## 3. kernel.bbclass and kernel-yocto.bbclass

### kernel.bbclass — Generic Kernel Build Logic

Provides the core mechanics applicable to ANY Linux kernel recipe (not just linux-yocto): invoking the kernel's own Makefile-based build system correctly for cross-compilation, handling `.config` generation, building modules, and packaging the result.

```bitbake
inherit kernel

# Key variables kernel.bbclass introduces:
KERNEL_IMAGETYPE = "zImage"          # or "bzImage", "Image", "uImage", etc.
KERNEL_DEVICETREE = "my-board.dtb"
KERNEL_EXTRA_ARGS = ""
```

### kernel-yocto.bbclass — linux-yocto-Specific Tooling

Layers on TOP of `kernel.bbclass` (used specifically by `linux-yocto`-family recipes) to add the SCC/feature-branch configuration merging mechanism described in Section 5.

```bitbake
inherit kernel kernel-yocto
```

### Standard Kernel Recipe Tasks

```
do_validate_branches   ← (kernel-yocto specific) verify the requested KBRANCH/SRCREV exist
do_kernel_metadata      ← (kernel-yocto specific) gather config fragments and feature descriptions
do_kernel_configme       ← merge all config fragments into a final .config
do_configure             ← standard task name, delegates to do_kernel_configme machinery
do_compile               ← runs the kernel's own `make` build (vmlinux, modules, dtbs)
do_compile_kernelmodules ← separate task specifically for building loadable modules
do_install                ← installs kernel image, modules, and (if applicable) DTBs into ${D}
do_deploy                  ← copies the final kernel image/DTB into DEPLOY_DIR_IMAGE for the image build to consume
do_sizecheck                ← verifies the built kernel image doesn't exceed a configured maximum size (important for bootloaders with fixed-size kernel partitions)
do_kernel_link_images        ← creates conveniently-named symlinks to the latest kernel build output
```

---

## 4. Kernel Configuration — defconfig vs Fragments

### Traditional defconfig Approach (works with any kernel recipe, including non-linux-yocto)

```bitbake
SRC_URI += "file://defconfig"

do_configure:prepend() {
    cp ${WORKDIR}/defconfig ${B}/.config
}
```

A `defconfig` is simply a complete (or near-complete, kernel `make oldconfig` fills gaps with defaults) kernel `.config` file, generated on a reference build via `make savedefconfig` and checked into the BSP layer.

### linux-yocto Fragment Approach (preferred for linux-yocto-based recipes)

Instead of one large, monolithic, hard-to-review defconfig file, linux-yocto encourages small, focused **configuration fragments** — plain text files containing only the specific `CONFIG_*` options relevant to one feature/concern, which get programmatically MERGED together at build time:

```
# my-board-usb.cfg
CONFIG_USB=y
CONFIG_USB_EHCI_HCD=y
CONFIG_USB_OHCI_HCD=y

# my-board-display.cfg
CONFIG_DRM=y
CONFIG_DRM_PANEL=y
CONFIG_FB=y
```

```bitbake
SRC_URI += "file://my-board-usb.cfg \
            file://my-board-display.cfg \
            "
```

`kernel-yocto.bbclass` automatically discovers `.cfg` files referenced in `SRC_URI` and merges them (along with the kernel's own base defconfig for the target architecture) using the kernel's own `merge_config.sh` mechanism, then runs `make oldconfig`/`olddefconfig` to resolve any newly-exposed dependent options to their defaults.

### Inspecting the Final Merged Configuration

```bash
bitbake -c menuconfig virtual/kernel
# Opens the standard kernel Kconfig ncurses UI, pre-loaded with the
# fully-merged configuration — lets you interactively explore/adjust,
# then save changes back out as a new fragment via:

bitbake -c diffconfig virtual/kernel
# Shows exactly what changed between the original merged config and
# whatever you adjusted in menuconfig — output is itself a valid .cfg
# fragment you can save directly into your layer
```

This `menuconfig` → `diffconfig` workflow is the standard, recommended way to interactively discover and create new configuration fragments rather than hand-writing `CONFIG_*` lines from memory.

---

## 5. The kernel-yocto "Features" and SCC Files

For more substantial kernel customization beyond simple config options — adding actual source code patches bundled together with their relevant config options as one coherent "feature" — linux-yocto uses **SCC (Series Configuration Control)** files:

```
# my-feature.scc
define KFEATURE_DESCRIPTION "Add custom sensor driver support"

patch 0001-add-my-sensor-driver.patch
patch 0002-fix-sensor-driver-dts-binding.patch

kconf non-hardware my-sensor.cfg
```

```bitbake
SRC_URI += "file://my-feature.scc \
            file://0001-add-my-sensor-driver.patch \
            file://0002-fix-sensor-driver-dts-binding.patch \
            file://my-sensor.cfg \
            "

KERNEL_FEATURES:append = " features/my-feature/my-feature.scc"
```

This bundles patches and their corresponding config requirements as one named, reusable, documented "feature" — useful when a feature genuinely needs both code changes AND config changes together, and you want that pairing to be a single coherent, reviewable unit rather than two separately-tracked things that could drift out of sync.

---

## 6. Writing a Kernel bbappend

The overwhelming majority of real-world BSP kernel customization happens via `.bbappend`, not by writing an entirely new kernel recipe from scratch:

```bitbake
# meta-mylayer/recipes-kernel/linux/linux-yocto_%.bbappend

FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

SRC_URI += "file://my-board-usb.cfg \
            file://my-board-display.cfg \
            file://0001-fix-my-board-quirk.patch \
            "

COMPATIBLE_MACHINE:my-board = "my-board"

# Pin a specific kernel branch/revision for this specific machine
KBRANCH:my-board = "v6.6/standard/my-board"
SRCREV_machine:my-board = "9f3c2a1b8e7d6f5a..."
```

### Machine-Specific Configuration via the Machine Config File

```bitbake
# conf/machine/my-board.conf (in the BSP layer)

PREFERRED_PROVIDER_virtual/kernel = "linux-yocto"
PREFERRED_VERSION_linux-yocto = "6.6%"

KERNEL_IMAGETYPE = "zImage"
KERNEL_DEVICETREE = "my-vendor/my-board.dtb"

MACHINE_FEATURES = "usbhost usbgadget alsa bluetooth wifi"
```

---

## 7. Using a Vendor/Custom Kernel Tree Instead of linux-yocto

Many real BSPs (especially for newer/specialized SoCs) instead use the SoC vendor's own kernel fork (e.g., NXP's, TI's, or a vendor-supplied downstream tree with hardware-specific patches not yet upstreamed) rather than linux-yocto's curated tree. This is still entirely normal and well-supported — it just means a more conventional recipe structure, closer to a generic Autotools-style recipe than the SCC/feature-fragment machinery.

```bitbake
SUMMARY = "Linux kernel for My Vendor SoC"
LICENSE = "GPL-2.0-only"
LIC_FILES_CHKSUM = "file://COPYING;md5=6bc538ed5bd9a7fc9398086aedcd7e46"

SRC_URI = "git://github.com/myvendor/linux.git;protocol=https;branch=myvendor-6.6 \
           file://defconfig \
           "
SRCREV = "abc123..."

PV = "6.6+git${SRCPV}"

S = "${WORKDIR}/git"

inherit kernel

KERNEL_IMAGETYPE = "Image"
COMPATIBLE_MACHINE = "my-vendor-board"

do_configure:prepend() {
    cp ${WORKDIR}/defconfig ${B}/.config
}
```

This still inherits `kernel.bbclass` (for the generic cross-compilation/install/packaging machinery) but skips `kernel-yocto.bbclass` (since there's no SCC/feature-fragment merging needed — a plain defconfig is used instead).

---

## 8. Kernel Modules — Out-of-Tree Module Recipes

A kernel module that's NOT part of the main kernel tree (e.g., a vendor's proprietary or just-not-yet-upstreamed driver) gets its own separate recipe, inheriting `module.bbclass`:

```bitbake
SUMMARY = "Custom sensor kernel module"
LICENSE = "GPL-2.0-only"
LIC_FILES_CHKSUM = "file://COPYING;md5=..."

inherit module

SRC_URI = "git://github.com/example/my-sensor-module.git;protocol=https;branch=main"
SRCREV = "abc123..."
PV = "1.0+git${SRCPV}"

S = "${WORKDIR}/git"

# Ensures this module recipe builds against the EXACT matching kernel build
# (correct kernel headers/build system version) — critical, since kernel
# modules are tightly ABI-coupled to the exact kernel they're built against
RPROVIDES:${PN} += "kernel-module-my-sensor"
```

`module.bbclass` automatically:
- Adds the necessary `DEPENDS`/`do_compile[depends]` on `virtual/kernel:do_shared_workdir` (ensuring the matching kernel's build tree/headers are available)
- Invokes the kernel's standard out-of-tree module build process (`make -C <kernel-build-dir> M=<module-source-dir> modules`)
- Installs the resulting `.ko` file(s) into the correct versioned modules directory (`/lib/modules/<kernel-version>/extra/`)

### module-base.bbclass / modules.bbclass for Module Auto-Loading

```bitbake
inherit module

KERNEL_MODULE_AUTOLOAD += "my-sensor"
# Ensures the module is automatically loaded at boot via /etc/modules-load.d/
```

---

## 9. Device Tree Handling

Device trees (`.dts`/`.dtsi` source, compiled to `.dtb`) describe the hardware layout to the kernel at boot — essential on ARM and many other embedded architectures lacking x86-style runtime hardware discovery (ACPI/PCI enumeration).

### Device Tree Source Location

```
linux-source-tree/arch/arm64/boot/dts/my-vendor/my-board.dts
```

Device trees are typically built as part of the kernel recipe itself (`make dtbs` is a standard target the kernel's own build system provides) rather than a separate recipe:

```bitbake
KERNEL_DEVICETREE = "my-vendor/my-board.dtb my-vendor/my-board-variant2.dtb"

# Add a custom/patched device tree source via the standard kernel patch mechanism
SRC_URI += "file://0001-add-my-board-devicetree.patch"
```

### Out-of-Tree / Externally Maintained Device Trees

Some BSP layers maintain device tree source files SEPARATELY from the main kernel tree (useful when iterating on hardware bring-up faster than the kernel recipe's own update cadence allows):

```bitbake
inherit devicetree

SRC_URI = "file://my-board.dts file://my-board-overlay.dtso"

COMPATIBLE_MACHINE = "my-board"
```

`devicetree.bbclass` (a more recent addition to oe-core) provides standalone device-tree compilation (via `dtc`, the device tree compiler) independent of needing to rebuild the entire kernel — particularly useful for device tree OVERLAYS (small, separately-loadable hardware description fragments, common on platforms like Raspberry Pi for HAT/add-on board support) that genuinely are logically separate, frequently-iterated artifacts.

---

## 10. Kernel Debugging Tools

```bitbake
# Build the kernel with debug symbols retained (NOT stripped) for use with
# gdb/crash-tool against a vmlinux image
KERNEL_DEBUG_TIMESTAMPS = "1"

IMAGE_INSTALL:append = " kernel-dbg"

# Enable kernel config options useful for debugging directly via a fragment
SRC_URI += "file://debug.cfg"
```

```
# debug.cfg
CONFIG_DEBUG_KERNEL=y
CONFIG_DEBUG_INFO=y
CONFIG_MAGIC_SYSRQ=y
CONFIG_KGDB=y
CONFIG_KGDB_SERIAL_CONSOLE=y
```

```bash
# Useful devtool integration for kernel iteration (same pattern as any recipe)
devtool modify virtual/kernel
cd build/workspace/sources/linux-yocto
# edit kernel source directly
devtool build virtual/kernel
devtool deploy-target virtual/kernel root@192.168.1.50
# reboot target to test the new kernel build
```

---

## 11. virtual/kernel and Kernel Provider Selection

```bitbake
# In a machine.conf file:
PREFERRED_PROVIDER_virtual/kernel = "linux-yocto"
PREFERRED_VERSION_linux-yocto = "6.6%"

# OR, for a vendor kernel:
PREFERRED_PROVIDER_virtual/kernel = "linux-myvendor"
```

Every recipe and image that needs "the kernel" references the abstract `virtual/kernel` name rather than hard-coding a specific recipe — this is the same `virtual/*` indirection mechanism covered in the BitBake document, letting a BSP layer swap in its preferred actual kernel recipe without modifying every dependent recipe/image.

```bash
# Confirm which actual recipe is currently satisfying virtual/kernel
bitbake-layers show-recipes virtual/kernel
bitbake -e virtual/kernel | grep ^PN=
```

---

## 12. U-Boot Recipe Structure

U-Boot (Das U-Boot) is the most widely used open-source bootloader in embedded Linux. Its recipe structure closely parallels the kernel's — a configuration step, a compile step, and deployment of specially-formatted output binaries (not normal ELF executables).

```bitbake
# u-boot_2024.01.bb (simplified, illustrating structure — real recipes
# typically use a shared .inc plus this kind of version-specific file)

SUMMARY = "Das U-Boot bootloader"
LICENSE = "GPL-2.0-only"
LIC_FILES_CHKSUM = "file://Licenses/README;md5=..."

SRC_URI = "git://source.denx.de/u-boot/u-boot.git;protocol=https;branch=master"
SRCREV = "abc123..."

S = "${WORKDIR}/git"

inherit uboot-config deploy

DEPENDS += "dtc-native bc-native python3-native"
```

### uboot-config.bbclass

Provides standardized `UBOOT_MACHINE`/`UBOOT_CONFIG` variable handling, mapping a machine name to U-Boot's own `<board>_defconfig` naming convention:

```bitbake
UBOOT_MACHINE = "my_board_defconfig"

# When a single board has multiple valid configurations (e.g., SPI-NOR boot
# vs eMMC boot variants of the same hardware), UBOOT_CONFIG provides a more
# flexible mapping:
UBOOT_CONFIG ??= "sd"
UBOOT_CONFIG[sd] = "my_board_sd_defconfig"
UBOOT_CONFIG[emmc] = "my_board_emmc_defconfig"
```

---

## 13. U-Boot Configuration and defconfig

Similar in concept to the Linux kernel's Kconfig system (U-Boot actually reuses much of the same Kconfig infrastructure), U-Boot is configured via a `<board>_defconfig` file selected at build time.

```bitbake
do_configure() {
    oe_runmake ${UBOOT_MACHINE}
}
```

### Customizing U-Boot Configuration via bbappend

```bitbake
# meta-mylayer/recipes-bsp/u-boot/u-boot_%.bbappend

FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

SRC_URI += "file://0001-my-board-fix.patch \
            file://my-board.cfg \
            "

do_configure:append() {
    cat ${WORKDIR}/my-board.cfg >> ${B}/.config
    oe_runmake -C ${B} olddefconfig
}
```

### U-Boot Build Output

```bitbake
KERNEL_IMAGETYPE = "zImage"

# U-Boot itself produces architecture/platform-specific binary output:
#   u-boot.bin       — raw binary, often the actual flashable artifact
#   u-boot.img       — with a U-Boot-specific header (legacy image format)
#   u-boot.itb       — FIT (Flattened Image Tree) format, modern standard,
#                       can bundle kernel+dtb+initramfs+signatures together
#   MLO / SPL        — Secondary Program Loader, a tiny first-stage bootloader
#                       (common on TI/many ARM SoCs needing a 2-stage boot)
```

---

## 14. U-Boot Environment and Boot Scripts

### Default Environment Variables

```bitbake
# Often supplied as a separate text file processed by mkenvimage/mkimage
SRC_URI += "file://uEnv.txt"

UBOOT_ENV = "uEnv"
UBOOT_ENV_SUFFIX = "env"
```

```
# uEnv.txt example content
bootargs=console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait rw
bootcmd=load mmc 0:1 ${kernel_addr_r} zImage; load mmc 0:1 ${fdt_addr_r} my-board.dtb; bootz ${kernel_addr_r} - ${fdt_addr_r}
```

### Boot Scripts (extlinux.conf, boot.scr)

```bitbake
inherit extlinuxboot
```

`extlinuxboot.bbclass` generates a standard `extlinux.conf` boot configuration file (compatible with U-Boot's `distro_bootcmd` generic boot flow, which many modern BSPs use instead of fully custom `bootcmd` scripting), automatically populated with the correct kernel image name, device tree name, and root filesystem boot arguments based on the image recipe's own configuration.

```bitbake
# Generated /boot/extlinux/extlinux.conf typically looks like:
# LABEL Yocto
#     KERNEL /zImage
#     FDT /my-board.dtb
#     APPEND root=/dev/mmcblk0p2 rootwait rw console=ttyS0,115200
```

---

## 15. virtual/bootloader and Alternative Bootloaders

```bitbake
PREFERRED_PROVIDER_virtual/bootloader = "u-boot"
```

Just like `virtual/kernel`, the bootloader is referenced abstractly. Common alternatives that a machine configuration might select instead of U-Boot:

| Bootloader   | Common Use Case                                                  |
|----------------|------------------------------------------------------------------|
| `u-boot`       | The overwhelming majority of embedded ARM/RISC-V/PowerPC boards   |
| `barebox`       | An alternative open-source bootloader, popular in some German/EU embedded ecosystems|
| `grub`          | x86/x86_64 platforms, especially when UEFI boot is required        |
| `systemd-boot`  | Simpler UEFI boot manager, an alternative to grub on x86 UEFI systems|
| `rpi-bootfiles` | Raspberry Pi's own proprietary GPU-firmware-driven boot process (not a traditional bootloader recipe in the U-Boot sense, but fills the same conceptual role)|

```bitbake
# x86_64 machine using grub instead of u-boot:
PREFERRED_PROVIDER_virtual/bootloader = "grub"
EFI_PROVIDER = "grub-efi"
```

---

## 16. Kernel and Bootloader Deployment (do_deploy)

Both kernel and bootloader recipes use `deploy.bbclass`'s `do_deploy` task, which copies final build artifacts into the shared `DEPLOY_DIR_IMAGE` directory — the staging area an image recipe's `do_image_wic` task later reads from to assemble the final bootable disk image.

```bitbake
inherit deploy

do_deploy() {
    install -d ${DEPLOYDIR}
    install -m 0644 ${B}/u-boot.bin ${DEPLOYDIR}/u-boot-${MACHINE}.bin
    ln -sf u-boot-${MACHINE}.bin ${DEPLOYDIR}/u-boot.bin
}
addtask deploy after do_compile before do_build
```

```bash
# Final deployed artifacts available for imaging/manual flashing:
ls tmp/deploy/images/<machine>/
zImage
zImage-my-board.dtb
u-boot.bin
modules-<machine>.tgz
```

---

## 17. Complete Worked Example — Custom BSP Kernel Setup

```bitbake
# meta-my-bsp/recipes-kernel/linux/linux-yocto_%.bbappend

FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

COMPATIBLE_MACHINE:my-board = "my-board"
KBRANCH:my-board = "v6.6/standard/my-board"
SRCREV_machine:my-board = "9f3c2a1b8e7d6f5a4c3b2a1908765432fedcba98"

KMACHINE:my-board = "my-board"

SRC_URI:append:my-board = " \
    file://0001-add-my-board-devicetree.patch \
    file://0002-add-my-sensor-driver.patch \
    file://my-board-base.cfg \
    file://my-board-usb.cfg \
    file://my-board-display.cfg \
    file://my-feature.scc \
    "

KERNEL_FEATURES:append:my-board = " features/my-feature/my-feature.scc"
```

```bitbake
# meta-my-bsp/conf/machine/my-board.conf

#@TYPE: Machine
#@NAME: My Board
#@DESCRIPTION: Machine configuration for My Custom Board (ARM Cortex-A53)

require conf/machine/include/arm/armv8a/tune-cortexa53.inc

MACHINEOVERRIDES =. "my-board:"

PREFERRED_PROVIDER_virtual/kernel = "linux-yocto"
PREFERRED_VERSION_linux-yocto = "6.6%"

PREFERRED_PROVIDER_virtual/bootloader = "u-boot"
PREFERRED_VERSION_u-boot = "2024.01%"

KERNEL_IMAGETYPE = "Image"
KERNEL_DEVICETREE = "my-vendor/my-board.dtb"

UBOOT_MACHINE = "my_board_defconfig"

MACHINE_FEATURES = "usbhost usbgadget alsa bluetooth wifi vfat ext2"

SERIAL_CONSOLES = "115200;ttyS0"

WKS_FILE = "my-board.wks"
IMAGE_FSTYPES += "wic wic.bmap"

MACHINE_EXTRA_RRECOMMENDS = "kernel-modules"
```

```bitbake
# meta-my-bsp/recipes-bsp/u-boot/u-boot_%.bbappend

FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

SRC_URI:append:my-board = " file://0001-my-board-pinmux-fix.patch"
```

---

## 18. Common Pitfalls and Debugging

### 1. Kernel config fragment silently not applying any effect
The most common cause: the option the fragment sets DEPENDS on another option that isn't enabled, and `make olddefconfig` (run automatically after fragment merging) silently resolves the dependent option back to its default (often `n`/disabled) rather than erroring. Always verify with `bitbake -c menuconfig virtual/kernel` that the option actually landed as expected in the final merged `.config`, and check for dependency warnings during `do_kernel_configme`.

### 2. "ERROR: no recipes available for virtual/kernel"
Means no kernel recipe in any active layer declares `COMPATIBLE_MACHINE` matching your current `MACHINE`, or `PREFERRED_PROVIDER_virtual/kernel` points at a recipe name that doesn't exist/isn't enabled. Check `COMPATIBLE_MACHINE` regex in both the kernel recipe and any bbappends.

### 3. Kernel builds fine but board doesn't boot — wrong device tree
Verify `KERNEL_DEVICETREE` exactly matches the actual `.dts` filename (including any vendor subdirectory prefix) that the kernel build actually produced — check `tmp/deploy/images/<machine>/` for the actual `.dtb` filename generated and compare.

### 4. Out-of-tree kernel module fails to load: "invalid module format"
Almost always a kernel-ABI version mismatch — the module was built against a different exact kernel build/config than what's actually running on the target. Verify the module recipe's `DEPENDS`/build-time linkage to `virtual/kernel` is correctly resolving to the SAME kernel build as what's deployed, and that `uname -r` on the target exactly matches the modules directory the module was installed into.

### 5. U-Boot defconfig changes via bbappend don't take effect
Remember U-Boot's `.config` (like the kernel's) is regenerated from the named `<board>_defconfig` file at `do_configure` time — appending raw config lines via `do_configure:append()` AFTER the base `oe_runmake ${UBOOT_MACHINE}` step (as shown in Section 13) is necessary, and you typically need to follow it with `oe_runmake olddefconfig` to resolve dependent options correctly, exactly analogous to kernel fragment merging.

### 6. Device tree changes not picked up after editing a .dts file directly during development
If using `devtool modify virtual/kernel` for iteration, remember device trees are normally built as PART of the kernel's own `make dtbs`/`make` invocation — a `devtool build` should pick up `.dts` changes automatically through the kernel's normal incremental Makefile-based rebuild, but verify the specific `.dtb` you expect is actually listed in `KERNEL_DEVICETREE` and gets regenerated (check the build log timestamp on the output `.dtb` file).

### 7. SRCREV pinned to a commit that doesn't exist on the specified KBRANCH
linux-yocto's `do_validate_branches` task specifically checks that the given `SRCREV` actually exists as a reachable commit on the specified `KBRANCH` — a common mistake when manually updating one without correspondingly checking/updating the other after rebasing a custom kernel branch.

---

## 19. Interview Questions and Answers

**Q1: Why does linux-yocto use small configuration fragments and SCC feature files instead of one large defconfig per board, and what specific problem does this architecture solve?**

**A:** The core problem is maintainability at scale across many boards sharing a common kernel codebase. With one large, monolithic defconfig per board, every board's full configuration is essentially an independent, opaque blob of hundreds or thousands of `CONFIG_*` lines — when the kernel itself is updated (a new upstream version, or a CVE security patch), there's no structural way to know which specific config lines in each board's giant defconfig relate to which specific concern, making it hard to confidently merge upstream changes or understand what's actually board-specific versus copy-pasted boilerplate. linux-yocto's fragment approach instead expresses configuration as small, focused, individually-named files (`usb.cfg`, `display.cfg`, a custom `my-feature.scc` bundling a patch with its required config) that get programmatically MERGED at build time using the kernel's own standard `merge_config.sh`/`olddefconfig` machinery. This means: (1) a board's true customization footprint is small and reviewable — you can see exactly which fragments a board pulls in; (2) fragments are reusable across multiple boards that share a common need (e.g., the same USB controller fragment used by five different boards from the same vendor); (3) upstream kernel updates (a `SRCREV` bump on the shared base branch) automatically carry forward correctly merged with all the board-specific fragments, since the fragments express DELTAS from a common base rather than complete, independently-drifting configurations.

---

**Q2: Explain the relationship between kernel.bbclass and kernel-yocto.bbclass — why are these two separate classes rather than one combined class?**

**A:** `kernel.bbclass` provides the GENERIC mechanics that apply to building ANY Linux kernel via BitBake, regardless of which specific kernel source tree or configuration methodology is used — correctly invoking the kernel's own cross-compilation-aware Makefile build system, handling the standard `do_compile`/`do_install`/`do_deploy` task structure, generating kernel and kernel-module packages, and so on. This generic logic is exactly as applicable to a vendor's downstream kernel fork (Section 7's example) as it is to linux-yocto. `kernel-yocto.bbclass` is layered ON TOP of `kernel.bbclass` specifically to add the SCC-file/configuration-fragment merging machinery, branch validation (`do_validate_branches` checking a given `SRCREV` actually exists on the declared `KBRANCH`), and other tooling that is SPECIFIC to the linux-yocto project's particular shared-tree-plus-fragments methodology — none of which makes sense or is needed for a recipe building a vendor's conventional, already-complete kernel fork with a traditional single defconfig. Separating these lets `kernel.bbclass` remain a clean, minimal, reusable foundation that ANY kernel recipe can build on, while `kernel-yocto.bbclass` opts a recipe into the additional, more opinionated linux-yocto-specific tooling only when that recipe actually wants and needs it.

---

**Q3: What is the practical difference between a kernel configuration fragment (.cfg) and an SCC feature file, and when would you use one over the other?**

**A:** A plain `.cfg` fragment contains ONLY `CONFIG_*` Kconfig option settings — no source code changes, just configuration. It's the right tool when the board/feature genuinely only needs a configuration change with no corresponding kernel source modification — for example, simply enabling a driver that already exists in the mainline kernel tree but isn't enabled by the base defconfig (a common USB controller, a standard I2C bus driver already upstream). An SCC (Series Configuration Control) file is a higher-level construct that can BUNDLE one or more actual source code patches together with their associated configuration requirements as one named, documented, reusable "feature" — appropriate when a feature genuinely requires both code changes (a new driver not yet in the kernel tree, a bug fix patch) AND specific config options to actually activate that new code, and you want this pairing tracked and applied as a single coherent unit rather than as two separately-listed, easy-to-accidentally-decouple `SRC_URI` entries (where someone might add the config fragment but forget the patch, or vice versa, leaving an inconsistent half-applied state). In short: pure config-only changes use a `.cfg` fragment directly; config-plus-code changes that should always travel together use an `.scc` file.

---

**Q4: A custom out-of-tree kernel module fails to load on the target with "invalid module format." What's the most likely root cause, and how does the Yocto build system normally prevent this?**

**A:** "Invalid module format" almost always indicates a kernel ABI/version mismatch — Linux kernel modules are tightly coupled to the EXACT kernel build they were compiled against (not just the kernel version NUMBER, but the exact configuration and build, since `vermagic` and symbol versioning/CRC checks embedded in both the kernel and the module must match exactly). This typically happens when a module was built against a different kernel build than what's actually running on the target device — for example, building the module recipe before a kernel recipe update propagated, or manually copying a `.ko` file built in one environment onto a device running a kernel from a different build. Yocto's `module.bbclass` is specifically designed to prevent this: it establishes a proper BitBake task dependency (`do_compile[depends] += "virtual/kernel:do_shared_workdir"`, conceptually) ensuring the module recipe ALWAYS builds against the exact, currently-configured kernel recipe's build tree and headers — as long as you do a full, consistent BitBake-managed image build (not manually copying files around outside the build system), the module and kernel placed into the same image are guaranteed to match. The failure mode in practice almost always traces back to some manual, out-of-band step (manually copying a `.ko` file via scp from a different build, or a `devtool deploy-target` of just the module without a correspondingly redeployed matching kernel) that bypassed this normally-enforced consistency.

---

**Q5: What is virtual/bootloader, and walk through how a BSP layer would replace U-Boot with a different bootloader for a specific board.**

**A:** `virtual/bootloader` is the same abstract-provider indirection pattern used for `virtual/kernel`: every recipe and mechanism in the build that needs "the bootloader" (most directly, the image-building/`do_image_wic` machinery that needs to know what bootloader binary to install into the boot partition) references this virtual name rather than hard-coding "u-boot" specifically, allowing the actual bootloader implementation to be swapped per-machine/per-distro without modifying every dependent piece of the build. To replace U-Boot with a different bootloader for a specific board, a BSP layer's machine configuration file would set `PREFERRED_PROVIDER_virtual/bootloader = "my-alternative-bootloader"` (pointing at whatever actual recipe name provides that alternative bootloader, which itself must declare `PROVIDES += "virtual/bootloader"`, typically via inheriting an appropriate class), along with whatever bootloader-specific configuration variables that alternative bootloader's own recipe/class expects (analogous to U-Boot's `UBOOT_MACHINE`/`UBOOT_CONFIG`). A concrete real example: an x86_64 machine configuration commonly sets `PREFERRED_PROVIDER_virtual/bootloader = "grub-efi"` and `EFI_PROVIDER = "grub-efi"` instead of U-Boot, since GRUB (with UEFI) rather than U-Boot is the standard bootloader choice for x86 platforms — the rest of the image-building machinery (wic partition layout referencing `--source bootimg-partition`, the image recipe's `EXTRA_IMAGEDEPENDS`) continues working correctly because it depends on the abstract `virtual/bootloader`, automatically picking up whichever concrete recipe the machine configuration has selected.
