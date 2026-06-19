# Yocto Image Recipes — Complete Reference Guide

---

## Table of Contents

1. [Introduction to Image Recipes](#1-introduction-to-image-recipes)
2. [How Image Recipes Differ from Normal Recipes](#2-how-image-recipes-differ-from-normal-recipes)
3. [image.bbclass and the Image Build Pipeline](#3-imagebbclass-and-the-image-build-pipeline)
4. [Standard Reference Images (oe-core/poky)](#4-standard-reference-images-oe-corepoky)
5. [IMAGE_INSTALL and Package Selection](#5-image_install-and-package-selection)
6. [IMAGE_FEATURES and DISTRO_FEATURES](#6-image_features-and-distro_features)
7. [Writing a Custom Image Recipe](#7-writing-a-custom-image-recipe)
8. [Image Recipe Variables Reference](#8-image-recipe-variables-reference)
9. [Package Manager Backends (RPM, DEB, IPK)](#9-package-manager-backends-rpm-deb-ipk)
10. [Root Filesystem Construction (do_rootfs)](#10-root-filesystem-construction-do_rootfs)
11. [IMAGE_FSTYPES and Output Formats](#11-image_fstypes-and-output-formats)
12. [WIC — Partitioned Disk Images](#12-wic--partitioned-disk-images)
13. [Image Size Management](#13-image-size-management)
14. [Read-Only and Read-Write Rootfs Strategies](#14-read-only-and-read-write-rootfs-strategies)
15. [Customizing the Rootfs with ROOTFS_POSTPROCESS_COMMAND](#15-customizing-the-rootfs-with-rootfs_postprocess_command)
16. [Image Generation Hooks and Custom Tasks](#16-image-generation-hooks-and-custom-tasks)
17. [SDK Generation from Images](#17-sdk-generation-from-images)
18. [Complete Worked Example — Custom Product Image](#18-complete-worked-example--custom-product-image)
19. [Common Pitfalls and Debugging](#19-common-pitfalls-and-debugging)
20. [Interview Questions and Answers](#20-interview-questions-and-answers)

---

## 1. Introduction to Image Recipes

An **image recipe** is a special kind of `.bb` recipe whose job is not to build a single piece of software, but to **assemble a complete, bootable root filesystem** by selecting and combining the output packages of many other recipes. Where a normal recipe's tasks end at `do_package`/`do_populate_sysroot` (producing individual installable package files), an image recipe's tasks continue further: `do_rootfs` (assemble all selected packages into one coherent filesystem tree, resolving dependencies via the package manager) and `do_image`/`do_image_<format>` (convert that filesystem tree into final, deployable image file formats like ext4, squashfs, or a complete partitioned disk image).

```
Normal recipe:    fetch → unpack → patch → configure → compile → install → package
                                                                                │
                                                                                ▼
Image recipe:                                              (selects MANY packages)
                                                                  → do_rootfs
                                                                  → do_image
                                                                  → do_image_ext4 / do_image_wic / ...
```

Image recipes conventionally live in a `recipes-core/images/` (or similar) directory and are named with an `-image` suffix by strong convention (though not technically required), e.g., `core-image-minimal.bb`, `core-image-sato.bb`.

---

## 2. How Image Recipes Differ from Normal Recipes

| Aspect                  | Normal Recipe                            | Image Recipe                                       |
|----------------------------|---------------------------------------------|-----------------------------------------------------|
| Inherits                  | `autotools`/`cmake`/etc.                    | `core-image` (which itself inherits `image`)         |
| `SRC_URI`                 | Points at actual upstream source code        | Usually empty/unused — there's no "source" to fetch  |
| Primary "dependency" mechanism | `DEPENDS` (build-time)                  | `IMAGE_INSTALL`/`PACKAGE_INSTALL` (runtime package selection)|
| Output                    | One or more `.rpm`/`.deb`/`.ipk` package files| A complete root filesystem + bootable image file(s)   |
| `do_compile`/`do_install`  | The real, substantive build work             | Essentially no-ops (nothing to compile/install per se)|
| Unique tasks               | N/A                                          | `do_rootfs`, `do_image`, `do_image_<fstype>`          |
| `PACKAGES`/`FILES`         | Splits build output into packages             | Not applicable — an image isn't itself "packaged" further (though `do_populate_sdk` is a related but separate concept)|

```bitbake
# A truly minimal image recipe:
SUMMARY = "A minimal demo image"
LICENSE = "MIT"

inherit core-image

IMAGE_INSTALL = "packagegroup-core-boot"
```

That alone — just `inherit core-image` plus a package selection — is enough to produce a fully bootable root filesystem image.

---

## 3. image.bbclass and the Image Build Pipeline

`image.bbclass` (and `core-image.bbclass`, which most real image recipes actually `inherit` since it adds sensible defaults on top of the lower-level `image.bbclass`) defines the entire pipeline:

```
do_rootfs
   ↓ (package manager — dnf/apt/opkg depending on PACKAGE_CLASSES — resolves and
   ↓  installs every package from IMAGE_INSTALL plus their RDEPENDS, into a
   ↓  fresh filesystem tree at ${IMAGE_ROOTFS})
do_image
   ↓ (runs ROOTFS_POSTPROCESS_COMMAND hooks: cleanup, permission fixups,
   ↓  removing package manager metadata if not needed on target, etc.)
do_image_<fstype>
   ↓ (one task PER entry in IMAGE_FSTYPES — e.g., do_image_ext4, do_image_tar,
   ↓  do_image_wic — converts the finished rootfs tree into that specific
   ↓  output file format)
do_image_complete
   ↓ (meta-task depending on all the do_image_<fstype> tasks)
do_build
   (top-level meta-task)
```

### core-image.bbclass vs image.bbclass

`image.bbclass` provides the low-level mechanics (the task pipeline above). `core-image.bbclass` (which essentially every real-world image recipe inherits, directly or via inheriting `core-image`) adds a convenient, higher-level interface on top: predefined `IMAGE_FEATURES` options that map to ready-made package groups (so you write `IMAGE_FEATURES += "ssh-server-openssh"` instead of manually listing every package an SSH server needs), and sensible default `IMAGE_INSTALL` baseline content.

```bitbake
inherit core-image
# (which itself does: inherit image, then adds the IMAGE_FEATURES convenience layer)
```

---

## 4. Standard Reference Images (oe-core/poky)

| Image Recipe                  | Contents                                                          |
|----------------------------------|------------------------------------------------------------------|
| `core-image-minimal`            | Smallest possible bootable image — just enough to boot to a shell |
| `core-image-minimal-dev`        | Minimal + development headers/tools for on-target compilation     |
| `core-image-minimal-initramfs`  | Minimal built specifically as an initramfs (early-boot root)      |
| `core-image-full-cmdline`       | Minimal + a fuller set of standard command-line utilities (more than busybox provides) |
| `core-image-base`               | Full hardware-enablement console image (no GUI) — common starting point for real products|
| `core-image-x11`                | Minimal X11 graphical environment, no desktop shell                |
| `core-image-sato`               | Full Sato reference GUI desktop environment (GTK-based)            |
| `core-image-sato-dev`           | Sato + development tools                                          |
| `core-image-sato-sdk`           | Sato + the FULL SDK toolchain baked directly into the image (large)|
| `core-image-rt`                 | Minimal + PREEMPT_RT real-time kernel variant                      |
| `core-image-lsb`/`core-image-lsb-sdk`| Linux Standard Base compliance-oriented images (largely legacy)|
| `core-image-testmaster`/`core-image-testmaster-initramfs`| Used internally by the Yocto Project's automated test infrastructure|

Most real products don't ship `core-image-sato` (a reference desktop demo) — they instead create a **custom image recipe** (commonly inheriting `core-image` directly, or `require`-ing `core-image-minimal.bb` as a starting template) tailored to the product's actual needs.

---

## 5. IMAGE_INSTALL and Package Selection

```bitbake
# Base set always included by core-image.bbclass (varies by exact mechanism, but
# conceptually): packagegroup-core-boot and similar low-level essentials
IMAGE_INSTALL = "packagegroup-core-boot ${CORE_IMAGE_EXTRA_INSTALL}"

# The conventional way to ADD packages without disturbing the careful base set:
IMAGE_INSTALL:append = " htop nano openssh python3 my-custom-app"

# Explicitly REMOVE a package that some included packagegroup pulls in but
# you don't want
IMAGE_INSTALL:remove = "unwanted-package"
```

### packagegroup-* Recipes — Bundling Related Packages

A `packagegroup-*` recipe is a recipe whose sole purpose is to be an "empty" container that pulls in a curated set of other packages via `RDEPENDS` — making it easy for an image recipe to include a whole coherent feature set with one `IMAGE_INSTALL` entry instead of listing dozens of individual package names.

```bitbake
# Example: packagegroup-my-product-base.bb
SUMMARY = "Base package set for MyProduct devices"
LICENSE = "MIT"

inherit packagegroup

PACKAGES = "${PN}"

RDEPENDS:${PN} = "\
    bash \
    openssh-sshd \
    dropbear \
    htop \
    my-app \
    my-app-config \
    "
```

```bitbake
# Then in the image recipe:
IMAGE_INSTALL:append = " packagegroup-my-product-base"
```

This indirection is valuable because it lets you define "what a MyProduct device needs" exactly once, in a single reusable recipe, rather than duplicating a long package list across multiple image variants (e.g., a "production" image and a "debug" image that share 95% of the same base package set).

### PACKAGE_INSTALL vs IMAGE_INSTALL

`PACKAGE_INSTALL` is the LOWER-LEVEL variable that `do_rootfs` actually reads to know what to install — `IMAGE_INSTALL` is the higher-level, image-recipe-author-facing variable that `core-image.bbclass` processes (merging in IMAGE_FEATURES-derived packages, etc.) to ultimately compute `PACKAGE_INSTALL`. As a recipe author you almost always set `IMAGE_INSTALL`, not `PACKAGE_INSTALL` directly, letting the class machinery handle the translation correctly.

---

## 6. IMAGE_FEATURES and DISTRO_FEATURES

### IMAGE_FEATURES — Image-Level Toggleable Capabilities

```bitbake
IMAGE_FEATURES += "ssh-server-openssh"
IMAGE_FEATURES += "package-management"
IMAGE_FEATURES += "debug-tweaks"
IMAGE_FEATURES += "tools-debug"
IMAGE_FEATURES += "dev-pkgs"
IMAGE_FEATURES += "read-only-rootfs"
```

| Feature                | Effect                                                              |
|---------------------------|----------------------------------------------------------------|
| `ssh-server-openssh`     | Installs and enables full OpenSSH server                          |
| `ssh-server-dropbear`    | Installs and enables the much smaller Dropbear SSH server         |
| `package-management`     | Leaves the package manager (rpm/dpkg/opkg) and its database on the target — needed for runtime package install/update, omitted by default for a leaner immutable image|
| `debug-tweaks`           | **Development only**: empty root password, allow root SSH login    |
| `tools-debug`            | Installs gdb, strace, and similar debugging tools                  |
| `tools-sdk`              | Installs a full native compiler toolchain ONTO the target itself (large — for on-device compilation)|
| `tools-profile`          | Installs profiling tools (oprofile, perf, etc.)                    |
| `dev-pkgs`               | Installs the `-dev` (headers) package variant for every package in the image|
| `dbg-pkgs`               | Installs the `-dbg` (debug symbols) package variant for every package |
| `doc-pkgs`               | Installs the `-doc` package variant for every package               |
| `read-only-rootfs`       | Configures the rootfs and relevant services for a read-only root filesystem (see Section 14)|
| `splash`                 | Includes a graphical splash screen during boot                      |
| `x11-base`               | Minimal X11 server, no window manager/desktop                       |
| `allow-empty-password`   | Allow empty passwords for login (separate from debug-tweaks's root-specific behavior)|
| `empty-root-password`    | Sets root's password to empty specifically                          |

### DISTRO_FEATURES — Distribution-Wide Capability Flags

`DISTRO_FEATURES` is set once at the distro-configuration level (not per-image) and controls which OPTIONAL CAPABILITIES are compiled into recipes throughout the ENTIRE build — not just which packages get installed into one image, but which features get conditionally compiled into the packages themselves via each recipe's own `PACKAGECONFIG`/conditional logic checking `DISTRO_FEATURES`.

```bitbake
# In a distro .conf file:
DISTRO_FEATURES = "acl alsa bluetooth ext2 ipv4 ipv6 pcmcia usbgadget usbhost \
                    wifi xattr nfs zeroconf pci 3g nfc systemd wayland opengl"

DISTRO_FEATURES:append = " ${DISTRO_FEATURES_LIBC}"
```

```bitbake
# Inside an individual recipe, conditionally enabling a feature based on
# whether the distro has opted into it:
PACKAGECONFIG ??= "${@bb.utils.filter('DISTRO_FEATURES', 'wayland x11', d)}"
```

The crucial distinction: `IMAGE_FEATURES` decides what gets INSTALLED into one specific image; `DISTRO_FEATURES` decides what gets BUILT (compiled in as an option) across the ENTIRE distribution, affecting every recipe that checks it, for every image built under that distro configuration. A package compiled WITHOUT wayland support (because `DISTRO_FEATURES` lacked "wayland") can never gain wayland support just by adjusting one image's `IMAGE_INSTALL` — the capability has to be present at the distro level for any image built under it to potentially use it.

---

## 7. Writing a Custom Image Recipe

### Method 1: From Scratch

```bitbake
SUMMARY = "MyProduct production image"
LICENSE = "MIT"

inherit core-image

IMAGE_FEATURES += "ssh-server-openssh package-management"

IMAGE_INSTALL = "packagegroup-core-boot \
                  packagegroup-my-product-base \
                  my-app \
                  my-app-config \
                  "

IMAGE_INSTALL:remove = "packagegroup-core-ssh-dropbear"

IMAGE_ROOTFS_SIZE = "1048576"
IMAGE_ROOTFS_EXTRA_SPACE = "524288"

LICENSE_CREATE_PACKAGE = "1"
```

### Method 2: require an Existing Image as a Base

```bitbake
# my-product-image.bb
require recipes-core/images/core-image-minimal.bb

SUMMARY = "MyProduct image, based on core-image-minimal"

IMAGE_INSTALL:append = " my-app openssh htop"
IMAGE_FEATURES:append = " ssh-server-openssh"
```

This is a common, pragmatic pattern — start from a well-tested reference image's recipe and layer your own additions on top via `require` + `:append`, rather than reconstructing everything from scratch.

### Method 3: A Family of Related Images Sharing a Common .inc

```bitbake
# my-product-common.inc
inherit core-image
IMAGE_INSTALL = "packagegroup-core-boot packagegroup-my-product-base"
IMAGE_FEATURES += "package-management"

# my-product-image-prod.bb
require my-product-common.inc
SUMMARY = "Production image — minimal, locked down"

# my-product-image-dev.bb
require my-product-common.inc
SUMMARY = "Development image — debug tools included"
IMAGE_FEATURES += "debug-tweaks tools-debug ssh-server-openssh"
IMAGE_INSTALL:append = " gdb strace tcpdump"
```

---

## 8. Image Recipe Variables Reference

| Variable                     | Purpose                                                          |
|---------------------------------|--------------------------------------------------------------|
| `IMAGE_INSTALL`                | Primary package selection list                                  |
| `IMAGE_INSTALL:append`/`:remove`| Standard add/remove customization pattern                       |
| `IMAGE_FEATURES`                | Toggleable high-level capability set (maps to package groups)   |
| `IMAGE_FSTYPES`                 | Output file format(s) to generate (ext4, tar.bz2, wic, etc.)     |
| `IMAGE_ROOTFS_SIZE`             | Target rootfs size in KB (if not auto-calculated)                 |
| `IMAGE_ROOTFS_EXTRA_SPACE`      | Extra free space (KB) added beyond computed content size          |
| `IMAGE_OVERHEAD_FACTOR`         | Multiplier applied to computed rootfs size for filesystem overhead|
| `IMAGE_LINGUAS`                 | Which locale/language packages to include (e.g., "en-us de-de")  |
| `PACKAGE_EXCLUDE`               | Globally exclude specific packages from ALL images in this build  |
| `PACKAGE_EXCLUDE_COMPLEMENTARY` | Exclude entire complementary package classes (e.g., all `-dev`)   |
| `BAD_RECOMMENDATIONS`           | Suppress specific RRECOMMENDS from being auto-pulled in            |
| `NO_RECOMMENDATIONS`            | Globally disable ALL RRECOMMENDS processing (smaller, stricter images)|
| `ROOTFS_POSTPROCESS_COMMAND`    | Shell functions to run after rootfs assembly, before imaging       |
| `IMAGE_PREPROCESS_COMMAND`      | Shell functions to run before rootfs assembly begins                |
| `EXTRA_IMAGEDEPENDS`            | Extra recipe build dependencies for the image (e.g., bootloader)   |
| `INITRAMFS_IMAGE`               | Name of a separate initramfs image recipe to bundle/use             |
| `INITRD_IMAGE`/`INITRD_IMAGE_LIVE`| Initrd image to use for certain boot configurations               |
| `WKS_FILE`                      | Wic kickstart file defining partition layout (see Section 12)       |
| `DEPENDS` (rare on image recipes)| Build-time deps for the image recipe ITSELF (e.g., a custom imaging tool)|

---

## 9. Package Manager Backends (RPM, DEB, IPK)

```bitbake
# Set globally in local.conf or distro config — affects ALL recipes' do_package_write_*
PACKAGE_CLASSES = "package_rpm"
# Alternatives: package_deb, package_ipk, package_tar (rarely used alone)

# Multiple backends can be enabled simultaneously (each produces its own package
# files in parallel; the FIRST listed is what do_rootfs actually uses to build images)
PACKAGE_CLASSES = "package_rpm package_ipk"
```

| Format  | Package Manager on Target | Typical Use Case                                    |
|----------|---------------------------|---------------------------------------------------|
| RPM     | `dnf`/`rpm`                | Most common modern default; good dependency resolution, delta-RPM support|
| DEB     | `apt`/`dpkg`                | Familiar to Debian/Ubuntu-experienced teams           |
| IPK     | `opkg`                      | Lightweight, traditionally favored for very small/embedded targets|

The choice mostly affects: final image size overhead (package manager + database size on target if `package-management` IMAGE_FEATURE is kept), familiarity for the team maintaining the product, and whether runtime package management (field updates via package feed) is needed at all (many embedded products use whole-image OTA updates instead and don't need ANY package manager on the running target, in which case the choice matters far less and `package-management` is usually left OUT of `IMAGE_FEATURES` to save space).

---

## 10. Root Filesystem Construction (do_rootfs)

`do_rootfs` is the task that actually invokes the chosen package manager (dnf, apt, or opkg, running in a special build-host-side mode targeting the rootfs directory) to resolve the full dependency closure of everything in `PACKAGE_INSTALL`, download/copy the relevant packages from the locally-built package feed, and unpack/install them into `${IMAGE_ROOTFS}`.

```bash
# Inspect the assembled-but-not-yet-imaged rootfs directly (useful for debugging)
ls tmp/work/<machine>/<image-recipe>/<version>/rootfs/

# See the full package manifest that ended up in a built image
cat tmp/deploy/images/<machine>/<image-name>.manifest
```

### Dependency Resolution During do_rootfs

Just like a normal `apt install`/`dnf install` on a desktop distro, do_rootfs's package manager invocation resolves the FULL transitive closure of `RDEPENDS` for everything listed in `IMAGE_INSTALL` — meaning the image's actual final package list is typically much larger than what you explicitly wrote in `IMAGE_INSTALL`, because every package's own runtime dependencies (and THEIR dependencies, recursively) get pulled in automatically.

---

## 11. IMAGE_FSTYPES and Output Formats

```bitbake
IMAGE_FSTYPES = "ext4 tar.bz2"

# Common additional/alternative formats:
IMAGE_FSTYPES += "wic wic.bmap"      # Partitioned disk image (see Section 12)
IMAGE_FSTYPES += "squashfs"           # Compressed, read-only filesystem
IMAGE_FSTYPES += "ubi ubifs"          # For raw NAND flash targets
IMAGE_FSTYPES += "cpio.gz"            # Common initramfs format
IMAGE_FSTYPES += "jffs2"              # Older flash filesystem format
IMAGE_FSTYPES += "vmdk"               # VMware virtual disk
IMAGE_FSTYPES += "container"          # OCI/Docker container image format
```

| Format       | Typical Storage Medium / Use Case                                |
|---------------|------------------------------------------------------------------|
| `ext4`       | General-purpose; most common for eMMC/SD-card-based embedded Linux|
| `squashfs`   | Read-only, highly compressed — common for the read-only base layer of an A/B OTA scheme|
| `ubifs`/`ubi`| Raw NAND flash (requires UBI/MTD-aware bootloader and kernel)      |
| `jffs2`      | Older raw flash filesystem, mostly superseded by UBIFS on new designs|
| `wic`        | A complete, bootable, partitioned disk image (the "whole SD card image")|
| `tar.bz2`/`tar.gz`| Compressed tarball — useful as an intermediate format, NFS root, or container base|
| `cpio.gz`    | Standard initramfs/initrd archive format                            |
| `vmdk`/`vdi`/`qcow2`| Virtual machine disk formats, for testing images in a VM         |
| `container`  | OCI-compatible container image, importable directly with `docker load`|

```bash
# Output location after a successful build
ls tmp/deploy/images/<machine>/
core-image-minimal-<machine>.ext4
core-image-minimal-<machine>.tar.bz2
core-image-minimal-<machine>.manifest
core-image-minimal-<machine>.wic
```

---

## 12. WIC — Partitioned Disk Images

**Wic** ("Where's It Coming from" — yes, genuinely a tongue-in-cheek backronym referencing Fedora's similar `livecd-tools` heritage) generates a complete, bootable, multi-partition disk image — the format you'd write directly to an SD card or eMMC with `dd` and have a fully working, bootable device.

### Kickstart (.wks) File Syntax

```
# my-board.wks
part /boot --source bootimg-partition --ondisk mmcblk0 --fstype=vfat --label boot --active --align 4096 --size 64M

part / --source rootfs --ondisk mmcblk0 --fstype=ext4 --label root --align 4096

bootloader --ptable msdos
```

| Directive                | Meaning                                                          |
|-----------------------------|------------------------------------------------------------------|
| `part`                     | Defines one partition                                              |
| `--source bootimg-partition`| Populate from kernel/DTB/bootloader files staged in DEPLOY_DIR    |
| `--source rootfs`           | Populate from the assembled root filesystem (the normal do_rootfs output)|
| `--ondisk`                  | Target disk device name reference (for bootloader configuration, not literally written by wic itself)|
| `--fstype`                  | Filesystem to format this partition as                              |
| `--label`                   | Filesystem volume label                                              |
| `--active`                  | Mark this partition as the bootable/active one (MBR partition tables)|
| `--align`                   | Alignment in KB (important for eMMC/SD performance — typically align to erase-block size)|
| `--size`                    | Fixed partition size (if omitted, sized to fit content + IMAGE_ROOTFS_EXTRA_SPACE)|
| `bootloader --ptable`       | Partition table type: `msdos` (MBR) or `gpt`                       |

### Using a wks File

```bitbake
WKS_FILE = "my-board.wks"
IMAGE_FSTYPES += "wic wic.bmap"
```

```bash
bitbake my-product-image
# Output: tmp/deploy/images/<machine>/my-product-image-<machine>.wic

# Write directly to an SD card (DESTRUCTIVE — verify device path carefully)
sudo bmaptool copy tmp/deploy/images/<machine>/my-product-image-<machine>.wic /dev/sdX
# bmaptool (using the companion .wic.bmap file) is dramatically faster than
# plain dd because it skips writing empty/sparse regions of the image
```

### Inspecting/Building Wic Images Manually (without a full BitBake image build)

```bash
# wic can also be invoked directly against an already-built rootfs for rapid
# partition-layout iteration without re-running the entire image build
wic create my-board.wks -e my-product-image
wic ls my-product-image-<machine>.wic
wic cp my-product-image-<machine>.wic:1/boot.scr ./extracted-boot.scr
```

---

## 13. Image Size Management

```bitbake
# Explicit fixed rootfs size in KB (if you want precise control, e.g., to
# exactly match a fixed eMMC partition size)
IMAGE_ROOTFS_SIZE = "1048576"     # 1 GB

# Extra headroom added on top of the computed/specified size, for runtime
# growth (logs, downloaded updates staged locally, application data)
IMAGE_ROOTFS_EXTRA_SPACE = "262144"   # +256 MB

# Multiplier applied to the raw computed content size before EXTRA_SPACE is
# added, accounting for filesystem metadata overhead (inodes, journal, etc.)
IMAGE_OVERHEAD_FACTOR = "1.3"
```

### Reducing Image Size

```bitbake
# Skip installing RRECOMMENDS entirely — often a significant size reduction,
# since RRECOMMENDS frequently pulls in optional-but-not-essential extras
NO_RECOMMENDATIONS = "1"

# Exclude entire classes of complementary packages globally
PACKAGE_EXCLUDE_COMPLEMENTARY = "1"
IMAGE_FEATURES:remove = "dev-pkgs dbg-pkgs doc-pkgs"

# Strip locale data down to only what's needed
IMAGE_LINGUAS = "en-us"

# Remove package manager + its database from the final image (if not needed
# for runtime package management — common for OTA-whole-image-update products)
IMAGE_FEATURES:remove = "package-management"

# Globally exclude specific known-bloated packages you don't need
PACKAGE_EXCLUDE = "perl python2"
```

### Analyzing What's Taking Up Space

```bash
# The manifest lists every package and lets you spot unexpectedly large entries
cat tmp/deploy/images/<machine>/<image>.manifest

# du-style breakdown of the actual rootfs content by directory
du -h --max-depth=2 tmp/work/<machine>/<image>/<ver>/rootfs/ | sort -rh | head -30
```

---

## 14. Read-Only and Read-Write Rootfs Strategies

```bitbake
IMAGE_FEATURES += "read-only-rootfs"
```

Enabling this `IMAGE_FEATURE` triggers a cascade of automatic adjustments throughout the build:
- Mounts the rootfs as read-only at boot (via fstab/init configuration)
- Reconfigures various daemons/applications that normally write to `/etc`, `/var`, or elsewhere on the rootfs to instead write to a separate writable location (commonly tmpfs or a dedicated writable data partition)
- Disables/adjusts package-manager post-install scripts that assume a writable root at every boot

### Why Read-Only Rootfs Matters for Embedded Products

- **Power-loss resilience:** A device that loses power mid-write to its root filesystem risks filesystem corruption requiring a full reflash; a read-only root eliminates this entire failure class for the base system
- **A/B OTA update schemes:** Pairs naturally with dual-partition (A/B) update strategies — the currently-running partition is mounted read-only and never touched; updates are written entirely to the inactive partition, then the bootloader switches which partition is "active"
- **Tamper resistance/integrity:** A read-only root, especially combined with a verified-boot/dm-verity setup, makes it much harder for malware or a misbehaving application to persistently modify core system files

### Common Companion Variable

```bitbake
# Where genuinely persistent, writable application data should live instead
# of the (now read-only) normal locations — typically a separate data partition
VOLATILE_LOG_DIR = "1"     # /var/log becomes a tmpfs (non-persistent logs)
```

---

## 15. Customizing the Rootfs with ROOTFS_POSTPROCESS_COMMAND

```bitbake
ROOTFS_POSTPROCESS_COMMAND += "remove_unwanted_files; set_custom_permissions; "

remove_unwanted_files() {
    rm -rf ${IMAGE_ROOTFS}/usr/share/man
    rm -rf ${IMAGE_ROOTFS}/usr/share/doc
}

set_custom_permissions() {
    chmod 600 ${IMAGE_ROOTFS}/etc/my-app/secrets.conf
    chown -R 1000:1000 ${IMAGE_ROOTFS}/home/appuser
}
```

`ROOTFS_POSTPROCESS_COMMAND` is a semicolon-separated list of shell function names, each run in sequence after the package manager has finished assembling the rootfs but before it gets converted into final image format(s) — the standard place for any "last mile" filesystem tweaks that don't naturally belong inside any individual package's own `do_install`.

### IMAGE_PREPROCESS_COMMAND — Before Rootfs Assembly

```bitbake
IMAGE_PREPROCESS_COMMAND += "generate_build_info; "

generate_build_info() {
    echo "Build date: $(date)" > ${WORKDIR}/build-info.txt
    echo "Git commit: ${SRCREV}" >> ${WORKDIR}/build-info.txt
}
```

---

## 16. Image Generation Hooks and Custom Tasks

```bitbake
# Add a completely custom task to the image build pipeline
do_generate_checksums() {
    cd ${IMGDEPLOYDIR}
    sha256sum ${IMAGE_NAME}.ext4 > ${IMAGE_NAME}.ext4.sha256
}
addtask generate_checksums after do_image_complete before do_build

# Inject extra build-time dependencies needed only for image assembly
# (e.g., a custom bootloader recipe that must complete before imaging)
EXTRA_IMAGEDEPENDS += "u-boot my-custom-flashing-tool-native"
```

---

## 17. SDK Generation from Images

Image recipes are also the basis for SDK generation — `bitbake <image> -c populate_sdk` produces a standalone cross-toolchain installer matched to exactly that image's package set and target architecture.

```bash
# Standard (non-extensible) SDK — cross-compiler + sysroot matching the image
bitbake core-image-minimal -c populate_sdk

# Extensible SDK (includes devtool + bundled sstate for fast iterative builds)
bitbake core-image-minimal -c populate_sdk_ext

# Output:
ls tmp/deploy/sdk/
poky-glibc-x86_64-core-image-minimal-cortexa72-toolchain-5.0.sh
```

```bitbake
# Customize what's included in the SDK's target sysroot beyond the image's
# own package set (e.g., extra -dev headers a typical app developer would need)
TOOLCHAIN_TARGET_TASK:append = " my-app-dev"
SDKIMAGE_FEATURES += "dev-pkgs dbg-pkgs"
```

---

## 18. Complete Worked Example — Custom Product Image

```bitbake
SUMMARY = "MyProduct Gateway — production firmware image"
DESCRIPTION = "Complete root filesystem for the MyProduct IoT Gateway device, \
including the gateway application, secure boot configuration, and OTA \
update client."
LICENSE = "MIT"

inherit core-image

# ── Package selection ──
IMAGE_INSTALL = "\
    packagegroup-core-boot \
    packagegroup-my-product-base \
    gateway-app \
    gateway-app-config \
    mender-client \
    dropbear \
    "

IMAGE_INSTALL:remove = "packagegroup-core-ssh-openssh"

# ── Feature toggles ──
IMAGE_FEATURES += "read-only-rootfs"
IMAGE_FEATURES:remove = "package-management dev-pkgs dbg-pkgs doc-pkgs"

# ── Size reduction ──
NO_RECOMMENDATIONS = "1"
IMAGE_LINGUAS = "en-us"
PACKAGE_EXCLUDE = "perl"

# ── Size management ──
IMAGE_ROOTFS_EXTRA_SPACE = "131072"
IMAGE_OVERHEAD_FACTOR = "1.25"

# ── Output formats ──
IMAGE_FSTYPES = "ext4 wic wic.bmap"
WKS_FILE = "my-product-gateway.wks"

# ── Post-processing ──
ROOTFS_POSTPROCESS_COMMAND += "strip_extra_docs; lockdown_permissions; "

strip_extra_docs() {
    rm -rf ${IMAGE_ROOTFS}/usr/share/man
    rm -rf ${IMAGE_ROOTFS}/usr/share/doc
    rm -rf ${IMAGE_ROOTFS}/usr/share/info
}

lockdown_permissions() {
    chmod 700 ${IMAGE_ROOTFS}/etc/my-product
    find ${IMAGE_ROOTFS}/etc/my-product -name '*.key' -exec chmod 600 {} \;
}

# ── Build provenance ──
IMAGE_PREPROCESS_COMMAND += "write_build_manifest; "

write_build_manifest() {
    echo "BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)" > ${WORKDIR}/build-manifest.txt
    echo "MACHINE=${MACHINE}" >> ${WORKDIR}/build-manifest.txt
    echo "DISTRO_VERSION=${DISTRO_VERSION}" >> ${WORKDIR}/build-manifest.txt
}
```

### A Companion Development Variant

```bitbake
# my-product-gateway-image-dev.bb
require my-product-gateway-image.bb

SUMMARY = "MyProduct Gateway — development image (debug-enabled)"

IMAGE_FEATURES += "debug-tweaks tools-debug ssh-server-openssh package-management"
IMAGE_FEATURES:remove = "read-only-rootfs"

IMAGE_INSTALL:append = " gdb strace tcpdump htop"
```

---

## 19. Common Pitfalls and Debugging

### 1. IMAGE_INSTALL:append missing the leading space
`IMAGE_INSTALL:append = "mypackage"` (no leading space) silently concatenates onto the previous word, producing `...corepackagegroupmypackage` — a non-existent package name that fails dependency resolution with a confusing error. Always write `IMAGE_INSTALL:append = " mypackage"` with the leading space.

### 2. A package compiles fine but is mysteriously absent from the final image
Check it's actually listed (directly or via a packagegroup) in `IMAGE_INSTALL`, and check it's not being pulled out by `PACKAGE_EXCLUDE` or accidentally matched by an overly broad `IMAGE_INSTALL:remove` pattern.

### 3. Image build fails with unresolvable RDEPENDS during do_rootfs
This is the package manager's dependency resolver failing — almost always means some package's `RDEPENDS` references a package name that doesn't exist anywhere in the currently configured build (a typo, a package that exists in a layer you haven't enabled, or a package only built for a different `MACHINE`/architecture). Check the exact error for which package's RDEPENDS couldn't be satisfied.

### 4. read-only-rootfs feature breaks an application that writes to /etc at runtime
Any application/daemon expecting to persistently write configuration changes to `/etc` (rather than treating `/etc` as read-only baseline config) needs explicit accommodation — either redirect its writable state to a separate persistent data partition, or that daemon genuinely isn't compatible with a read-only-root design without modification.

### 5. WIC image boots but is the wrong size / partition too small
Check `--size` in the `.wks` file against actual rootfs content size (`IMAGE_ROOTFS_SIZE`/`IMAGE_ROOTFS_EXTRA_SPACE`) — if the rootfs partition's fixed `--size` is smaller than the actual assembled rootfs content, wic will fail (or worse, silently truncate, depending on version) rather than auto-growing, since you explicitly requested a fixed size.

### 6. Image works on QEMU but not on real hardware
Frequently a missing `EXTRA_IMAGEDEPENDS` for a bootloader/firmware blob that QEMU doesn't need but real hardware does, or a `.wks` partition layout assuming a disk geometry/boot-ROM requirement specific to real hardware that QEMU's generic virtual disk doesn't enforce. Compare against your BSP layer's reference image recipe for the exact required `EXTRA_IMAGEDEPENDS`/`WKS_FILE` settings.

### 7. Locale-related size bloat despite trying to minimize
`IMAGE_LINGUAS` only controls locale data shipped by individual packages that respect it — some packages (especially those not properly following the standard `glibc-locale` packaging convention) may still pull in larger locale support through their own `RDEPENDS`. Check the manifest for unexpectedly large locale-related packages specifically.

---

## 20. Interview Questions and Answers

**Q1: What is the fundamental conceptual difference between a normal recipe and an image recipe, in terms of what each one "builds"?**

**A:** A normal recipe builds ONE piece of software — it fetches that software's specific source code, compiles it, and packages the result into one or more installable binary packages (.rpm/.deb/.ipk). Its scope is a single project's build process. An image recipe builds nothing of its own in that sense — it has essentially no source code or compile step at all. Instead, its job is SELECTION and ASSEMBLY: given a list of already-built packages (specified via `IMAGE_INSTALL`, each of which came from its own separate normal recipe having already run through the full fetch/compile/package pipeline), the image recipe's `do_rootfs` task invokes a package manager to resolve the full dependency closure of that selection and install everything into a fresh filesystem tree, and then `do_image`/`do_image_<fstype>` tasks convert that assembled filesystem tree into final, deployable image file(s) — an ext4 filesystem image, a tarball, a fully partitioned bootable disk image, etc. In short: normal recipes answer "how do I build this ONE piece of software," while image recipes answer "given everything that's already been built, which subset of it constitutes a complete, bootable system, and in what final file format should that system be delivered."

---

**Q2: Explain the difference between IMAGE_FEATURES and DISTRO_FEATURES, and give an example where conflating them would cause confusion.**

**A:** `DISTRO_FEATURES` is set once at the distribution-configuration level and determines what CAPABILITIES GET COMPILED INTO recipes throughout the entire build, before any specific image is even being assembled — many recipes contain conditional logic (often via `PACKAGECONFIG` checking `bb.utils.filter('DISTRO_FEATURES', ...)`) that only builds in support for a given capability (Wayland support, Bluetooth support, systemd integration) if that feature name is present in `DISTRO_FEATURES`. `IMAGE_FEATURES`, by contrast, operates entirely at the level of one specific image recipe and only controls which ALREADY-BUILT packages get INSTALLED into that particular image — it cannot retroactively add compiled-in capability to a package that wasn't built with that capability enabled at the distro level. The confusion this causes in practice: a developer adds `IMAGE_FEATURES += "wayland-support-toggle-xyz"` (hypothetically) to their image recipe expecting Wayland support to suddenly work, but if `DISTRO_FEATURES` at the distro-configuration level never included "wayland" in the first place, the underlying graphics stack packages were never compiled with Wayland support at all — no amount of IMAGE_FEATURES tweaking on a single image can retroactively grant a capability the packages themselves were never built to support. Fixing this requires changing `DISTRO_FEATURES` (a build-wide, often very expensive — full rebuild of affected recipes — change), not just the one image recipe.

---

**Q3: Why does Yocto use a real package manager (rpm/dpkg/opkg) during do_rootfs even for a fully read-only, non-package-manageable final image?**

**A:** Using a real package manager during `do_rootfs`, even when the final product will have NO package manager left on the running target (`package-management` IMAGE_FEATURE omitted) and the rootfs will be entirely read-only, provides correct, well-tested DEPENDENCY RESOLUTION at build time, which is genuinely hard to reimplement correctly from scratch — the package manager handles resolving the full transitive closure of every package's `RDEPENDS`/`RRECOMMENDS`, detecting and reporting conflicts (`RCONFLICTS`), and respecting version constraints, exactly the same battle-tested logic that desktop Linux distributions rely on for the same purpose. After `do_rootfs` finishes correctly resolving and installing this full dependency closure into the rootfs tree, the package manager's own runtime database/binaries can then simply be DELETED from the final rootfs (which is exactly what omitting `package-management` from `IMAGE_FEATURES` effectively arranges) — you get the benefit of correct, automated dependency resolution at BUILD time, without paying the runtime cost (disk space for the package database, the package manager binary itself, and the attack surface of having a package manager present) on the deployed device. This is conceptually similar to how a compiled language's build-time dependency resolver (e.g., npm/cargo resolving a dependency tree) doesn't need to remain present in the final compiled binary's runtime environment.

---

**Q4: What is a packagegroup recipe and why is it preferable to listing dozens of individual packages directly in IMAGE_INSTALL?**

**A:** A `packagegroup-*` recipe is a thin, typically near-empty recipe (inheriting `packagegroup.bbclass`) whose entire purpose is to declare a curated list of other packages via `RDEPENDS` — when included in `IMAGE_INSTALL`, the package manager pulls in that whole RDEPENDS list automatically, just as installing any package pulls in its dependencies. The value of this indirection is centralization and reuse: if "what does a MyProduct device's base system need" is defined ONCE inside `packagegroup-my-product-base`, then every image variant that needs that same base feature set (a production image, a development image, a manufacturing-test image, a recovery image) references it with one line, and any future change to that base set (adding a new required package, removing a deprecated one) is made in exactly one place and automatically propagates to every image that includes that packagegroup — versus having to find-and-edit the same long package list duplicated across N separate image recipe files, where it's easy to update one and forget another, leading to inconsistent images. This is precisely the same "don't repeat yourself" reasoning behind shared libraries/modules in regular software engineering, applied to the recipe-selection level of an embedded Linux build.

---

**Q5: Walk through what happens if you set IMAGE_ROOTFS_SIZE smaller than what the selected packages actually require.**

**A:** `IMAGE_ROOTFS_SIZE` (whether explicitly set or auto-computed from the actual installed content size plus `IMAGE_OVERHEAD_FACTOR` and `IMAGE_ROOTFS_EXTRA_SPACE`) determines the SIZE of the filesystem image `do_image_<fstype>` creates — but `do_rootfs` has already, in the PRECEDING task, installed the full actual content into `${IMAGE_ROOTFS}` on the build host's disk, which has no inherent size limit beyond available build-host disk space. If you then explicitly hard-code `IMAGE_ROOTFS_SIZE` to a value smaller than that already-assembled content actually occupies, the filesystem-creation step (e.g., `mkfs.ext4` being told to create a filesystem of that explicit size, then trying to populate it from the larger source tree) will fail with a clear "no space left on device"-style error during image creation — it does not silently truncate or drop files. This is a build-time failure, not a silent runtime corruption risk, which is the safer failure mode. The fix is either to increase `IMAGE_ROOTFS_SIZE`/`IMAGE_ROOTFS_EXTRA_SPACE`, or — more often the right fix — reduce the actual package selection/content (removing genuinely unneeded packages, doc/dev/dbg variants, locale data) if the goal was to fit within a specific fixed eMMC/flash partition size constraint that the current package selection simply doesn't fit within.
