# Yocto/OpenEmbedded Variables — Complete Reference Guide

---

## Table of Contents

1. [Introduction and How to Use This Reference](#1-introduction-and-how-to-use-this-reference)
2. [Path and Directory Variables](#2-path-and-directory-variables)
3. [Package Identity Variables](#3-package-identity-variables)
4. [Architecture and Cross-Compilation Variables](#4-architecture-and-cross-compilation-variables)
5. [Machine and Distro Variables](#5-machine-and-distro-variables)
6. [Dependency Variables](#6-dependency-variables)
7. [Source Fetching Variables](#7-source-fetching-variables)
8. [Compiler and Build Flag Variables](#8-compiler-and-build-flag-variables)
9. [Packaging Variables](#9-packaging-variables)
10. [Image-Specific Variables](#10-image-specific-variables)
11. [Build Performance and Cache Variables](#11-build-performance-and-cache-variables)
12. [License and Compliance Variables](#12-license-and-compliance-variables)
13. [Layer Variables](#13-layer-variables)
14. [Task and Class Control Variables](#14-task-and-class-control-variables)
15. [Variable Precedence and Resolution Order](#15-variable-precedence-and-resolution-order)
16. [How to Discover a Variable's Actual Value](#16-how-to-discover-a-variables-actual-value)
17. [Common Pitfalls and Debugging](#17-common-pitfalls-and-debugging)
18. [Interview Questions and Answers](#18-interview-questions-and-answers)

---

## 1. Introduction and How to Use This Reference

OpenEmbedded/Yocto defines several hundred variables across BitBake itself, oe-core, and the broader layer ecosystem. This document organizes the most important, frequently-encountered variables by FUNCTIONAL CATEGORY rather than alphabetically, since understanding WHY a group of variables exists together (e.g., the FHS-aligned path variables, or the full DEPENDS/RDEPENDS family) is more valuable for genuine comprehension than a flat alphabetical dump. For variables covered in depth elsewhere in this document set (BitBake operators, recipe-level packaging variables, image-level variables, BSP machine variables), this reference provides a concise summary with a pointer to the relevant deep-dive document.

---

## 2. Path and Directory Variables

### Build-Time Directory Variables (recipe-scoped)

| Variable    | Meaning                                                              |
|---------------|----------------------------------------------------------------|
| `WORKDIR`    | Recipe's private build scratch root: `tmp/work/<arch>/<pn>/<pv-pr>/`|
| `S`          | Unpacked, patched source directory                                  |
| `B`          | Build directory (configure/compile run here; may differ from S)     |
| `D`          | Destination/staging root that `do_install` populates                |
| `T`          | Temporary directory for task logs/scripts: `${WORKDIR}/temp`        |
| `SYSROOT_DESTDIR` | Staging area for `do_populate_sysroot` output                  |
| `STAGING_DIR_HOST`| Sysroot for the target/host architecture (this recipe's dependencies)|
| `STAGING_DIR_NATIVE`| Sysroot for native (build-host) tools                          |
| `STAGING_DIR_TARGET`| Sysroot when build≠host≠target (Canadian Cross / SDK builds)   |
| `STAGING_BINDIR_NATIVE`| Native tool binaries location                                |
| `RECIPE_SYSROOT`| Per-recipe view of dependency sysroot (modern, isolated approach)  |
| `RECIPE_SYSROOT_NATIVE`| Per-recipe native sysroot                                    |

### FHS-Aligned Installation Path Variables (used in do_install)

| Variable          | Default Path        | Purpose                                |
|---------------------|----------------------|-------------------------------------------|
| `prefix`           | `/usr`               | Root of the installation hierarchy        |
| `exec_prefix`      | `${prefix}`           | Root for architecture-dependent files     |
| `bindir`           | `/usr/bin`            | User executables                           |
| `sbindir`          | `/usr/sbin`           | System admin executables                    |
| `libdir`           | `/usr/lib` (or `/usr/lib64`)| Libraries                            |
| `libexecdir`       | `/usr/libexec`         | Internal executables not meant for direct user invocation|
| `includedir`       | `/usr/include`         | C/C++ header files                          |
| `sysconfdir`       | `/etc`                | Configuration files                          |
| `datadir`          | `/usr/share`           | Architecture-independent data                |
| `localstatedir`    | `/var`                 | Variable runtime data                         |
| `sharedstatedir`   | `/com` (or `/var/lib`) | Shared mutable architecture-independent data|
| `docdir`           | `/usr/share/doc`       | Documentation                                 |
| `mandir`           | `/usr/share/man`       | Man pages                                     |
| `infodir`          | `/usr/share/info`      | GNU info pages                                |
| `systemd_unitdir`/`systemd_system_unitdir`| `/lib/systemd/system`| systemd unit files          |
| `base_bindir`/`base_sbindir`/`base_libdir`| `/bin`, `/sbin`, `/lib`| Pre-usrmerge legacy root-level paths (often symlinked to /usr equivalents on modern distros)|

### Build-Output/Deploy Directory Variables

| Variable             | Meaning                                                          |
|------------------------|------------------------------------------------------------------|
| `TMPDIR`               | Top-level build output root (`build/tmp`)                         |
| `DEPLOY_DIR`           | Root of all final deployable output (`tmp/deploy`)                  |
| `DEPLOY_DIR_IMAGE`     | Final images, kernels, bootloaders for the current MACHINE          |
| `DEPLOY_DIR_RPM`/`_DEB`/`_IPK`| Final binary package feed output, by format                  |
| `IMGDEPLOYDIR`         | Image-recipe-specific deploy staging (before final copy to DEPLOY_DIR_IMAGE)|
| `DL_DIR`               | Shared download cache for all fetched source                        |
| `SSTATE_DIR`           | Shared-state cache directory                                          |
| `PERSISTENT_DIR`       | Cross-build persistent data (e.g., fetcher metadata caches)            |

---

## 3. Package Identity Variables

| Variable   | Meaning                                                                |
|--------------|------------------------------------------------------------------------|
| `PN`        | Package Name (e.g., "busybox") — usually auto-derived from recipe filename|
| `PV`        | Package Version (e.g., "1.36.1") — usually auto-derived from filename     |
| `PR`        | Package Revision (e.g., "r0") — bump to force rebuild without changing PV |
| `PE`        | Package Epoch — rarely used, for version-comparison edge cases             |
| `BPN`       | "Base" PN — PN with common prefix/suffix stripping applied (e.g., strips `nativesdk-`)|
| `BP`        | "Base" package+version string: `${BPN}-${PV}`                             |
| `PF`        | Full package+version+revision specifier: `${PN}-${EXTENDPE}${PV}-${PR}`  |
| `EXTENDPE`  | Epoch formatted for use in PF (empty if PE is unset)                       |
| `PKGV`/`PKGR`/`PKGE`| The actual version/revision/epoch used in generated PACKAGE output (can differ from PV/PR/PE in rare cases)|
| `SRCPV`     | A computed version-string fragment derived from a git SRCREV, for use in PV (e.g. `"AUTOINC+abc1234"`)|

```bitbake
# Demonstration of how these relate for a recipe file "busybox_1.36.1.bb":
PN = "busybox"
PV = "1.36.1"
PR = "r0"
BPN = "busybox"
BP = "busybox-1.36.1"
PF = "busybox-1.36.1-r0"
```

---

## 4. Architecture and Cross-Compilation Variables

| Variable           | Meaning                                                              |
|-----------------------|----------------------------------------------------------------|
| `TARGET_ARCH`        | CPU architecture of the final target (e.g., `arm`, `aarch64`, `x86_64`, `riscv64`)|
| `TARGET_OS`           | Target OS string (e.g., `linux`, `linux-gnueabi`, `linux-musl`)        |
| `TARGET_VENDOR`       | Vendor string component of the target triplet (typically `-poky` or similar)|
| `TARGET_SYS`          | Full target system triplet (e.g., `arm-poky-linux-gnueabi`)             |
| `TARGET_PREFIX`       | Cross-toolchain binary prefix (e.g., `arm-poky-linux-gnueabi-`)          |
| `TARGET_FPU`          | Floating point ABI selection (e.g., `hard`, `soft`)                      |
| `HOST_ARCH`/`HOST_OS`/`HOST_SYS`| Same concepts, for the HOST system (usually = target for normal recipes; = build for -native recipes)|
| `BUILD_ARCH`/`BUILD_OS`/`BUILD_SYS`| Same concepts, for the BUILD machine (where compilation actually runs)|
| `SDKMACHINE`           | Architecture of the machine an SDK itself will run on                     |
| `MULTIMACH_TARGET_SYS` | A combined identifier used for distinguishing parallel architecture-specific build output|
| `PACKAGE_ARCH`         | The package architecture string used in package feed naming/compatibility|
| `TUNE_ARCH`            | Base architecture name from the active CPU tuning file                    |
| `TUNE_FEATURES`        | Active CPU tuning feature flags (e.g., `aarch64 cortexa72`)               |
| `DEFAULTTUNE`          | The selected tuning profile name for the current MACHINE                   |
| `BASE_LIB`             | `lib` or `lib64`, depending on architecture's standard library directory convention|

---

## 5. Machine and Distro Variables

(Full coverage in the BSP Layer and Build System documents — summarized here.)

| Variable                | Meaning                                                            |
|----------------------------|------------------------------------------------------------------|
| `MACHINE`                  | The active board/machine identifier (e.g., `raspberrypi4`)         |
| `MACHINE_ARCH`              | Machine-specific package architecture string                        |
| `MACHINE_FEATURES`          | Hardware capabilities present on this machine                        |
| `MACHINE_EXTRA_RDEPENDS`/`_RRECOMMENDS`| Extra packages pulled into every image for this machine specifically|
| `MACHINEOVERRIDES`          | Hierarchical override chain for board families                       |
| `SERIAL_CONSOLES`           | UART device(s) and baud rate(s) for console/getty                    |
| `KERNEL_IMAGETYPE`          | Kernel binary format expected (zImage, Image, bzImage, etc.)         |
| `KERNEL_DEVICETREE`         | Device tree file(s) to build/deploy for this machine                  |
| `DISTRO`                    | The active distribution policy identifier                            |
| `DISTRO_FEATURES`           | Distribution-wide optional capability flags                          |
| `DISTRO_FEATURES_BACKFILL`/`_CONSIDERED`| Mechanism for managing default feature inclusion/exclusion across releases|
| `DISTRO_VERSION`/`DISTRO_NAME`/`DISTRO_CODENAME`| Distro identity/branding metadata                       |
| `PACKAGE_CLASSES`           | Package format(s) to build (`package_rpm`, `package_deb`, `package_ipk`)|

---

## 6. Dependency Variables

(Full coverage in the Recipes document — summarized here.)

| Variable                  | Time    | Meaning                                                       |
|-----------------------------|---------|------------------------------------------------------------------|
| `DEPENDS`                  | Build   | Recipe needs another recipe's sysroot output to build              |
| `RDEPENDS:${PN}`           | Runtime | Package needs another package installed to run                     |
| `RRECOMMENDS:${PN}`        | Runtime | Soft/optional runtime dependency                                    |
| `RSUGGESTS:${PN}`          | Runtime | Even softer suggestion (package-manager hint only)                  |
| `RCONFLICTS:${PN}`         | Runtime | Cannot coexist with the named package                                |
| `RPROVIDES:${PN}`          | Runtime | This package also satisfies another package name                     |
| `RREPLACES:${PN}`          | Runtime | This package replaces/supersedes another package                      |
| `PROVIDES`                 | Build   | This recipe also satisfies another (often virtual/*) recipe name      |
| `PREFERRED_PROVIDER_<virtual>`| Config| Resolves which real recipe satisfies a virtual/* name                |
| `PREFERRED_VERSION_<pn>`   | Config  | Pins which version of a multi-version-available recipe to use         |
| `BBCLASSEXTEND`            | Build   | Generates additional recipe variants (native, nativesdk) from one recipe file|
| `do_X[depends]`            | Task    | Explicit task-level dependency beyond what DEPENDS auto-generates       |

---

## 7. Source Fetching Variables

(Full coverage in the BitBake and Recipes documents — summarized here.)

| Variable            | Meaning                                                              |
|------------------------|------------------------------------------------------------------|
| `SRC_URI`              | Source location(s) and patch/file list                              |
| `SRC_URI[md5sum]`/`[sha256sum]`| Checksum verification for non-git fetches                      |
| `SRCREV`               | Pinned git/SCM commit hash (or `${AUTOREV}` — avoid in production)   |
| `SRCREV_FORMAT`        | How multiple SRCREVs (multi-repo fetches) combine into SRCPV          |
| `SRCPV`                | Computed version-string fragment from SRCREV, for embedding in PV     |
| `PREMIRRORS`           | Mirrors checked BEFORE the original SRC_URI location                  |
| `MIRRORS`              | Mirrors checked AFTER the original location fails                     |
| `BB_GENERATE_MIRROR_TARBALLS`| Generate local mirror tarballs of every fetched source           |
| `BB_NO_NETWORK`        | Forbid all network access (fail loudly instead of fetching)            |
| `UPSTREAM_CHECK_URI`/`_REGEX`/`_GITTAGREGEX`| Drives devtool's upstream-version-checking mechanism      |
| `FILESEXTRAPATHS`      | Additional search paths for `file://` SRC_URI entries (critical in bbappends)|
| `FILESPATH`            | The full computed search path for `file://` entries                     |

---

## 8. Compiler and Build Flag Variables

| Variable      | Meaning                                                                |
|-----------------|------------------------------------------------------------------------|
| `CC`           | The cross C compiler invocation (includes target prefix and base flags)|
| `CXX`          | The cross C++ compiler invocation                                       |
| `CPP`          | The cross C preprocessor invocation                                      |
| `AS`           | The cross assembler                                                       |
| `LD`           | The cross linker                                                          |
| `AR`           | The cross archiver (static libraries)                                      |
| `STRIP`        | The cross strip tool (debug symbol stripping)                              |
| `NM`           | The cross symbol-table tool                                                |
| `OBJCOPY`/`OBJDUMP`| Cross binary-manipulation/inspection tools                              |
| `RANLIB`       | The cross archive-indexing tool                                            |
| `CFLAGS`       | C compiler flags (optimization level, architecture tuning, hardening)       |
| `CXXFLAGS`     | C++ compiler flags                                                          |
| `CPPFLAGS`     | C preprocessor flags (include paths, preprocessor defines)                   |
| `LDFLAGS`      | Linker flags (often includes security hardening like `-Wl,-z,relro`)        |
| `EXTRA_OECONF` | Extra arguments appended to Autotools `./configure` invocations              |
| `EXTRA_OEMAKE` | Extra arguments/variables passed to `oe_runmake`                              |
| `EXTRA_OECMAKE`| Extra arguments appended to CMake invocations                                 |
| `EXTRA_OEMESON`| Extra arguments appended to Meson invocations                                  |
| `PARALLEL_MAKE`| Parallel job count for an individual recipe's own make invocation (`-j N`)   |
| `DEBUG_BUILD`  | When set, builds with debug-oriented compiler flags instead of optimized ones |
| `SELECTED_OPTIMIZATION`| The actual optimization flag in effect (e.g., `-O2`)                    |

---

## 9. Packaging Variables

(Full coverage in the Recipes document — summarized here.)

| Variable               | Meaning                                                          |
|---------------------------|------------------------------------------------------------------|
| `PACKAGES`                | List of output packages this recipe produces                       |
| `FILES:<pkg>`             | Glob patterns defining what files belong in each output package    |
| `ALLOW_EMPTY:<pkg>`       | Permit a package with zero matched files to still be generated      |
| `PACKAGE_BEFORE_PN`       | Insert extra packages before the main `${PN}` package in PACKAGES order|
| `PACKAGE_ARCH`            | Architecture string used for this package's compatibility tagging   |
| `PACKAGE_STRIP`           | Whether to strip debug symbols from packaged binaries (normally yes)|
| `INSANE_SKIP:<pkg>`       | Selectively disable specific QA checks for a package                |
| `LICENSE:<pkg>`           | Per-package license override (when a recipe's packages have differing licenses)|
| `CONFFILES:<pkg>`         | Marks files as configuration files (package manager preserves local edits on upgrade)|
| `pkg_postinst:<pkg>`      | Shell script run on the TARGET at package installation time          |
| `pkg_postrm:<pkg>`        | Shell script run on the TARGET at package removal time               |
| `pkg_preinst:<pkg>`/`pkg_prerm:<pkg>`| Pre-install/pre-removal target-side scripts                |
| `SYSTEMD_SERVICE:<pkg>`   | systemd unit file(s) associated with a package (via systemd.bbclass)|
| `SYSTEMD_AUTO_ENABLE:<pkg>`| Whether the systemd service is enabled by default                  |
| `USERADD_PARAM:<pkg>`/`GROUPADD_PARAM:<pkg>`| User/group creation parameters (via useradd.bbclass)     |

---

## 10. Image-Specific Variables

(Full coverage in the Image Recipes document — summarized here.)

| Variable                     | Meaning                                                          |
|---------------------------------|--------------------------------------------------------------|
| `IMAGE_INSTALL`                 | Primary package selection list for an image                      |
| `IMAGE_FEATURES`                 | Toggleable high-level image capability set                        |
| `IMAGE_FSTYPES`                   | Output file format(s) to generate                                  |
| `IMAGE_ROOTFS_SIZE`               | Target rootfs size in KB                                            |
| `IMAGE_ROOTFS_EXTRA_SPACE`        | Extra free space added beyond computed content size                  |
| `IMAGE_OVERHEAD_FACTOR`           | Multiplier applied for filesystem metadata overhead                  |
| `IMAGE_LINGUAS`                    | Locale/language packages to include                                 |
| `PACKAGE_EXCLUDE`                  | Globally exclude specific packages from all images                  |
| `PACKAGE_EXCLUDE_COMPLEMENTARY`    | Exclude entire complementary package classes (-dev, -dbg, -doc)      |
| `NO_RECOMMENDATIONS`               | Globally disable all RRECOMMENDS processing                          |
| `ROOTFS_POSTPROCESS_COMMAND`       | Shell functions run after rootfs assembly                            |
| `IMAGE_PREPROCESS_COMMAND`         | Shell functions run before rootfs assembly                           |
| `WKS_FILE`                          | Wic kickstart file defining partition layout                         |
| `EXTRA_IMAGEDEPENDS`               | Extra recipe build dependencies for the image (bootloader, etc.)      |

---

## 11. Build Performance and Cache Variables

| Variable                | Meaning                                                              |
|----------------------------|------------------------------------------------------------------|
| `BB_NUMBER_THREADS`        | Number of parallel BitBake tasks across recipes                      |
| `PARALLEL_MAKE`             | Parallel job count within an individual recipe's own build           |
| `BB_DISKMON_DIRS`            | Disk-space monitoring thresholds (abort build before catastrophic exhaustion)|
| `SSTATE_DIR`                  | Local shared-state cache location                                      |
| `SSTATE_MIRRORS`              | Network-shared sstate cache mirror(s)                                  |
| `BB_HASHSERVE`                 | Hash equivalence server configuration                                   |
| `BB_SIGNATURE_HANDLER`          | Which signature computation algorithm BitBake uses                       |
| `RM_WORK_EXCLUDE`                | Recipes exempted from automatic `rm_work` cleanup                       |
| `INHERIT`                          | Global class inheritance applied to every recipe (e.g., `+= "rm_work"`)|
| `BBINCLUDELOGS`                     | Include full task logs inline in BitBake's own console output on failure|

---

## 12. License and Compliance Variables

| Variable                | Meaning                                                              |
|----------------------------|------------------------------------------------------------------|
| `LICENSE`                  | SPDX-style license identifier(s) for this recipe's software           |
| `LIC_FILES_CHKSUM`         | Checksum verification of the actual license text file(s)              |
| `LICENSE_CREATE_PACKAGE`   | Generate a dedicated `-lic` package containing license text             |
| `COPY_LIC_MANIFEST`/`COPY_LIC_DIRS`| Copy full license text into the deployed license manifest        |
| `INCOMPATIBLE_LICENSE`     | Licenses to entirely exclude from the build (e.g., excluding GPL-3.0 for a closed ecosystem)|
| `CVE_PRODUCT`              | Correct mapping for CVE database product-name matching                  |
| `CVE_VERSION`               | Override version string used for CVE matching                            |
| `CVE_CHECK_SKIP_RECIPE`     | Skip CVE scanning for a specific recipe (e.g., a confirmed false positive)|

---

## 13. Layer Variables

(Full coverage in the Layers document — summarized here.)

| Variable                       | Meaning                                                       |
|-----------------------------------|------------------------------------------------------------|
| `BBPATH`                          | Search path extension for relatively-referenced files         |
| `BBFILES`                          | Glob patterns locating a layer's .bb/.bbappend files            |
| `BBFILE_COLLECTIONS`               | Registers a layer's symbolic name                                |
| `BBFILE_PATTERN_<name>`            | Regex mapping file paths to a collection                         |
| `BBFILE_PRIORITY_<name>`           | Conflict-resolution priority                                     |
| `LAYERDEPENDS_<name>`              | Layer dependency declarations                                    |
| `LAYERSERIES_COMPAT_<name>`        | Declares supported Yocto release codenames                       |
| `BBLAYERS`                          | The active list of layer paths (set in bblayers.conf)            |

---

## 14. Task and Class Control Variables

| Variable                  | Meaning                                                              |
|------------------------------|------------------------------------------------------------------|
| `inherit`                    | Pulls in a `.bbclass`'s logic/variables/tasks into this recipe        |
| `INHERIT`                     | Global inherit applied automatically to EVERY recipe in the build      |
| `addtask`                      | Declares a custom task and its ordering relative to other tasks         |
| `EXPORT_FUNCTIONS`              | (older mechanism) Exports class-defined functions into recipe scope     |
| `do_X[noexec]`                  | Marks a task as a no-op (skipped entirely)                              |
| `do_X[nostamp]`                  | Forces a task to always re-run, ignoring its completion stamp             |
| `do_X[depends]`                    | Explicit cross-recipe task dependency                                     |
| `do_X[vardeps]`                     | Forces a task to be considered changed if a specific variable's value changes (signature computation)|
| `do_X[prefuncs]`/`[postfuncs]`        | Functions to run immediately before/after a task                          |
| `COMPATIBLE_MACHINE`                    | Regex restricting which MACHINE values this recipe is valid for           |
| `COMPATIBLE_HOST`                        | Regex restricting which HOST_SYS values this recipe is valid for          |
| `PACKAGECONFIG`                            | Toggleable optional build-time feature flags (see Recipes document)       |
| `OVERRIDES`                                  | The active list of override context strings (machine, distro, class-*, etc.)|

---

## 15. Variable Precedence and Resolution Order

Understanding WHERE a variable's final value comes from, when multiple layers/files could potentially set it, is one of the most valuable debugging skills in Yocto work. The general resolution order (later overrides earlier, for a plain `=` assignment; `:append`/`:prepend`/`:remove` are special-cased as described in the BitBake document):

```
1. bitbake.conf (oe-core's base configuration — the absolute foundation)
2. conf/machine/${MACHINE}.conf (and any conf/machine/include/*.inc it requires)
3. conf/distro/${DISTRO}.conf (and any conf/distro/include/*.inc it requires)
4. Each layer's .bbclass files, in inherit order, as the recipe is parsed
5. The recipe's own .bb file content (and any .inc files it requires)
6. Each matching .bbappend file, in ascending layer-priority order
7. local.conf (parsed AFTER all of the above — highest-priority plain assignment wins)
8. Command-line -D/--define style overrides (rarely used, highest possible precedence)
```

```bitbake
# A useful mental model: imagine BitBake conceptually concatenating all
# applicable files in this order and processing them top to bottom — a
# later '=' assignment to the same variable always wins over an earlier one,
# while ':append'/':prepend' accumulate regardless of this file-level ordering
# (their own internal ordering is governed by layer priority specifically,
# as detailed in the BitBake document).
```

### OVERRIDES — The Mechanism Behind Context-Sensitive Variable Resolution

```bitbake
# OVERRIDES is itself a variable — a colon-separated list of "context" strings
# currently active, conceptually similar to:
OVERRIDES = "pn-${PN}:${MACHINE}:${DISTRO}:class-target:..."
```

Every `VAR:override` syntax (covered extensively in the BitBake document) works by checking whether `override` appears in the current `OVERRIDES` list — `VAR:my-board = "x"` only takes effect when `my-board` is present in `OVERRIDES`, which happens automatically whenever `MACHINE = "my-board"` is the active machine. This is the unifying mechanism behind machine-specific, distro-specific, and class-specific (`class-native`, `class-target`, `class-nativesdk`) variable overrides all using the exact same colon syntax.

---

## 16. How to Discover a Variable's Actual Value

This is one of the single most useful day-to-day debugging skills:

```bash
# Dump EVERY variable's final, fully-resolved value for a specific recipe
bitbake -e busybox > busybox-env.txt

# Search for one specific variable
bitbake -e busybox | grep '^SRC_URI='
bitbake -e busybox | grep '^WORKDIR='

# Including the COMMENT showing exactly which file last set this variable
# (BitBake's -e output includes provenance comments like
# "# $WORKDIR [2 operations]" tracing every contributing assignment)
bitbake -e busybox | grep -B5 '^WORKDIR='

# Dump the GLOBAL environment (no specific recipe — base configuration only,
# useful for checking MACHINE/DISTRO-level variables without recipe-specific noise)
bitbake -e | less

# For an IMAGE recipe specifically (shows IMAGE_INSTALL/IMAGE_FEATURES resolution)
bitbake -e core-image-minimal | grep '^IMAGE_INSTALL='
```

```bash
# devshell drops you into a real shell with every variable correctly
# exported as a genuine shell environment variable — useful for quickly
# checking something interactively without parsing -e output:
bitbake -c devshell busybox
echo $CFLAGS
echo $S
```

---

## 17. Common Pitfalls and Debugging

### 1. Assuming a variable change took effect without verifying via `bitbake -e`
The single most common source of "I changed the variable but nothing happened" confusion is a variable being set in the WRONG override context (wrong MACHINE, wrong PN), or being overridden again LATER in the resolution order by something else (a distro config, a higher-priority bbappend). Always verify with `bitbake -e <recipe> | grep ^VARNAME=` rather than assuming.

### 2. Confusing PN with BPN
For most ordinary recipes `PN` and `BPN` are identical, but for `-native`/`-nativesdk`/multilib variant recipes they diverge (`BPN` strips the variant-specific prefix/suffix) — code that needs the "true" base package name regardless of variant should use `BPN`, not `PN`.

### 3. Hardcoding a path instead of using the corresponding FHS variable
`install -d ${D}/usr/bin` instead of `install -d ${D}${bindir}` — works today, but breaks silently if a distro configuration ever changes the FHS layout convention (rare but supported), and is generally considered poor recipe style/non-upstreamable.

### 4. Setting a variable in local.conf that should really be in a distro/machine config
`local.conf` is the HIGHEST-precedence, build-host-specific configuration layer — appropriate for genuinely build-host-specific settings (parallelism, local paths) but a poor place for settings that should travel WITH the product definition itself (which packages to include, which features to enable) since `local.conf` is conventionally NOT checked into version control (it's regenerated per build-host from a template). Product-defining settings belong in a distro or image recipe, which IS version-controlled.

### 5. Not understanding that OVERRIDES context determines which `:override` syntax actually applies
Writing `SRC_URI:append:my-typo'd-machine-name = "..."` silently does nothing (no error) if the override string doesn't exactly match anything currently in `OVERRIDES` — there's no spell-check on override names. Verify the exact `MACHINE`/override spelling with `bitbake -e <recipe> | grep ^OVERRIDES=`.

---

## 18. Interview Questions and Answers

**Q1: Explain the relationship between PN, BPN, PV, PR, and PF, and give a concrete scenario where PN and BPN actually differ.**

**A:** For an ordinary recipe, `PN` (Package Name) and `BPN` (Base Package Name) are identical — both simply equal the recipe's name as derived from its filename (e.g., "busybox" from `busybox_1.36.1.bb`). They diverge specifically for variant recipes generated via `BBCLASSEXTEND` or multilib mechanisms: a `-native` variant of a recipe has `PN = "busybox-native"` but `BPN` remains `"busybox"` (BitBake strips the recognized `-native` suffix when computing BPN specifically so that recipe logic needing to reference "the real underlying package name regardless of which variant this is" — for example, to find shared source files or apply common patches across both the target and native variants — has a way to do so without hardcoding variant-specific suffix-stripping logic itself). `PV` is simply the version string, `PR` is the package revision (bumped to force a rebuild/new package generation without an upstream version change), and `PF` is the fully-qualified "package+version+revision" identifier (`${PN}-${EXTENDPE}${PV}-${PR}`) used internally for things like the `tmp/work/` directory naming and stamp file naming, ensuring different versions/revisions of the same recipe never collide in the build's intermediate state.

---

**Q2: A developer wants a setting to apply to every developer's build automatically, version-controlled alongside the product's source. Should they put it in local.conf? Why or why not?**

**A:** No — `local.conf` is specifically designed as BUILD-HOST-LOCAL, per-developer-machine configuration, and is conventionally NOT checked into version control (it's typically auto-generated from a template the first time `oe-init-build-env` runs, and developers are expected to make genuinely local adjustments to it — parallelism settings appropriate to their specific machine's core count, perhaps a locally-relevant `DL_DIR`/`SSTATE_DIR` path). Settings that should apply consistently to EVERY developer's build, and that genuinely define what the product/distribution actually IS (which packages it includes, which features it enables, which DISTRO/MACHINE combination it targets), belong instead in version-controlled metadata: a distro configuration file, an image recipe, or — for the very common case of "every developer needs the same default `local.conf`/`bblayers.conf` content" — a `TEMPLATECONF`-based template (covered in the Build System document), which IS checked into version control and automatically populates each developer's freshly-created `local.conf` with the correct shared baseline starting point, while still leaving room for genuinely machine-local tweaks layered on top by each individual developer afterward.

---

**Q3: Explain how the OVERRIDES mechanism unifies machine-specific, distro-specific, and native/target/nativesdk-specific variable customization under one syntax.**

**A:** `OVERRIDES` is itself a BitBake variable — a colon-separated list of "context strings" that are currently considered active for the recipe being parsed, automatically populated with values like the current `MACHINE` name, the current `DISTRO` name, and class-membership markers like `class-native`/`class-target`/`class-nativesdk` depending on which variant of a recipe is being built. The `VARIABLE:override` syntax (e.g., `SRC_URI:append:my-board`, `DEPENDS:class-native`) works by checking, at parse/resolution time, whether the specific override string after the colon is present anywhere in the current `OVERRIDES` list — if so, that override clause takes effect; if not, it's simply ignored (no error). This single, uniform mechanism is what allows the EXACT SAME syntax to express machine-specific behavior (`:my-board`, active when `MACHINE = "my-board"`), distro-specific behavior (`:my-distro`, active when `DISTRO = "my-distro"`), and build-variant-specific behavior (`:class-native`, active specifically when building the `-native` variant of a recipe via `BBCLASSEXTEND`) — rather than needing three entirely separate, special-cased syntaxes for what are conceptually the same underlying need: "apply this setting only in a specific, named context." Understanding `OVERRIDES` as the single unifying mechanism behind all of these surface-level syntaxes is genuinely one of the more valuable "aha" moments in deeply understanding how BitBake's metadata language actually works.

---

**Q4: When debugging "why is this variable not what I expect," what is the single most useful command, and what does its output actually tell you beyond just the final value?**

**A:** `bitbake -e <recipe> | grep ^VARNAME=` (or piping to `grep -B N` to see a few lines of preceding context) is the single most useful command — but its value goes beyond just showing the FINAL resolved value. BitBake's `-e` (environment dump) output includes PROVENANCE COMMENTS directly above each variable's final value, showing the actual sequence of files/operations that contributed to that final value (e.g., "# $VARNAME [3 operations]" followed by a trace of which specific file set the base value, which file appended to it, and which file's `:append` or override clause contributed each piece). This provenance information is what actually lets you answer the genuinely useful debugging question — not just "what is the value" (which you might already suspect is wrong) but "WHERE did this wrong value actually come from," letting you trace back to the specific layer/bbappend/machine-config/distro-config file responsible, rather than having to manually grep through every layer's files hoping to spot the relevant assignment by inspection alone.

---

**Q5: Why does Yocto define so many separate path variables (bindir, sbindir, libexecdir, etc.) instead of just using literal paths like "/usr/bin" directly in recipes?**

**A:** Several converging reasons: (1) **FHS compliance and portability across distro policy variations** — while `/usr/bin` is overwhelmingly the standard convention, the FHS-variable abstraction theoretically allows a distro configuration to adjust these conventions (a historically real, if now less common, need — and the `usrmerge` DISTRO_FEATURE transition, where `/bin` became a symlink to `/usr/bin` across the ecosystem, is a concrete recent example of exactly this kind of path-convention shift that recipes using the proper variables handled transparently, while any recipe with hardcoded literal `/bin` paths would have needed individual fixing). (2) **Self-documenting intent** — `install -m 0755 myapp ${D}${sbindir}/` immediately communicates "this is a system administration executable" to a human reader in a way that a literal `/usr/sbin` path doesn't as clearly signal as INTENTIONAL categorization versus an arbitrary hardcoded choice. (3) **Consistency enforcement and tooling support** — using the standard variables means QA tooling (`insane.bbclass`) and packaging logic can reliably reason about WHERE in the filesystem hierarchy a given installed file is meant to live, supporting automated package-splitting heuristics (the default `FILES:${PN}-dev` glob pattern relying on `${includedir}`/`${libdir}/pkgconfig` being used consistently is a direct example) that would be much harder to implement reliably against arbitrary, inconsistently-hardcoded literal paths across thousands of independently-authored recipes. In short: it's the same underlying justification for using named constants instead of magic literal values in any software engineering context, applied specifically to filesystem installation paths.
