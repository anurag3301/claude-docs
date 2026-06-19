# Yocto BSP (Board Support Package) Layers — Complete Reference Guide

---

## Table of Contents

1. [What Is a BSP Layer?](#1-what-is-a-bsp-layer)
2. [BSP Layer Directory Structure](#2-bsp-layer-directory-structure)
3. [The MACHINE Configuration File](#3-the-machine-configuration-file)
4. [TUNE Files and CPU Architecture Tuning](#4-tune-files-and-cpu-architecture-tuning)
5. [MACHINE_FEATURES](#5-machine_features)
6. [Serial Console Configuration](#6-serial-console-configuration)
7. [Machine-Specific Overrides](#7-machine-specific-overrides)
8. [Creating a New BSP Layer with yocto-bsp / bitbake-layers](#8-creating-a-new-bsp-layer-with-yocto-bsp--bitbake-layers)
9. [The BSP Tools: wic, formfactor, and Bootloader Integration](#9-the-bsp-tools-wic-formfactor-and-bootloader-integration)
10. [Multi-Machine BSP Layers (Families of Boards)](#10-multi-machine-bsp-layers-families-of-boards)
11. [BSP Layer Compliance (Yocto Project Compatible)](#11-bsp-layer-compliance-yocto-project-compatible)
12. [Hardware Bring-Up Workflow Using a BSP Layer](#12-hardware-bring-up-workflow-using-a-bsp-layer)
13. [Complete Worked Example — A New Board BSP Layer](#13-complete-worked-example--a-new-board-bsp-layer)
14. [Common Pitfalls and Debugging](#14-common-pitfalls-and-debugging)
15. [Interview Questions and Answers](#15-interview-questions-and-answers)

---

## 1. What Is a BSP Layer?

A **BSP (Board Support Package) layer** is a specialization of the general layer concept (covered in the Layers document) focused entirely on **hardware enablement** for one specific board, SoC family, or closely related family of boards. Where a generic software-feature layer (like `meta-qt5`) adds capability that's largely hardware-independent, a BSP layer's entire reason for existing is to answer: "what does BitBake need to know to produce a working, bootable image for THIS specific piece of silicon and the board it's mounted on?"

A BSP layer typically provides:
- One or more `MACHINE` configuration files (`conf/machine/*.conf`)
- CPU architecture tuning includes (or references to oe-core's existing tune files)
- A kernel recipe or kernel bbappend with board-specific device tree/config/patches
- A bootloader recipe or bbappend with board-specific configuration
- Any hardware-specific userspace drivers/firmware blobs/calibration tools not part of mainline
- A `.wks` Wic kickstart file describing the board's expected partition layout
- Formfactor configuration (display resolution/orientation defaults, if applicable)

---

## 2. BSP Layer Directory Structure

```
meta-my-board/
├── conf/
│   ├── layer.conf
│   └── machine/
│       ├── my-board.conf                    ← primary machine definition
│       ├── my-board-lite.conf                ← a variant/SKU of the same family
│       └── include/
│           └── my-board-common.inc            ← shared config across variants
│
├── recipes-kernel/
│   └── linux/
│       ├── linux-yocto_%.bbappend
│       └── linux-yocto/
│           ├── my-board-base.cfg
│           ├── my-board-usb.cfg
│           └── 0001-add-my-board-dts.patch
│
├── recipes-bsp/
│   ├── u-boot/
│   │   ├── u-boot_%.bbappend
│   │   └── u-boot/
│   │       └── 0001-my-board-pinmux.patch
│   └── my-board-firmware/
│       └── my-board-firmware_1.0.bb            ← e.g., binary blobs for a GPU/modem
│
├── recipes-graphics/
│   └── xorg-xserver/
│       └── xserver-xorg-conf_%.bbappend        ← board-specific display config
│
├── recipes-bsp/formfactor/
│   └── formfactor/
│       └── my-board/
│           └── machconfig                       ← screen size/orientation defaults
│
├── wic/
│   └── my-board.wks
│
├── README
├── README.hardware                              ← (Yocto Compatible requirement) hardware-specific build/flash instructions
└── COPYING.MIT
```

---

## 3. The MACHINE Configuration File

This is the heart of a BSP layer — a single `.conf` file that, when `MACHINE = "my-board"` is set in `local.conf`, pulls in everything needed to target this specific hardware.

```bitbake
# conf/machine/my-board.conf

#@TYPE: Machine
#@NAME: My Custom Board
#@DESCRIPTION: Machine configuration for My Custom Board, based on the
# XYZ-SoC (quad-core Cortex-A53), 2GB LPDDR4, eMMC storage.

# ── CPU/Architecture tuning ──
require conf/machine/include/arm/armv8a/tune-cortexa53.inc
DEFAULTTUNE = "cortexa53"

# ── Kernel selection ──
PREFERRED_PROVIDER_virtual/kernel = "linux-yocto"
PREFERRED_VERSION_linux-yocto ?= "6.6%"
KERNEL_IMAGETYPE = "Image"
KERNEL_DEVICETREE = "my-vendor/my-board.dtb"

# ── Bootloader selection ──
PREFERRED_PROVIDER_virtual/bootloader = "u-boot"
PREFERRED_VERSION_u-boot ?= "2024.01%"
UBOOT_MACHINE = "my_board_defconfig"

# ── Serial console ──
SERIAL_CONSOLES = "115200;ttyS0"

# ── Hardware capability flags ──
MACHINE_FEATURES = "usbhost usbgadget alsa bluetooth wifi vfat ext2 keyboard touchscreen"

# ── Image output ──
WKS_FILE = "my-board.wks"
IMAGE_FSTYPES ?= "ext4 wic wic.bmap tar.bz2"

# ── Required additional firmware/recipes pulled into every image for this machine ──
MACHINE_EXTRA_RRECOMMENDS = "kernel-modules linux-firmware-my-wifi-chip"

# ── GPU/graphics provider (if applicable) ──
PREFERRED_PROVIDER_virtual/egl = "my-vendor-gpu-userland"
PREFERRED_PROVIDER_virtual/libgles2 = "my-vendor-gpu-userland"

# ── Machine-specific package architecture tuning string ──
MACHINE_ARCH = "my_board"
```

### Required vs Conventional Variables

| Variable                       | Required? | Purpose                                                       |
|-----------------------------------|-----------|----------------------------------------------------------------|
| `DEFAULTTUNE`                     | Effectively yes | Selects the CPU tuning profile (compiler flags, ABI)      |
| `PREFERRED_PROVIDER_virtual/kernel`| Effectively yes| Which kernel recipe to use for this machine                  |
| `PREFERRED_PROVIDER_virtual/bootloader`| Effectively yes| Which bootloader recipe to use                            |
| `MACHINE_FEATURES`                | Recommended | Drives conditional inclusion of hardware-relevant packages/config in many recipes throughout the build|
| `SERIAL_CONSOLES`                 | Recommended | Ensures getty/console is correctly started on the right UART  |
| `KERNEL_IMAGETYPE`                | Yes (if using kernel.bbclass) | Tells the build/imaging process what kernel binary format to expect|
| `WKS_FILE`                        | If using wic image output | Defines partition layout                               |

---

## 4. TUNE Files and CPU Architecture Tuning

oe-core ships an extensive library of pre-built TUNE include files for essentially every common embedded CPU architecture/microarchitecture combination — a BSP layer almost always REUSES one of these rather than writing tuning parameters from scratch.

```
meta/conf/machine/include/arm/armv7a/tune-cortexa9.inc
meta/conf/machine/include/arm/armv8a/tune-cortexa53.inc
meta/conf/machine/include/arm/armv8a/tune-cortexa72.inc
meta/conf/machine/include/x86/tune-corei7.inc
meta/conf/machine/include/riscv/tune-riscv64.inc
meta/conf/machine/include/mips/tune-mips32.inc
```

```bitbake
# A machine.conf simply requires the appropriate one:
require conf/machine/include/arm/armv8a/tune-cortexa72.inc
DEFAULTTUNE = "cortexa72"
```

### What a Tune File Actually Sets

```bitbake
# Excerpt of conceptual content within tune-cortexa72.inc:
TUNEVALID[cortexa72] = "Enable Cortex-A72 specific processor optimizations"
TUNE_CCARGS:append = "${@bb.utils.contains('TUNE_FEATURES', 'cortexa72', ' -mcpu=cortex-a72', '', d)}"
AVAILTUNES:append = " cortexa72"
TUNE_FEATURES:tune-cortexa72 = "aarch64 cortexa72"
PACKAGE_EXTRA_ARCHS:tune-cortexa72 = "aarch64 armv8a cortexa72"
BASE_LIB:tune-cortexa72 = "lib64"
```

This determines: exact `-mcpu`/`-march` compiler flags passed to GCC/Clang for every recipe built for this machine, the package architecture string used in package feed naming/compatibility checking (`PACKAGE_EXTRA_ARCHS`), and the base library directory convention (`lib` vs `lib64`).

### Multiple CPU Cores in a Family — Tuning for the Lowest Common Denominator vs Specific Variants

```bitbake
# A "generic armv8a" tune that works on ANY ARMv8-A core but doesn't use
# Cortex-A72-specific instruction scheduling optimizations:
DEFAULTTUNE = "armv8a"

# vs the more specific, better-optimized but less portable choice:
DEFAULTTUNE = "cortexa72"
```

Choosing a more SPECIFIC tune produces better-optimized code for that exact CPU but makes the resulting binary package feed incompatible with OTHER, even closely related CPU variants — a relevant consideration if you intend to share one package feed across a family of boards with slightly different exact cores.

---

## 5. MACHINE_FEATURES

Similar in spirit to `DISTRO_FEATURES` but specifically describing HARDWARE capabilities present on this machine, which many recipes check to conditionally include hardware-specific support.

```bitbake
MACHINE_FEATURES = "usbhost usbgadget alsa bluetooth wifi screen touchscreen \
                     keyboard pcbios efi qemu-usermode rtc act_led numeric_lcd \
                     vfat ext2 apm acpi pci pcmcia usb1394 "
```

| Feature        | Meaning                                                            |
|------------------|--------------------------------------------------------------------|
| `usbhost`       | Board has USB host capability (can have USB devices plugged in)     |
| `usbgadget`     | Board has USB device/gadget capability (can BE a USB device to a host)|
| `alsa`          | Board has audio hardware, include ALSA support                       |
| `bluetooth`     | Board has Bluetooth hardware                                          |
| `wifi`          | Board has WiFi hardware                                                |
| `screen`        | Board has a display                                                    |
| `touchscreen`   | Board has touch input                                                  |
| `keyboard`      | Board has (or commonly has attached) a physical keyboard                |
| `rtc`           | Board has a hardware Real-Time Clock                                    |
| `acpi`          | Board supports ACPI (mainly x86)                                        |
| `efi`           | Board boots via UEFI                                                    |

```bitbake
# Inside a recipe checking MACHINE_FEATURES to conditionally include support:
PACKAGECONFIG ??= "${@bb.utils.filter('MACHINE_FEATURES', 'bluetooth wifi', d)}"
```

`MACHINE_EXTRA_RDEPENDS`/`MACHINE_EXTRA_RRECOMMENDS` are commonly set alongside `MACHINE_FEATURES` to pull in hardware-specific firmware/kernel-module packages that this exact board needs but which aren't appropriate to force onto every machine using the same distro configuration:

```bitbake
MACHINE_EXTRA_RRECOMMENDS = "linux-firmware-my-wifi-chipset kernel-module-my-touchscreen-driver"
```

---

## 6. Serial Console Configuration

```bitbake
SERIAL_CONSOLES = "115200;ttyS0"

# Multiple consoles (e.g., a debug UART plus a separate user-facing UART):
SERIAL_CONSOLES = "115200;ttyS0 115200;ttyAMA0"

# Older variable name (still seen in some BSP layers, gradually deprecated):
SERIAL_CONSOLES_CHECK = "${SERIAL_CONSOLES}"
```

This single variable ensures the correct `getty` (login prompt) process is started on the correct UART device at the correct baud rate, both for systemd-based images (generating the correct `serial-getty@.service` instance) and SysVinit-based images (generating the correct `/etc/inittab` entry).

---

## 7. Machine-Specific Overrides

The `MACHINE` value automatically becomes an available override throughout the entire build — letting ANY recipe (not just ones inside the BSP layer itself) conditionally adjust behavior for one specific machine without needing the BSP layer's permission or involvement.

```bitbake
# In ANY recipe, anywhere in any layer:
SRC_URI:append:my-board = " file://my-board-specific.patch"
EXTRA_OECONF:my-board = "--enable-my-board-quirk"
DEPENDS:append:my-board = " my-board-specific-lib"
```

This is exactly the same override mechanism as `DISTRO`-based or `class-target`-based overrides — `MACHINE` values are simply automatically added to the active `OVERRIDES` list.

### MACHINEOVERRIDES — Hierarchical Machine Override Chains

For a family of related boards, `MACHINEOVERRIDES` lets a more specific board ALSO inherit override behavior intended for the whole family:

```bitbake
# conf/machine/my-board-lite.conf (a cost-reduced variant of my-board)
require conf/machine/my-board.conf

MACHINEOVERRIDES =. "my-board-family:"
```

```bitbake
# Now a recipe can target the WHOLE family with one override:
SRC_URI:append:my-board-family = " file://shared-family-fix.patch"

# ...while still being able to target just the lite variant specifically:
SRC_URI:append:my-board-lite = " file://lite-only-cost-reduction-fix.patch"
```

---

## 8. Creating a New BSP Layer with yocto-bsp / bitbake-layers

### Modern Approach: bitbake-layers create-layer + Manual Machine Conf

```bash
bitbake-layers create-layer ../meta-my-board
bitbake-layers add-layer ../meta-my-board

mkdir -p ../meta-my-board/conf/machine
# Then hand-write conf/machine/my-board.conf as shown in Section 3,
# typically starting by copying a similar existing board's machine.conf
# from a reference BSP layer as a starting template
```

### Starting from an Existing Reference BSP

In practice, the most common real-world starting point is NOT a blank layer — it's cloning an existing, similar BSP layer (often provided by the SoC vendor, or a closely related reference board from the layer index) and adapting it, since exact register-level hardware bring-up details (pin muxing, clock trees, regulator configuration) are extremely hardware-specific and rarely worth writing from a completely blank slate when a closely related reference design exists.

```bash
git clone https://github.com/myvendor/meta-myvendor-bsp.git
cp -r meta-myvendor-bsp meta-my-board
cd meta-my-board
# Rename machine.conf, adjust device tree references, update u-boot defconfig
# name, adjust any board-specific GPIO/pinmux patches
```

---

## 9. The BSP Tools: wic, formfactor, and Bootloader Integration

### formfactor — Display and Input Defaults

```
recipes-bsp/formfactor/formfactor/my-board/machconfig
```

```bash
# machconfig content example:
HAVE_TOUCHSCREEN=1
HAVE_KEYBOARD=0
DISPLAY_CAN_ROTATE=0
DISPLAY_ORIENTATION=0
DISPLAY_WIDTH_PIXELS=800
DISPLAY_HEIGHT_PIXELS=480
DISPLAY_BPP=24
DISPLAY_DPI=96
DISPLAY_SUBPIXEL_ORDER=vrgb
```

This is consumed by GUI-stack recipes (e.g., Sato, certain Qt configuration defaults) to provide sensible out-of-the-box display behavior matched to the actual physical screen attached to this board, without needing runtime auto-detection logic in every GUI application.

### wic — Partition Layout (See Image Recipes Document for Full Detail)

```
wic/my-board.wks
```

```
part /boot --source bootimg-partition --ondisk mmcblk0 --fstype=vfat --label boot --active --align 4096 --size 64M
part / --source rootfs --ondisk mmcblk0 --fstype=ext4 --label root --align 4096

bootloader --ptable msdos
```

---

## 10. Multi-Machine BSP Layers (Families of Boards)

A single BSP layer commonly supports MULTIPLE related boards/SKUs — sharing the bulk of configuration via `require`/include patterns while keeping genuinely board-specific differences isolated.

```
meta-my-soc-family/
├── conf/machine/
│   ├── include/
│   │   └── my-soc-common.inc          ← shared SoC-level config (CPU tune, common drivers)
│   ├── my-board-dev-kit.conf           ← vendor's own development/eval board
│   ├── my-board-product-v1.conf        ← a customer product based on the same SoC
│   └── my-board-product-v2.conf        ← a later hardware revision
```

```bitbake
# conf/machine/include/my-soc-common.inc
require conf/machine/include/arm/armv8a/tune-cortexa53.inc
DEFAULTTUNE = "cortexa53"
PREFERRED_PROVIDER_virtual/kernel = "linux-yocto"
KERNEL_IMAGETYPE = "Image"
MACHINE_FEATURES = "usbhost usbgadget alsa wifi bluetooth"

# conf/machine/my-board-product-v1.conf
require conf/machine/include/my-soc-common.inc
KERNEL_DEVICETREE = "my-vendor/my-board-product-v1.dtb"
UBOOT_MACHINE = "my_board_product_v1_defconfig"
WKS_FILE = "my-board-product-v1.wks"

# conf/machine/my-board-product-v2.conf
require conf/machine/include/my-soc-common.inc
KERNEL_DEVICETREE = "my-vendor/my-board-product-v2.dtb"
UBOOT_MACHINE = "my_board_product_v2_defconfig"
WKS_FILE = "my-board-product-v2.wks"
MACHINE_FEATURES:append = " touchscreen"   # v2 added a touchscreen, v1 didn't
```

This structure is exactly analogous to the `.inc` file pattern used for recipe version families — shared logic factored out once, with each concrete machine config file expressing only its genuine deltas.

---

## 11. BSP Layer Compliance (Yocto Project Compatible)

The Yocto Project runs a formal "Yocto Project Compatible" certification program for BSP and software layers, with specific structural and documentation requirements:

- A `README` describing the layer's purpose, maintainer contact, and dependencies
- A `README.hardware` (for BSP layers specifically) with concrete build/flash/boot instructions for the actual hardware
- A `COPYING.MIT` (or equivalent) license file for the layer's OWN original content (distinct from `LICENSE`/`LIC_FILES_CHKSUM` inside individual recipes, which describes the licensing of the SOFTWARE each recipe builds)
- Correctly declared `LAYERSERIES_COMPAT`
- Passing the `yocto-check-layer` automated compliance script

```bash
# Run the official compliance checker against your layer
yocto-check-layer --layers /path/to/meta-my-board

# Checks for: required files present, no obviously broken/missing dependencies,
# recipes don't trigger common QA failures, machine configs are well-formed, etc.
```

Compliance isn't a hard technical requirement for a layer to FUNCTION (BitBake doesn't refuse to use a non-compliant layer), but it matters for: layers intended for public/community distribution via the OpenEmbedded Layer Index, and any organization wanting to formally claim "Yocto Project Compatible" status for marketing/compliance purposes with a vendor's hardware.

---

## 12. Hardware Bring-Up Workflow Using a BSP Layer

A typical real-world new-board bring-up sequence:

```bash
# 1. Start from the closest available reference BSP (vendor eval board, or
#    a similar existing product)
git clone <vendor-reference-bsp-repo> meta-my-board

# 2. Get SOMETHING booting first — usually just the unmodified reference
#    board's machine.conf, to validate the basic toolchain/build works at all
bitbake core-image-minimal

# 3. Adapt the device tree for actual board differences (different
#    DRAM size/timing, different peripheral wiring, removed/added components)
devtool modify virtual/kernel
# edit arch/.../dts/my-board.dts directly
devtool build virtual/kernel
devtool deploy-target virtual/kernel root@192.168.1.50  # or via TFTP/serial for early bring-up

# 4. Iterate on U-Boot configuration for board-specific pin muxing/clock setup
devtool modify virtual/bootloader
# edit board-specific U-Boot config/source
devtool build virtual/bootloader

# 5. Once stable, finalize all devtool work back into the BSP layer
devtool finish virtual/kernel meta-my-board
devtool finish virtual/bootloader meta-my-board

# 6. Build and validate the final, clean (non-devtool-workspace) image
bitbake -c cleansstate virtual/kernel virtual/bootloader
bitbake core-image-minimal
```

This demonstrates how the BSP layer document's content (machine configuration, kernel/bootloader recipes) and the devtool document's iterative workflow combine in practice during actual hardware bring-up.

---

## 13. Complete Worked Example — A New Board BSP Layer

```bitbake
# meta-acme-widget/conf/layer.conf
BBPATH .= ":${LAYERDIR}"
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"
BBFILE_COLLECTIONS += "acme-widget"
BBFILE_PATTERN_acme-widget = "^${LAYERDIR}/"
BBFILE_PRIORITY_acme-widget = "7"
LAYERDEPENDS_acme-widget = "core"
LAYERSERIES_COMPAT_acme-widget = "scarthgap styhead"
```

```bitbake
# meta-acme-widget/conf/machine/acme-widget.conf

#@TYPE: Machine
#@NAME: Acme Widget Gateway Board
#@DESCRIPTION: Acme Corp's Widget Gateway product, based on the Acme AW-SoC1
# (dual-core Cortex-A53 @ 1.2GHz, 512MB DDR3, 4GB eMMC)

require conf/machine/include/arm/armv8a/tune-cortexa53.inc
DEFAULTTUNE = "cortexa53"

PREFERRED_PROVIDER_virtual/kernel = "linux-yocto"
PREFERRED_VERSION_linux-yocto ?= "6.6%"
KERNEL_IMAGETYPE = "Image"
KERNEL_DEVICETREE = "acme/acme-widget.dtb"

PREFERRED_PROVIDER_virtual/bootloader = "u-boot"
UBOOT_MACHINE = "acme_widget_defconfig"

SERIAL_CONSOLES = "115200;ttyS0"

MACHINE_FEATURES = "usbhost usbgadget wifi bluetooth ext4 rtc"
MACHINE_EXTRA_RRECOMMENDS = "kernel-modules linux-firmware-acme-wifi"

WKS_FILE = "acme-widget.wks"
IMAGE_FSTYPES ?= "ext4 wic wic.bmap"

MACHINE_ARCH = "acme_widget"
```

```bitbake
# meta-acme-widget/recipes-kernel/linux/linux-yocto_%.bbappend

FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

COMPATIBLE_MACHINE:acme-widget = "acme-widget"
KBRANCH:acme-widget = "v6.6/standard/acme-widget"
SRCREV_machine:acme-widget = "9f3c2a1b8e7d6f5a4c3b2a1908765432fedcba98"
KMACHINE:acme-widget = "acme-widget"

SRC_URI:append:acme-widget = " \
    file://0001-add-acme-widget-devicetree.patch \
    file://acme-widget-base.cfg \
    file://acme-widget-wifi.cfg \
    "
```

```
# meta-acme-widget/wic/acme-widget.wks
part /boot --source bootimg-partition --ondisk mmcblk0 --fstype=vfat --label boot --active --align 4096 --size 64M
part / --source rootfs --ondisk mmcblk0 --fstype=ext4 --label root --align 4096
bootloader --ptable msdos
```

```bash
# Usage:
bitbake-layers add-layer meta-acme-widget
echo 'MACHINE = "acme-widget"' >> conf/local.conf
bitbake core-image-minimal
```

---

## 14. Common Pitfalls and Debugging

### 1. New MACHINE not recognized — "Nothing PROVIDES 'virtual/kernel'"
Check `conf/machine/<machine-name>.conf` filename EXACTLY matches the `MACHINE` value set in local.conf (case-sensitive, no extension mismatch), and that the BSP layer is actually added via `bitbake-layers show-layers`.

### 2. Wrong CPU tuning silently applied
Verify `require conf/machine/include/.../tune-*.inc` is actually present and `DEFAULTTUNE` matches a tune name that file actually defines — a typo here doesn't always hard-fail, sometimes silently falling back to a more generic (less optimized, but still functional) tuning, masking the mistake.

### 3. Image builds but real hardware doesn't boot, QEMU equivalent works fine
Classic missing-bootloader-configuration symptom — double check `UBOOT_MACHINE`/U-Boot defconfig name is exactly correct for your actual board (not a similarly-named but different board from a reference BSP you copied from), and that `KERNEL_DEVICETREE` matches the actual physical board revision (a common mistake when copying from a vendor eval-board BSP to a customer product board with a similar but not identical SoC pinout).

### 4. MACHINE_FEATURES set but a recipe still doesn't include the expected hardware support
Remember `MACHINE_FEATURES` alone doesn't automatically do anything — individual RECIPES must explicitly check for it (commonly via `PACKAGECONFIG` filtering, as shown in Section 5) for it to have any actual effect; not every recipe's optional features are wired up to check `MACHINE_FEATURES`, especially less common/niche recipes. Verify the specific recipe you expect to respond actually contains conditional logic referencing `MACHINE_FEATURES`.

### 5. Two boards in the same family silently diverge over time due to copy-paste machine.conf files
A common maintenance trap when NOT using the shared `.inc` file pattern (Section 10) — a fix/update applied to one board's standalone `.conf` file simply never propagates to a sibling board's separate, copy-pasted `.conf` file. Refactoring toward a shared common `.inc` file with only genuine per-board deltas in each board's own `.conf` is the standard fix once a BSP layer's board count grows past one or two.

---

## 15. Interview Questions and Answers

**Q1: What is the minimum set of things a MACHINE configuration file genuinely needs to define for a basic bootable image, and why are the rest "merely" conventional/recommended?**

**A:** At a strict technical minimum, a `MACHINE` configuration needs enough information for BitBake to successfully resolve `virtual/kernel` and (if using wic/a full bootable disk image rather than just an NFS-root style deployment) `virtual/bootloader` to actual buildable recipes, and a `DEFAULTTUNE` so the cross-compiler knows what CPU instructions/ABI to target — without these, recipes simply cannot be built and resolved into a coherent image at all. Everything else covered in this document (`MACHINE_FEATURES`, `SERIAL_CONSOLES`, formfactor configuration, `WKS_FILE`) is "merely" strongly conventional in the sense that a build WILL technically still complete and produce SOME image without them, but the result will be missing important real-world functionality: without `SERIAL_CONSOLES` correctly set, you won't get a login prompt on the actual UART you're physically connected to even though the kernel itself boots fine; without `MACHINE_FEATURES` appropriately set, various recipes throughout the build won't know to conditionally include hardware-relevant support; without a `WKS_FILE`, you simply don't get a directly-flashable partitioned disk image output format (though `ext4`/`tar.bz2` outputs would still work fine for other deployment methods like NFS-root or manually-scripted partitioning). The practical reality is that essentially every real BSP layer sets all of these, because skipping them just means re-discovering the need for them one frustrating debugging session at a time.

---

**Q2: Explain why a BSP layer for a family of related boards typically factors shared configuration into a common .inc file rather than just copy-pasting between each board's individual machine.conf.**

**A:** This is exactly the same DRY (don't-repeat-yourself) reasoning that applies throughout software engineering, here applied at the machine-configuration level. When N related boards (sharing the same core SoC, differing mainly in peripheral population, form factor, or minor hardware revisions) each have a fully independent, copy-pasted `machine.conf`, any change that should logically apply to the WHOLE family — a kernel branch update, a newly-discovered CPU tuning correction, a shared driver fix — has to be manually, separately applied to every single board's file, and it's extremely easy for this process to be incomplete (a fix applied to boards 1, 2, and 3 but accidentally forgotten on board 4, which then silently drifts out of sync and may carry a known-bad configuration indefinitely until someone notices). Factoring the genuinely shared configuration (CPU tune selection, kernel provider/branch, common `MACHINE_FEATURES`, shared driver requirements) into one `conf/machine/include/<family>-common.inc` file that every board's individual `.conf` file `require`s means a single edit to that shared file automatically and correctly propagates to every board in the family simultaneously, while each board's own `.conf` file remains small and focused, containing ONLY its genuine, board-specific deltas (its specific device tree filename, its specific U-Boot defconfig name, any board-unique `MACHINE_FEATURES` additions/removals) — making it immediately obvious, just by reading that short file, exactly how this board differs from the rest of its family.

---

**Q3: What's the difference between MACHINE_FEATURES and DISTRO_FEATURES, and why might the same physical capability sometimes need to be reflected in both?**

**A:** (This connects to the Image Recipes document's coverage of DISTRO_FEATURES, viewed from the BSP/hardware side.) `MACHINE_FEATURES` describes what HARDWARE CAPABILITIES this specific board physically has (does it have Bluetooth silicon at all, does it have a touchscreen, does it have USB host capability) — it's set once per machine, in the BSP layer's machine configuration. `DISTRO_FEATURES` describes what SOFTWARE CAPABILITIES the overall distribution build has chosen to compile support for across the board (is Bluetooth software stack support built at all, anywhere in this distro configuration) — it's set once per distro, independent of any specific machine. These can genuinely diverge in both directions: a distro might set `DISTRO_FEATURES` to include "bluetooth" (compiling Bluetooth stack support into relevant packages) even though it's ALSO being built for some machines that have no Bluetooth hardware at all (those specific machines' images simply won't usefully use that compiled-in capability, but it doesn't hurt anything beyond a small amount of unused compiled code); conversely, a machine could have actual Bluetooth hardware (`MACHINE_FEATURES` correctly includes "bluetooth") but if the distro-wide `DISTRO_FEATURES` never included "bluetooth" in the first place, no package was ever compiled with Bluetooth support, and that hardware capability is simply unusable in software regardless of what the machine configuration says — `MACHINE_FEATURES` describing real hardware capability is necessary but not sufficient; the corresponding `DISTRO_FEATURES` entry must ALSO be present at the distro level for any recipe's conditional `PACKAGECONFIG` logic (which typically checks `DISTRO_FEATURES`, not `MACHINE_FEATURES`, for build-time compilation decisions) to actually activate that capability in the compiled software.

---

**Q4: Why do most real-world BSP layers start from an existing vendor/reference board layer rather than being written entirely from scratch?**

**A:** The overwhelming majority of the genuinely hard, low-level work in bringing up a new embedded board — correct DRAM timing/calibration parameters for the specific memory chips used, exact pin-muxing configuration matching the board's actual schematic, clock tree configuration for the specific oscillators/PLLs present, power sequencing for the specific PMIC and voltage rail design, and the device tree describing all of this to the kernel — is fundamentally HARDWARE-SPECIFIC, low-level, and time-consuming to get right, requiring either detailed datasheets/schematics or substantial trial-and-error with test equipment (oscilloscopes, logic analyzers) on physical hardware. An SoC vendor's own reference/evaluation board BSP layer has already done this hard work for THAT specific reference design, and a new product based on the SAME SoC family is very likely to share the vast majority of this low-level configuration with only modest deltas (different exact DRAM part/size, different peripheral population, different physical connector layout) — meaning starting from that reference layer and applying targeted deltas to the device tree, U-Boot pin-muxing config, and machine configuration is dramatically faster and less error-prone than re-deriving correct DRAM timing parameters and pin-mux tables from a completely blank starting point. This mirrors a more general engineering principle: don't re-derive low-level, hardware-tightly-coupled configuration from first principles when a closely related, already-validated reference design exists to build from instead.

---

**Q5: A new board boots correctly in QEMU's emulated equivalent machine but fails to boot on the real physical hardware. Walk through a systematic debugging approach grounded in BSP layer concepts.**

**A:** Since QEMU's emulated machine necessarily abstracts away most of the genuinely hardware-specific concerns a real BSP layer has to handle correctly, this symptom strongly points toward something in the REAL-hardware-specific configuration layer rather than a generic kernel/userspace software bug (which would typically fail in QEMU too). A systematic approach: (1) First confirm you're actually getting ANY output at all on the physical UART — verify `SERIAL_CONSOLES` baud rate and UART device name exactly match the physical board's actual debug UART wiring and the serial terminal program's configuration; many "doesn't boot" reports are actually "boots fine, but I'm not looking at the right serial port/baud rate." (2) If genuinely no boot activity at all, suspect the bootloader stage specifically — verify `UBOOT_MACHINE` matches the EXACT board variant (a common mistake: using a vendor eval-board's defconfig name on a customer product board with a similar but not identical SoC/memory configuration, leading to wrong DRAM timing being applied and the board hanging before even reaching a console-visible state). (3) If the bootloader successfully starts (visible U-Boot banner on serial) but the kernel fails to boot or hangs, suspect the device tree — verify `KERNEL_DEVICETREE` points at a `.dtb` that actually correctly describes THIS board's physical peripheral wiring/memory map, not a similar-but-different reference board's device tree copied without adaptation. (4) If the kernel boots but specific peripherals don't work, check the relevant kernel configuration fragments and `MACHINE_FEATURES`/`PACKAGECONFIG` wiring for that specific peripheral are actually present and correctly merged into the final kernel `.config` (via `bitbake -c menuconfig virtual/kernel`). Each of these stages corresponds to a genuinely different layer of BSP-specific configuration (bootloader config, device tree, kernel config), and systematically confirming which stage is the first one that diverges from expected behavior on real hardware (versus QEMU, which never exercises any of these real-hardware-specific configuration layers) is the key diagnostic structure.
