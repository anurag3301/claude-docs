# Yocto Project Build System — Architecture Reference Guide

---

## Table of Contents

1. [Introduction — Yocto Project vs Poky vs OpenEmbedded](#1-introduction--yocto-project-vs-poky-vs-openembedded)
2. [The Component Stack](#2-the-component-stack)
3. [Setting Up a Build Environment](#3-setting-up-a-build-environment)
4. [The build/ Directory Structure](#4-the-build-directory-structure)
5. [The tmp/ Directory Deep Dive](#5-the-tmp-directory-deep-dive)
6. [DISTRO Configuration](#6-distro-configuration)
7. [The Yocto Release Model and Codenames](#7-the-yocto-release-model-and-codenames)
8. [Build Performance — Parallelism and Caching](#8-build-performance--parallelism-and-caching)
9. [Cross-Compilation Toolchain Architecture](#9-cross-compilation-toolchain-architecture)
10. [Source Mirrors and Network Independence](#10-source-mirrors-and-network-independence)
11. [Reproducible Builds](#11-reproducible-builds)
12. [Testing Infrastructure — ptest, testimage, and OEQA](#12-testing-infrastructure--ptest-testimage-and-oeqa)
13. [License Manifest and Compliance Tooling](#13-license-manifest-and-compliance-tooling)
14. [CVE Scanning](#14-cve-scanning)
15. [Continuous Integration Considerations](#15-continuous-integration-considerations)
16. [Common Pitfalls and Debugging](#16-common-pitfalls-and-debugging)
17. [Interview Questions and Answers](#17-interview-questions-and-answers)

---

## 1. Introduction — Yocto Project vs Poky vs OpenEmbedded

These three names are frequently (and incorrectly) used interchangeably; understanding the actual relationship between them clarifies a lot of otherwise-confusing documentation and terminology.

| Name                  | What It Actually Is                                                  |
|--------------------------|------------------------------------------------------------------|
| **OpenEmbedded**         | The original, broader community project building embedded Linux distributions using BitBake. Predates the Yocto Project. Produces **OpenEmbedded-Core (oe-core)**, the foundational `meta` layer with base recipes/classes.|
| **Yocto Project**        | A Linux Foundation collaborative project (started 2010) that took BitBake + OpenEmbedded-Core and added project governance, additional tooling (devtool, the eSDK, documentation, testing infrastructure, certification programs), and produces the **Poky** reference distribution as a complete, ready-to-use example/starting point.|
| **Poky**                 | The actual reference distribution/build system you download and use day-to-day (`git clone https://git.yoctoproject.org/poky`) — it bundles BitBake + oe-core (`meta`) + a minimal reference distro policy layer (`meta-poky`) + reference machine/BSP support (`meta-yocto-bsp`) into one cohesive, immediately buildable starting point.|

```
Yocto Project (governance, branding, certification, broader tooling)
   │
   └── produces/maintains → Poky (the reference distribution you actually clone and build)
                                │
                                ├── BitBake (the task-execution engine, also independently usable)
                                ├── meta/        ← OpenEmbedded-Core (oe-core), the foundational layer
                                ├── meta-poky/   ← Poky's own minimal distro policy
                                └── meta-yocto-bsp/ ← reference/QEMU machine support
```

**In casual conversation**, "I'm using Yocto" almost always means "I'm using Poky (or a similar OpenEmbedded-based build setup) to build a custom embedded Linux distribution" — the terms have become practically synonymous in everyday usage even though they technically refer to different things in the formal architecture above.

---

## 2. The Component Stack

```
┌──────────────────────────────────────────────────────────┐
│  Your Product's Layers (BSP, application, distro policy)  │
├──────────────────────────────────────────────────────────┤
│  Community Layers (meta-openembedded, meta-qt5, etc.)     │
├──────────────────────────────────────────────────────────┤
│  meta-yocto-bsp / vendor BSP layers (reference hardware)  │
├──────────────────────────────────────────────────────────┤
│  meta-poky (reference distro policy)                      │
├──────────────────────────────────────────────────────────┤
│  meta / OpenEmbedded-Core (oe-core)                        │
│  — base classes, core recipes, fundamental build logic     │
├──────────────────────────────────────────────────────────┤
│  BitBake                                                    │
│  — generic task-execution engine, metadata parser           │
└──────────────────────────────────────────────────────────┘
```

Each layer in this stack depends only on the layer(s) below it, never above — `meta` (oe-core) has no knowledge of or dependency on any BSP or product layer; BitBake has no knowledge of or dependency on `meta` at all (it's a fully generic tool that happens to be the engine OpenEmbedded chose to build on).

---

## 3. Setting Up a Build Environment

```bash
# 1. Clone Poky (pick the branch matching your desired release)
git clone https://git.yoctoproject.org/poky -b scarthgap
cd poky

# 2. Optionally clone additional community layers
git clone https://git.openembedded.org/meta-openembedded -b scarthgap
git clone https://github.com/agherzan/meta-raspberrypi -b scarthgap

# 3. Source the environment setup script — this is REQUIRED every new shell
#    session before running any bitbake command
source oe-init-build-env build

# This script:
#   - Sets up PATH to include bitbake and related tools
#   - Creates (if not already present) build/conf/local.conf and
#     build/conf/bblayers.conf from templates
#   - Changes the current directory to build/
#   - Sets the TEMPLATECONF environment variable if a custom distro
#     template is being used

# 4. Add any additional layers
bitbake-layers add-layer ../meta-openembedded/meta-oe
bitbake-layers add-layer ../meta-raspberrypi

# 5. Edit build/conf/local.conf to set MACHINE, DISTRO, etc.
vim conf/local.conf
# MACHINE = "raspberrypi4"

# 6. Build
bitbake core-image-minimal
```

### TEMPLATECONF — Custom Default Configuration Templates

```bash
# A company can ship its own default local.conf/bblayers.conf TEMPLATES
# (rather than every developer hand-editing the same boilerplate every time)
TEMPLATECONF=../meta-mycompany/conf/templates/default source oe-init-build-env build

# meta-mycompany/conf/templates/default/local.conf.sample
# meta-mycompany/conf/templates/default/bblayers.conf.sample
# meta-mycompany/conf/templates/default/conf-notes.txt
```

This is the standard mechanism for distributing consistent, pre-configured build environments across a development team — new developers (or fresh CI runners) get correct `MACHINE`/`DISTRO`/layer settings automatically rather than needing tribal-knowledge manual setup instructions.

---

## 4. The build/ Directory Structure

```
build/
├── conf/
│   ├── local.conf              ← build-host-specific settings (MACHINE, parallelism, etc.)
│   ├── bblayers.conf           ← active layer list
│   └── templateconf.cfg         ← records which TEMPLATECONF was used to generate this build dir
│
├── cache/                       ← parsed-metadata cache (speeds up subsequent bitbake invocations)
│
├── downloads/                   ← DL_DIR: all fetched upstream source tarballs/git clones (shared, valuable to back up/cache across builds)
│
├── sstate-cache/                ← SSTATE_DIR: the shared-state cache (also extremely valuable to preserve/share)
│
├── tmp/                          ← TMPDIR: ALL actual build output — safe to delete entirely
│   ├── work/                     ← per-recipe build directories
│   ├── deploy/                   ← final images, packages, SDK installers
│   ├── sysroots-components/      ← staged sysroot content
│   ├── stamps/                   ← task completion markers (BitBake's "has this run" tracking)
│   └── log/                      ← cooker/parsing logs
│
└── workspace/                    ← devtool's workspace layer (only present if devtool has been used)
```

### What's Safe to Delete and What's Not

| Directory          | Safe to delete?                                                       |
|------------------------|----------------------------------------------------------------|
| `tmp/`                | **Yes, always safe** — entirely regeneratable from sstate-cache + downloads. Deleting it and rebuilding is a common, fast way to get a "clean" build (since sstate-cache means it's NOT actually a full from-scratch rebuild).|
| `downloads/`           | Safe but WASTEFUL to delete — losing it means every upstream source must be re-fetched from the network on the next build, which can be slow/fragile if upstream sources become unavailable later.|
| `sstate-cache/`        | Safe but WASTEFUL to delete — losing it means every task must genuinely re-execute from scratch (no accelerated "setscene" restoration), turning a 20-minute incremental rebuild into a multi-hour full rebuild.|
| `cache/`               | Safe to delete — just means the next bitbake invocation re-parses all metadata (slower startup, but otherwise harmless).|
| `conf/`                | **Do NOT delete carelessly** — contains your actual configuration (MACHINE, DISTRO, layer list); deleting this loses your build configuration, not just build output.|

```bash
# The common "give me a clean tmp but keep my expensive caches" pattern:
rm -rf tmp/
bitbake core-image-minimal
# Much faster than a truly first-time build, because sstate-cache and
# downloads are both preserved
```

---

## 5. The tmp/ Directory Deep Dive

```
tmp/work/<package_arch>/<recipe_name>/<version>-<revision>/
├── temp/
│   ├── log.do_fetch
│   ├── log.do_compile
│   ├── log.do_install
│   ├── run.do_compile          ← the ACTUAL generated shell script BitBake executed
│   └── ...
├── <source-dir>/                ← ${S}
├── <build-dir>/                  ← ${B} (if different from S)
├── image/                         ← ${D}
├── sysroot-destdir/                ← staged content for do_populate_sysroot
├── recipe-sysroot/                  ← THIS recipe's view of its DEPENDS' staged output
├── recipe-sysroot-native/            ← native (build-host-architecture) tool dependencies
└── pseudo/                            ← pseudo's fake-root database for this recipe (see below)
```

### tmp/work/<package_arch> — Why Multiple Architecture Subdirectories Exist

```
tmp/work/cortexa53-poky-linux/        ← normal target-architecture recipes
tmp/work/x86_64-linux/                 ← -native recipes (built to run on the build host)
tmp/work/corei7-64-poky-linux/         ← (example) a different machine's target arch, if multiple MACHINE builds share this build/ directory via multiconfig
tmp/work/allarch-poky-linux/           ← architecture-independent recipes (inherit allarch.bbclass)
```

This separation exists because a single `build/` directory's `tmp/work/` can simultaneously contain build output for several genuinely different "architectures" in the cross-compilation sense (the real target CPU, the build host's own native architecture for `-native` tools, and a special `allarch` bucket for recipes like pure-data/config packages that have no compiled, architecture-specific content at all) — keeping them in separate subdirectories prevents any possibility of a native tool's build output being confused with target-architecture output.

### pseudo — Fakeroot for Correct File Ownership

Because BitBake never actually runs as root (a deliberate, important security/safety property — you should never need root privileges to build an embedded Linux image), but installed root filesystem content legitimately needs files owned by various UIDs/GIDs (root-owned `/etc/shadow`, specific service-account-owned files, special device nodes), Yocto uses **pseudo** — a `LD_PRELOAD`-based tool that intercepts filesystem syscalls (`chown`, `chmod`, `mknod`, etc.) during `do_install`/`do_rootfs` and maintains a separate database recording "this file should APPEAR to be owned by root/UID-X" without ever actually requiring real root privileges on the build host.

```bash
# Every recipe's do_install effectively runs wrapped in pseudo:
pseudo bash -c 'install -d ${D}${sysconfdir}; chown root:root ${D}${sysconfdir}'
# Even though the actual build-host user doing this isn't root, pseudo's
# database correctly records that this file SHOULD be root-owned when the
# rootfs is finally assembled/packaged
```

---

## 6. DISTRO Configuration

While `MACHINE` configuration (covered in the BSP Layer document) defines hardware-specific build parameters, `DISTRO` configuration defines **distribution-wide policy** — package format choice, default feature set, security posture, and general "what kind of Linux distribution is this" decisions independent of any specific hardware target.

```bitbake
# conf/distro/my-product-distro.conf

DISTRO = "my-product-distro"
DISTRO_NAME = "MyProduct Embedded Linux"
DISTRO_VERSION = "1.0"

DISTRO_FEATURES = "systemd usrmerge wifi bluetooth ext2 vfat \
                    largefile ipv4 ipv6 pci usbhost"
DISTRO_FEATURES_BACKFILL_CONSIDERED = "sysvinit"

VIRTUAL-RUNTIME_init_manager = "systemd"
VIRTUAL-RUNTIME_initscripts = ""
VIRTUAL-RUNTIME_login_manager = "shadow-base"

PACKAGE_CLASSES = "package_rpm"

PREFERRED_VERSION_linux-yocto ?= "6.6%"

# Security posture defaults
DISTRO_FEATURES:append = " pam"
INIT_MANAGER = "systemd"

# Override the default Poky distro's hostname/branding throughout the build
DISTRO_CODENAME = "myproduct-v1"
SDK_VENDOR = "-mycompanysdk"
SDK_VERSION = "${DISTRO_VERSION}"
```

```bitbake
# In local.conf:
DISTRO = "my-product-distro"
```

### poky.conf — Studying the Reference Implementation

The best way to understand what a real, complete distro configuration looks like is reading `meta-poky/conf/distro/poky.conf` directly — it's deliberately kept as a relatively minimal, well-commented reference implementation that most custom distro configurations either directly base on (`require conf/distro/poky.conf` then override specific bits) or use as a structural template.

```bitbake
# A common, pragmatic pattern — inherit Poky's sensible defaults, override
# only what genuinely needs to differ for your product:
require conf/distro/poky.conf

DISTRO = "my-product-distro"
DISTRO_NAME = "MyProduct Embedded Linux"

# Override specific policy decisions:
PACKAGE_CLASSES = "package_ipk"
DISTRO_FEATURES:remove = "x11 wayland"
```

---

## 7. The Yocto Release Model and Codenames

Yocto Project releases follow a roughly twice-yearly cadence, each given an alphabetically-sequential codename (a tradition similar to Ubuntu's naming scheme), with Long Term Support (LTS) designation for certain releases that receive extended security/bug-fix maintenance.

```
... → kirkstone (LTS) → mickledore → nanbield → scarthgap (LTS) → styhead → walnascar → ...
```

```bash
# Always clone/checkout the specific release branch matching your project's
# chosen support window, rather than tracking a moving branch like "master"
# for any production work
git clone https://git.yoctoproject.org/poky -b scarthgap
```

### Why LTS Releases Matter for Production Products

A production embedded product with a multi-year field deployment lifespan typically standardizes on an LTS release specifically because: (1) it receives security patches and critical bug fixes for an extended period (historically around 4 years for Yocto LTS releases) without requiring a disruptive full version upgrade, and (2) the broader layer ecosystem (BSP layers, community software layers) tends to maintain compatibility branches against LTS releases for longer, since that's where the bulk of production usage concentrates.

```bash
# Checking the Yocto Project's published release/LTS schedule is standard
# practice before committing a long-lived product to a specific release
```

---

## 8. Build Performance — Parallelism and Caching

```bitbake
# In local.conf:

# How many TASKS BitBake runs in parallel across different recipes
BB_NUMBER_THREADS = "16"

# How many parallel jobs EACH individual recipe's own build system (make -j)
# should use
PARALLEL_MAKE = "-j 16"

# A reasonable starting point for both is the number of available CPU cores
# (or threads, if hyperthreading), though BB_NUMBER_THREADS can sometimes
# usefully exceed core count since many tasks (do_fetch in particular) are
# I/O-bound rather than CPU-bound
```

### Disk I/O Considerations

```bitbake
# tmpfs-backed TMPDIR can dramatically speed up I/O-heavy builds if enough
# RAM is available (the entire build's working set lives in RAM instead of
# disk) — a common technique on well-resourced build servers:
# (set up via OS-level tmpfs mount, then point TMPDIR at it)
TMPDIR = "/dev/shm/yocto-build-tmp"
```

### rm_work — Reclaiming Disk Space During Long Builds

```bitbake
INHERIT += "rm_work"

# Exempt specific recipes from automatic work-directory cleanup (e.g., ones
# you're actively iterating on/debugging and want to inspect after the build)
RM_WORK_EXCLUDE += "my-app-under-development virtual/kernel"
```

`rm_work.bbclass` automatically deletes each recipe's `tmp/work/.../<recipe>/` build directory immediately after that recipe's `do_build` completes successfully — since the recipe's USEFUL output is already safely captured in `sstate-cache` and the final package feed, the raw build-intermediate files serve no further purpose for a successful build, and deleting them as you go (rather than only at the very end) keeps peak disk usage dramatically lower for large, many-recipe builds — important since a full Yocto build of a graphical image can otherwise consume well over 100GB of `tmp/work/` content if left uncleaned.

---

## 9. Cross-Compilation Toolchain Architecture

Every Yocto build constructs its OWN complete cross-compilation toolchain from source (GCC, binutils, glibc/musl, etc.) rather than relying on a pre-built external toolchain — ensuring exact reproducibility and consistency between the toolchain and the target's actual C library/kernel headers.

```
gcc-cross-${TARGET_ARCH}     ← the actual cross C/C++ compiler
binutils-cross-${TARGET_ARCH}← cross assembler, linker, etc.
glibc (or musl)                ← target C library, cross-compiled for the target
linux-libc-headers              ← kernel headers matching the target's kernel version, used during glibc/application builds
gcc-cross-canadian-*            ← (SDK-specific) a "Canadian Cross" compiler: runs on the SDK host, targets the embedded target — distinct from gcc-cross, which runs on the BUILD host
```

### The Canadian Cross (Three-System Build)

```
Build machine (compiles the compiler)
   │
   ▼
Host machine (where the resulting compiler will RUN — e.g., a developer's SDK-installed laptop)
   │
   ▼
Target machine (what the resulting compiler PRODUCES CODE FOR — the embedded device)
```

When build ≠ host ≠ target (all three genuinely different machines — building on a Linux CI server, producing a compiler meant to run on a developer's possibly-different-architecture laptop inside an installed SDK, which itself targets a third, embedded architecture), this is termed a "Canadian Cross" build — long-standing GNU toolchain terminology referencing an era when three distinct national/political affiliations were jokingly invoked to describe the three-system relationship; the name has simply stuck in toolchain engineering ever since. `gcc-cross-canadian` recipes specifically handle this SDK-toolchain-generation scenario.

```bash
# Inspecting the actual constructed toolchain for a build:
ls tmp/sysroots-components/x86_64/gcc-cross-aarch64/usr/bin/
aarch64-poky-linux-gcc
aarch64-poky-linux-g++
aarch64-poky-linux-ld
```

---

## 10. Source Mirrors and Network Independence

For reproducibility, air-gapped build environments, and protection against upstream source disappearing (a real, recurring problem — projects move hosting, get taken down, or simply vanish), Yocto strongly supports mirroring all fetched sources.

```bitbake
# Generate a complete local mirror of every source this build needs
BB_GENERATE_MIRROR_TARBALLS = "1"

# Point at a pre-populated local/internal mirror BEFORE attempting the
# original upstream SRC_URI location
PREMIRRORS = "git://.*/.* http://mirror.mycompany.internal/sources/ \;downloadfilename=${BB_GENERATE_MIRROR_TARBALLS} \
              https?://.*/.* http://mirror.mycompany.internal/sources/ \;downloadfilename=${BB_GENERATE_MIRROR_TARBALLS}"

# Fall back to additional mirrors only if the primary SRC_URI location fails
MIRRORS += "https://.*/.* http://backup-mirror.mycompany.internal/sources/"

# For a build environment with NO internet access at all, once downloads/
# is fully pre-populated (e.g., copied from a build that DID have network
# access), this forces BitBake to NEVER attempt any network fetch, failing
# loudly instead if anything is missing rather than hanging on a network timeout:
BB_NO_NETWORK = "1"
```

```bash
# Generating a portable, complete source archive for offline/air-gapped use
# (common for safety-certified or export-controlled environments needing a
# fully self-contained, auditable build):
bitbake core-image-minimal --runall=fetch
tar czf complete-source-mirror.tar.gz downloads/
```

---

## 11. Reproducible Builds

A reproducible build means: building the exact same metadata/source twice (potentially on different machines, at different times) produces BIT-FOR-BIT IDENTICAL output binaries/packages/images. This matters enormously for security auditing (independently verifying that a shipped binary actually corresponds to its claimed source) and for build-system trust generally.

```bitbake
# Modern Yocto releases default to reproducible builds for most content via:
INHERIT += "reproducible_build"

# Key mechanisms involved:
#  - Build timestamps replaced with a fixed, deterministic reference
#    (commonly derived from SOURCE_DATE_EPOCH, a widely-adopted cross-project
#    standard for reproducible build timestamp handling)
#  - File ordering/timestamps within generated archives normalized
#  - Removal of build-host-specific paths from compiled output (debug info)
```

```bash
# Verifying reproducibility in practice: build twice (ideally on two
# different machines/at different times), compare:
sha256sum tmp/deploy/images/<machine>/core-image-minimal-<machine>.ext4
# Both builds should produce identical hashes if everything genuinely is
# reproducible end-to-end
```

In practice, achieving FULL reproducibility across an entire complex image (every package, every tool used to build them) is an ongoing, actively-improving effort across the broader Linux/open-source ecosystem (not unique to Yocto) — some specific packages/build steps remain known exceptions, but the trend and tooling support has improved substantially in recent Yocto releases.

---

## 12. Testing Infrastructure — ptest, testimage, and OEQA

### ptest — Package-Level Test Suites

```bitbake
inherit ptest

# Most ptest-enabled recipes provide a run-ptest script invoking the
# upstream project's OWN test suite
do_install_ptest() {
    install -d ${D}${PTEST_PATH}
    cp -r ${S}/tests/* ${D}${PTEST_PATH}/
}
```

`ptest` packages a piece of software's own upstream test suite as a SEPARATE, optional `-ptest` package, installable onto a target device specifically to run that software's real test suite IN the actual target environment (catching cross-compilation-specific or target-environment-specific bugs that would never show up in a build-host-only test run).

```bash
# On the target, after installing the relevant -ptest packages:
ptest-runner
# Runs every installed package's bundled test suite, reporting pass/fail
```

### testimage — Whole-Image Runtime Testing

```bitbake
# In local.conf or an image recipe:
IMAGE_CLASSES += "testimage"
TEST_TARGET = "qemu"
TEST_SUITES = "ping ssh systemd parselogs"
```

```bash
bitbake core-image-minimal -c testimage
# Boots the actual built image (in QEMU, or on real hardware if TEST_TARGET
# is configured for a physical device controller), then runs the listed
# OEQA test suites against it — verifying the image genuinely boots, accepts
# SSH connections, has a working systemd, has no unexpected error patterns
# in the boot log, etc.
```

### OEQA — The Underlying Test Framework

OEQA (OpenEmbedded Quality Assurance) is the Python-based test framework underlying `testimage` and several other Yocto testing mechanisms — custom test cases can be written following its conventions and added to `TEST_SUITES` for product-specific automated validation (e.g., a custom test verifying your specific application starts correctly and responds on its expected network port).

```python
# meta-mylayer/lib/oeqa/runtime/cases/myapp.py
from oeqa.runtime.case import OERuntimeTestCase

class MyAppTest(OERuntimeTestCase):
    def test_myapp_running(self):
        status, output = self.target.run('systemctl is-active myapp')
        self.assertEqual(status, 0, msg="myapp service is not running: %s" % output)
```

```bitbake
TEST_SUITES:append = " myapp"
```

---

## 13. License Manifest and Compliance Tooling

```bash
# Automatically generated for every image build (no special configuration
# needed beyond what's already implicit in core-image.bbclass):
cat tmp/deploy/licenses/<image-name>-<machine>-<datetime>/license.manifest
```

This file lists EVERY package included in the image, its declared `LICENSE`, and the package's recipe name — the canonical, audit-ready record of exactly what licenses are present in a specific built image.

```bitbake
# Generate a complete archive of every recipe's actual license text files
# (not just the manifest listing license NAMES, but the full source text
# of each license itself, as referenced by LIC_FILES_CHKSUM) — important
# for compliance regimes requiring the actual license text be redistributed
# alongside a product (a common GPL/LGPL obligation):
COPY_LIC_MANIFEST = "1"
COPY_LIC_DIRS = "1"
LICENSE_CREATE_PACKAGE = "1"
```

```bash
ls tmp/deploy/licenses/
# Per-recipe subdirectories, each containing the exact license text file(s)
# that recipe's LIC_FILES_CHKSUM checksums were computed against
```

### Generating an SPDX SBOM (Software Bill of Materials)

```bitbake
INHERIT += "create-spdx"
```

Modern Yocto releases include built-in SPDX (Software Package Data Exchange) SBOM generation — producing a machine-readable, industry-standard bill-of-materials document for every image, listing every component, its exact version, its license, and (where available) source provenance information. This has become increasingly important for software supply-chain security/compliance requirements (e.g., regulatory requirements in some industries/jurisdictions now mandate SBOM availability for shipped software products).

```bash
ls tmp/deploy/spdx/<machine>/
core-image-minimal-<machine>.spdx.json
```

---

## 14. CVE Scanning

```bitbake
INHERIT += "cve-check"

# Optionally restrict scanning/reporting to specific severity thresholds
CVE_CHECK_SHOW_WARNINGS = "1"
CVE_CHECK_SKIP_RECIPE = "some-recipe-with-a-known-false-positive"
```

```bash
bitbake core-image-minimal
# With cve-check inherited, produces a report of every known CVE (matched
# against the NVD — National Vulnerability Database — by package name/version)
# potentially affecting any recipe in the build:

cat tmp/deploy/cve/cve-summary.json
```

### CVE_PRODUCT — Correcting Recipe-to-CVE-Database Name Mapping

```bitbake
# When a recipe's PN doesn't exactly match the CVE database's product name
# for that software, CVE_PRODUCT corrects the mapping so scanning actually
# finds the relevant CVEs (otherwise silently scans against the WRONG/no
# product entry, producing false confidence that nothing was found):
CVE_PRODUCT = "openssl:openssl"
```

This is a critical, easy-to-overlook detail — without correct `CVE_PRODUCT` mapping for recipes whose `PN` doesn't naturally match their CVE database product identifier, `cve-check` can silently report "0 CVEs found" for a recipe simply because it never actually matched anything in the database, giving a false sense of security rather than a genuine clean bill of health.

```bash
# Reviewing CVE scan results is a standard, often contractually/regulatorily
# required step before any production release of an embedded Linux product,
# given the multi-year field deployment lifespans typical of embedded devices
# and the corresponding multi-year window during which newly discovered
# CVEs in bundled software components remain a live concern.
```

---

## 15. Continuous Integration Considerations

### Caching Strategy for CI

```bitbake
# A CI pipeline building the same product repeatedly (every commit/PR)
# should persist BOTH of these caches across CI runs for acceptable build
# times — without them, every CI run is effectively a multi-hour
# from-scratch build:
DL_DIR = "/persistent-ci-cache/downloads"
SSTATE_DIR = "/persistent-ci-cache/sstate-cache"
```

### Shared sstate-cache Across a Build Farm

```bitbake
# A network-shared sstate mirror lets MULTIPLE CI runners (and developer
# machines) all benefit from each other's already-completed task output,
# rather than each runner needing its own independent, redundant cache:
SSTATE_MIRRORS = "file://.* http://sstate.mycompany.internal/sstate-cache/PATH;downloadfilename=PATH"
```

### Parallel CI Builds of Multiple MACHINE/Image Combinations — multiconfig

```bitbake
# A single bitbake invocation can build MULTIPLE machine/image combinations
# in one pass, sharing common dependency builds (toolchain, shared
# architecture-independent recipes) rather than running entirely separate
# bitbake invocations per machine:

# conf/multiconfig/board-a.conf
MACHINE = "board-a"

# conf/multiconfig/board-b.conf
MACHINE = "board-b"
```

```bitbake
# In local.conf:
BBMULTICONFIG = "board-a board-b"
```

```bash
bitbake mc:board-a:core-image-minimal mc:board-b:core-image-minimal
# Builds BOTH machine targets in one bitbake invocation, sharing sstate
# for any genuinely common (e.g., allarch, or coincidentally
# same-architecture) build output between them
```

### Minimal, Headless CI Validation Builds

```bitbake
# A fast CI smoke-test build deliberately targeting the smallest possible
# image, to validate "does the metadata at least parse and produce SOMETHING
# bootable" quickly, separate from a slower, full nightly build of the
# complete product image set:
bitbake core-image-minimal
```

---

## 16. Common Pitfalls and Debugging

### 1. Forgetting to re-source oe-init-build-env in a new shell
Every new terminal session needs `source oe-init-build-env build` (or equivalent) run again — `bitbake` simply isn't on `PATH` otherwise. A common early-career mistake/confusion point: "bitbake: command not found" in a freshly opened terminal despite having successfully built earlier in a different terminal session.

### 2. Disk space exhaustion mid-build
Full graphical image builds can consume well over 100GB of `tmp/work/` if `rm_work` isn't enabled, plus `sstate-cache`/`downloads` growth over time. `BB_DISKMON_DIRS` in `local.conf` (mentioned in the BitBake document) proactively aborts a build before catastrophic out-of-space failures mid-task (which can leave corrupted partial state) rather than discovering the problem the hard way.

### 3. Confusing TMPDIR-safe-to-delete with conf/-NOT-safe-to-delete
A surprisingly common, costly mistake: running `rm -rf build/` (deleting EVERYTHING including `conf/local.conf`/`bblayers.conf`, not just `tmp/`) when the intent was just "give me a clean tmp." Always be precise: `rm -rf build/tmp` is safe and common; `rm -rf build` destroys your actual configuration too.

### 4. sstate-cache silently growing unbounded over time
Long-lived shared/CI sstate-cache directories accumulate entries for every recipe-signature-version combination ever built, including ones from long-abandoned recipe versions — periodic cleanup (`sstate-cache-management.sh`, a script bundled with Yocto specifically for this purpose) is sometimes needed to keep shared cache storage from growing indefinitely.

### 5. CVE scan reports nothing despite a recipe having known, real CVEs
Check `CVE_PRODUCT` mapping first (Section 14) — a silent "0 CVEs found" due to a name-mapping mismatch is a much more dangerous failure mode than an obvious error, since it looks like a clean result rather than a broken scan.

### 6. Builds that work locally fail in CI with seemingly unrelated errors
Frequently traces back to environment differences CI doesn't share with a developer's local machine — different available disk space (triggering `BB_DISKMON_DIRS` aborts CI hits but a developer's larger local disk never does), a stale/inconsistent shared `sstate-cache` mirror serving a CI runner a subtly different cached object than what a from-scratch local build produces (rare, but `bitbake-diffsigs` is the standard tool for tracking this down), or simply CI running on a different host architecture (affecting `-native` recipe behavior) than the developer's local machine.

### 7. LICENSE compliance gaps discovered late, near a release deadline
Reviewing `tmp/deploy/licenses/<image>/license.manifest` should be a routine, EARLY, and ongoing part of the development process (ideally automated as a CI check flagging any newly-introduced license that doesn't match an approved allowlist) — not a one-time audit performed only right before shipping, when discovering an unacceptable license deep in a transitive dependency is far more disruptive and time-pressured to resolve.

---

## 17. Interview Questions and Answers

**Q1: Clarify the actual relationship between BitBake, OpenEmbedded-Core, the Yocto Project, and Poky — terms that are often used loosely/interchangeably.**

**A:** BitBake is the generic, Linux-agnostic task-execution engine — it knows nothing inherently about building embedded Linux distributions; it's purely a metadata parser and task scheduler. OpenEmbedded-Core (oe-core, delivered as the `meta` layer) is the foundational collection of BitBake metadata — base classes, core recipes for essential utilities/libraries, fundamental cross-compilation machinery — that actually turns BitBake into something useful for building Linux systems; it's maintained by the broader OpenEmbedded community, which predates and is organizationally distinct from the Yocto Project. The Yocto Project is a Linux Foundation collaborative project that builds ON TOP of BitBake + oe-core, adding project governance, additional developer tooling (devtool, the eSDK, extensive documentation, automated testing infrastructure, BSP/layer compatibility certification programs), and produces Poky as its reference, ready-to-build distribution. Poky itself is simply BitBake + oe-core (`meta`) + a minimal example distro policy layer (`meta-poky`) + reference/QEMU machine support (`meta-yocto-bsp`), bundled together as the actual git repository most people clone and start building from. In casual usage "Yocto" has become a catch-all term for this entire stack, but technically: BitBake is the engine, oe-core is the foundational metadata, Poky is the buildable reference distribution, and the Yocto Project is the overarching collaborative effort/governance body producing and maintaining all of it together.

---

**Q2: Why does Yocto build its own complete cross-compilation toolchain from source for every build, rather than using a pre-built, off-the-shelf cross-compiler?**

**A:** Several reasons converge: (1) **Exact reproducibility and version consistency** — the toolchain (GCC, binutils, the C library) and the target's actual kernel headers must be precisely matched and consistently versioned for genuinely correct, bug-free cross-compilation; a generic, pre-built off-the-shelf cross-toolchain has no guaranteed relationship to your specific target's exact kernel version or C library configuration, creating a real risk of subtle ABI mismatches or missing/incorrect syscall definitions. (2) **C library choice flexibility** — building from source lets a distro configuration choose between glibc, musl, or other C library implementations as a `TCLIBC` policy decision, which a fixed pre-built toolchain wouldn't allow. (3) **Build-time configurability** — security hardening flags, specific architecture tuning (the exact `-mcpu`/`-march` settings for a specific CPU core, as covered in the BSP Layer document), and other compiler-level policy decisions can all be consistently applied across the ENTIRE toolchain build, not just application-level recipes. (4) **Reproducibility/auditability** — for organizations needing to audit or certify their entire software supply chain (increasingly common for safety-critical, security-sensitive, or regulated industries), having the toolchain ITSELF be built from known, auditable source as part of the same reproducible build process — rather than trusting an opaque, externally-provided binary blob — is a meaningfully stronger compliance/security posture. The tradeoff is build time: constructing a full toolchain from scratch is a genuinely expensive first-build cost, which is exactly why the shared-state cache (sstate) exists — once built, that toolchain build output is cached and reused across all subsequent builds/recipes needing it, so this cost is paid once, not repeatedly.

---

**Q3: Explain what "pseudo" does and why Yocto needs it, given that BitBake is explicitly designed to never run as root.**

**A:** A properly assembled root filesystem legitimately needs files with specific ownership/permissions that only root can normally set — `/etc/shadow` owned by root with restrictive permissions, device nodes created via `mknod` (which normally requires root privileges), files owned by specific non-default UIDs for various system service accounts. But running the ENTIRE BitBake build process as actual root would be a significant, unnecessary security risk (a compromised or buggy recipe's build script running as genuine root on the build host is a much more severe problem than the same bug running as a normal unprivileged user) — and Yocto deliberately, as a hard design principle, refuses to require or even support running as root for normal builds. Pseudo solves this by using `LD_PRELOAD` to intercept the relevant filesystem-ownership-related system calls (`chown`, `chmod`, `mknod`, and related) during the specific tasks that need them (primarily `do_install` and `do_rootfs`), and instead of actually performing those privileged operations (which would fail or be meaningless for a non-root user), pseudo records the INTENDED ownership/permissions in its own separate, persistent database. Later, when packages are created or the rootfs is assembled into a final image, the actual file content is correctly tagged/packaged with this recorded "intended" ownership — meaning the FINAL artifacts (packages, root filesystem images) correctly reflect root-owned files and special device nodes, even though no part of the actual build process ever ran with real root privileges on the build host. It's a clever, narrowly-scoped workaround that achieves the practical requirement (correctly-owned output files) without the security cost (an actually-root build process).

---

**Q4: A team needs to support a multi-year production deployment of an embedded product. Why does choosing a Yocto LTS release matter, and what are the practical tradeoffs of NOT choosing one?**

**A:** Yocto Project LTS (Long Term Support) releases receive extended security and critical-bug-fix maintenance for a substantially longer window (historically around four years) than the project's regular, non-LTS releases, which typically only receive active maintenance for a much shorter period before the community's attention moves to the next release. For a production embedded product with a multi-year field deployment lifespan — common in industrial, automotive, medical, and infrastructure embedded systems — this matters enormously: newly discovered CVEs in bundled open-source components (a near-certainty over a multi-year window, given the sheer volume of software typically included even in a "minimal" embedded Linux image) need a credible path to being patched without requiring a full, disruptive version upgrade of the entire underlying build system mid-deployment. Choosing a non-LTS release for a long-lived product creates a forced dilemma a year or two into deployment: either undertake a full Yocto version upgrade (itself a non-trivial undertaking, potentially requiring BSP layer updates, recipe compatibility fixes, and substantial requalification/testing effort for a regulated or safety-critical product) earlier than ideal, or continue shipping security patches for an officially unmaintained, unsupported base — neither of which is an attractive position for a team responsible for a deployed product's long-term security posture. Additionally, the broader layer ecosystem (BSP layers from silicon vendors, community software layers) tends to maintain longer-lived compatibility branches specifically tracking LTS releases, since that's where the bulk of real production usage concentrates — meaning a non-LTS release may also have a shorter practical window of available, actively-maintained third-party layer support, compounding the upgrade-pressure problem.

---

**Q5: What's the practical difference between IMAGE-level testing (testimage) and PACKAGE-level testing (ptest), and why does Yocto provide both rather than just one?**

**A:** `ptest` operates at the level of an INDIVIDUAL package's own upstream test suite, packaged as a separate `-ptest` package and run directly ON the target device (or QEMU equivalent) — its purpose is to catch cross-compilation-specific or target-environment-specific bugs in that ONE piece of software that the upstream project's own test suite is designed to detect, but which would never be exercised by a build-host-only compile-and-link success (a successful BUILD doesn't guarantee correct RUNTIME behavior on a genuinely different target architecture/environment than the build host). `testimage`, by contrast, operates at the level of the WHOLE ASSEMBLED IMAGE — it boots the actual, complete, final image artifact and validates SYSTEM-level properties: does it boot at all, does networking come up, does SSH actually accept connections, is systemd functioning correctly, are there unexpected error patterns in the boot log — concerns that are fundamentally about how all the INDIVIDUALLY-correct packages interact and integrate together as a complete, working system, which no individual package's own test suite could possibly catch (a package's ptest suite has no visibility into whether IT specifically conflicts with or is missing a runtime dependency relative to everything else installed in a specific image). Yocto provides both because they validate genuinely different, complementary failure modes: ptest catches "this specific piece of software is subtly broken when actually run on the target," while testimage catches "this specific COMBINATION of otherwise-individually-correct software, assembled into THIS specific image, doesn't actually work together as a coherent, bootable system" — both categories of bug are real, common, and essentially undetectable by the OTHER testing layer alone.

---

**Q6: Why is CVE_PRODUCT mapping a meaningfully dangerous thing to get wrong, beyond just being a minor inconvenience?**

**A:** The `cve-check` mechanism works by matching a recipe's name (`PN`) and version (`PV`) against entries in a CVE database (commonly NVD-derived data), to identify whether any publicly known vulnerabilities affect that exact piece of software at that exact version. When a recipe's `PN` happens to differ from how that same underlying software is identified/named within the CVE database itself (a surprisingly common occurrence — package names in Yocto/OpenEmbedded don't always exactly match the "official" product name a CVE database uses), the matching simply fails to find ANY entries for that recipe — not because there genuinely are no CVEs, but because the LOOKUP itself never successfully matched anything. The resulting `cve-check` report for that recipe shows zero CVEs found, which is INDISTINGUISHABLE, from the report consumer's perspective, from a recipe that genuinely has no known vulnerabilities — there's no separate "lookup failed" versus "lookup succeeded, found nothing" signal by default. This is a meaningfully worse failure mode than an obvious error or crash: a team relying on `cve-check` reports as part of their security/compliance process could reasonably believe a component has a clean security record, continue shipping it across multiple release cycles, while it actually carries known, publicly disclosed vulnerabilities that a CORRECTLY-mapped `CVE_PRODUCT` entry would have surfaced. This is precisely why explicitly auditing and correcting `CVE_PRODUCT` mappings for recipes (especially ones with non-obvious or commonly-renamed upstream project names) is a meaningful, non-optional part of taking CVE scanning seriously, rather than just enabling `cve-check` and trusting its output uncritically.
