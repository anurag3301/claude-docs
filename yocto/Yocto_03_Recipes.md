# Yocto/OpenEmbedded Recipes — Complete Deep Dive

---

## Table of Contents

1. [What Is a Recipe?](#1-what-is-a-recipe)
2. [Recipe File Naming and Versioning](#2-recipe-file-naming-and-versioning)
3. [Anatomy of a Minimal Recipe](#3-anatomy-of-a-minimal-recipe)
4. [Core Metadata Variables](#4-core-metadata-variables)
5. [LICENSE and LIC_FILES_CHKSUM](#5-license-and-lic_files_chksum)
6. [SRC_URI Deep Dive](#6-src_uri-deep-dive)
7. [The Source Directory — S, B, and WORKDIR](#7-the-source-directory--s-b-and-workdir)
8. [Build System Classes (autotools, cmake, meson, make)](#8-build-system-classes-autotools-cmake-meson-make)
9. [The Standard Task Pipeline in Detail](#9-the-standard-task-pipeline-in-detail)
10. [do_configure in Depth](#10-do_configure-in-depth)
11. [do_compile in Depth](#11-do_compile-in-depth)
12. [do_install in Depth](#12-do_install-in-depth)
13. [Packaging — PACKAGES, FILES, and Package Splitting](#13-packaging--packages-files-and-package-splitting)
14. [Dependencies — DEPENDS, RDEPENDS, and the Full Family](#14-dependencies--depends-rdepends-and-the-full-family)
15. [Patches and Patch Management](#15-patches-and-patch-management)
16. [PACKAGECONFIG — Optional Feature Flags](#16-packageconfig--optional-feature-flags)
17. [Cross-Compilation Variables](#17-cross-compilation-variables)
18. [Native and NativeSDK Recipes](#18-native-and-nativesdk-recipes)
19. [Multilib and Multiple Architecture Support](#19-multilib-and-multiple-architecture-support)
20. [systemd and SysVinit Service Integration](#20-systemd-and-sysvinit-service-integration)
21. [User and Group Management (useradd.bbclass)](#21-user-and-group-management-useraddbbclass)
22. [QA Checks (insane.bbclass)](#22-qa-checks-insanebbclass)
23. [Complete Worked Example — A Real Recipe](#23-complete-worked-example--a-real-recipe)
24. [Complete Worked Example — A Python Recipe](#24-complete-worked-example--a-python-recipe)
25. [Common Pitfalls and Debugging](#25-common-pitfalls-and-debugging)
26. [Interview Questions and Answers](#26-interview-questions-and-answers)

---

## 1. What Is a Recipe?

A **recipe** (`.bb` file) is the fundamental unit of build description in OpenEmbedded/Yocto. It tells BitBake everything needed to take a piece of software from "source code somewhere" to "one or more installable binary packages": where to get the source, what it depends on to build, how to configure/compile/install it, what license governs it, and how the resulting files should be split into final packages.

A recipe is **not** a shell script that runs top-to-bottom. It is a declarative description of a set of **tasks**, each with inputs, outputs, and dependencies, that BitBake's scheduler executes in the correct order — fetching from the network only in `do_fetch`, with no network access permitted in later tasks (by design, for build reproducibility and security).

**Conceptual model:**

```
Recipe (.bb file)
   │
   ├── Metadata (DESCRIPTION, LICENSE, HOMEPAGE, ...)
   ├── Source location (SRC_URI, SRCREV)
   ├── Build-time deps (DEPENDS)
   ├── Inherited build logic (inherit autotools/cmake/...)
   ├── Custom task overrides (do_configure:append, etc.)
   └── Packaging rules (PACKAGES, FILES:*, RDEPENDS:*)
          │
          ▼
   BitBake executes: fetch → unpack → patch → configure → compile
                      → install → package → populate_sysroot
          │
          ▼
   Output: one or more .rpm/.deb/.ipk packages, ready to include in an image
```

---

## 2. Recipe File Naming and Versioning

```
<PN>_<PV>.bb

PN = Package Name (e.g., "busybox")
PV = Package Version (e.g., "1.36.1")
```

BitBake automatically derives `PN` and `PV` from the filename unless explicitly overridden:

```bitbake
# File: myapp_2.5.bb
# Automatically: PN = "myapp", PV = "2.5"

# File: myapp_git.bb
# Automatically: PN = "myapp", PV = "git" (literal string "git" unless
# overridden — typically combined with SRCPV for a real version string)
PV = "1.0+git${SRCPV}"
```

### Multiple Versions Coexisting

You can have several version files for the same package in a layer:
```
recipes-myapp/myapp/
├── myapp_1.0.bb
├── myapp_2.0.bb
└── myapp_2.1.bb
```

BitBake picks the **highest version by default** unless `PREFERRED_VERSION_myapp` pins a specific one:
```bitbake
# In machine.conf, distro.conf, or local.conf
PREFERRED_VERSION_myapp = "1.0"
```

### Recipe Skeleton with require/.inc

```
myapp.inc                 ← shared logic across all versions
myapp_1.0.bb               ← require myapp.inc; PV-specific overrides
myapp_2.0.bb               ← require myapp.inc; PV-specific overrides
myapp_2.0.bbappend         ← layer customization on top of 2.0 specifically
```

---

## 3. Anatomy of a Minimal Recipe

```bitbake
SUMMARY = "A simple hello-world utility"
DESCRIPTION = "Prints a greeting to demonstrate a minimal Yocto recipe."
HOMEPAGE = "https://example.com/hello"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://COPYING;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "https://example.com/releases/hello-${PV}.tar.gz"
SRC_URI[sha256sum] = "abcdef0123456789abcdef0123456789abcdef0123456789abcdef01234567"

S = "${WORKDIR}/hello-${PV}"

inherit autotools

DEPENDS = "zlib"
```

That's a fully functional recipe. Everything else (the actual `./configure && make && make install` invocations, packaging logic, sysroot population) comes from `inherit autotools` plus the implicitly-inherited `base.bbclass`.

### Every Recipe Implicitly Inherits base.bbclass

```bitbake
# You never write this explicitly — it's automatic for EVERY recipe:
# inherit base
```

`base.bbclass` provides the default implementations of `do_fetch`, `do_unpack`, `do_patch`, the basic task skeleton, and core variable defaults that every other class builds upon.

---

## 4. Core Metadata Variables

| Variable             | Purpose                                                              | Example                                |
|------------------------|--------------------------------------------------------------------|------------------------------------------|
| `SUMMARY`             | One-line description (shown in package managers)                    | `"Lightweight HTTP server"`              |
| `DESCRIPTION`         | Longer, multi-sentence description                                   | `"A small, fast HTTP server suitable for embedded use."` |
| `HOMEPAGE`            | Upstream project URL                                                 | `"https://nginx.org"`                    |
| `BUGTRACKER`          | Bug tracker URL                                                      | `"https://trac.nginx.org"`               |
| `SECTION`             | Logical grouping for package managers                                | `"net"`, `"devel"`, `"libs"`             |
| `LICENSE`             | SPDX-style license identifier(s)                                     | `"GPL-2.0-only"`, `"MIT \| BSD-3-Clause"`|
| `LIC_FILES_CHKSUM`    | Checksum(s) of license text — verifies license hasn't silently changed| see Section 5                            |
| `PN`                  | Package/recipe name (usually auto-derived from filename)             | `"busybox"`                              |
| `PV`                  | Package version (usually auto-derived from filename)                 | `"1.36.1"`                               |
| `PR`                  | Package revision — bump to force a rebuild without changing PV       | `"r0"`, `"r1"`                           |
| `PE`                  | Package epoch — for version comparison edge cases (rarely needed)    | `"1"`                                     |
| `SRC_URI`             | Where to fetch source from                                            | see Section 6                             |
| `S`                   | Directory (relative to WORKDIR) where source ends up after unpack     | `"${WORKDIR}/${BPN}-${PV}"`               |
| `DEPENDS`             | Build-time dependencies                                              | `"zlib openssl"`                          |
| `RDEPENDS:${PN}`      | Runtime dependencies                                                 | `"bash"`                                   |
| `PROVIDES`            | Additional names this recipe satisfies                               | `"virtual/libgl"`                          |
| `INC_PR`              | Used in shared `.inc` files to centrally bump PR across many recipes  | `"r1"`                                     |
| `CVE_PRODUCT`         | Maps recipe to CVE database product name for vulnerability scanning  | `"openssl:openssl"`                        |
| `CVE_VERSION`         | Override version string used for CVE matching, if different from PV  | `"1.1.1w"`                                 |

### PR — Package Revision

`PR` (and the older convention of bumping it manually) signals to BitBake/the package manager that the **recipe itself changed** (a patch was added, a configure option changed) even though the **upstream version (PV) is identical**. This forces a real rebuild and produces a package that the package manager recognizes as newer than the previous build of the same `PV`.

```bitbake
PR = "r1"   # Bump this every time you make a non-version-bumping recipe change
```

Modern Yocto increasingly relies on the sstate/task-signature mechanism to automatically detect recipe changes rather than requiring manual `PR` bumps for local builds, but `PR` still matters for package manager **version ordering** when publishing package feeds (a binary feed needs PR bumps to know package B is "newer" than package A at the same PV).

---

## 5. LICENSE and LIC_FILES_CHKSUM

### LICENSE

Uses **SPDX license identifiers** (or close variants used historically by OpenEmbedded):

```bitbake
LICENSE = "MIT"
LICENSE = "GPL-2.0-only"
LICENSE = "Apache-2.0"
LICENSE = "BSD-3-Clause"

# Multiple licenses, ANY of which can apply to the WHOLE package (dual-license)
LICENSE = "MIT | Apache-2.0"

# Multiple licenses that ALL apply (different parts under different licenses)
LICENSE = "GPL-2.0-only & LGPL-2.1-only"

# Per-package licensing when a single recipe produces multiple PACKAGES
LICENSE:${PN} = "GPL-2.0-only"
LICENSE:${PN}-doc = "CC-BY-SA-4.0"
```

### LIC_FILES_CHKSUM

This is a **verification mechanism**, not just documentation. BitBake computes the actual checksum of the specified license file(s) at `do_fetch`/`do_unpack` time and compares it against the value you wrote in the recipe. If upstream silently changes their license text (a real, if rare, occurrence — sometimes even maliciously), the build **fails loudly** rather than silently shipping under outdated license assumptions.

```bitbake
# Whole-file checksum, starting at byte 0
LIC_FILES_CHKSUM = "file://COPYING;md5=0835ade698e0bcf8506ecda2f7b4f302"

# Checksum of a SPECIFIC byte range within a larger file (common for license
# text embedded at the top of a source file, e.g., when there's no separate
# LICENSE file)
LIC_FILES_CHKSUM = "file://src/main.c;beginline=1;endline=20;md5=abc123..."

# Multiple license files (e.g., dual-licensed software with two LICENSE files)
LIC_FILES_CHKSUM = "file://LICENSE-MIT;md5=aaa... \
                     file://LICENSE-APACHE;md5=bbb..."
```

### Computing the Checksum

```bash
# First attempt: leave it blank or wrong, let BitBake tell you the correct value
LIC_FILES_CHKSUM = "file://COPYING;md5=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

$ bitbake myrecipe
ERROR: ... LIC_FILES_CHKSUM ... does not match ...
   Expected: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   Got:      0835ade698e0bcf8506ecda2f7b4f302   ← copy this into the recipe
```

This "fail and tell you the right answer" pattern is the standard, accepted workflow for filling in checksums in Yocto recipes (used for `LIC_FILES_CHKSUM`, `SRC_URI[sha256sum]`, etc.)

---

## 6. SRC_URI Deep Dive

(Full fetcher reference is in the BitBake document; this section focuses on **recipe-author-facing** patterns.)

### Combining Multiple Source Components

```bitbake
SRC_URI = "https://example.com/myapp-${PV}.tar.gz \
           file://0001-fix-cross-compile.patch \
           file://0002-disable-tests.patch \
           file://myapp.service \
           file://myapp.cfg \
           "
SRC_URI[sha256sum] = "..."
```

Non-patch files referenced via `file://` (like `myapp.service`, `myapp.cfg` above) are simply copied into `${WORKDIR}` during `do_unpack` — they're commonly then explicitly installed in a custom `do_install:append()`.

### Git with Submodules

```bitbake
SRC_URI = "gitsm://github.com/example/repo-with-submodules.git;protocol=https;branch=main"
SRCREV = "abc123..."
```

`gitsm://` (instead of plain `git://`) tells BitBake's fetcher to also recursively fetch git submodules referenced by `.gitmodules`.

### Specifying Where a Patch Applies (subdir)

```bitbake
# When source has multiple nested components, target a patch at a subdirectory
SRC_URI = "https://example.com/bigproject-${PV}.tar.gz \
           file://fix-submodule-x.patch;patchdir=submodule-x \
           "
```

### Conditional SRC_URI by Override

```bitbake
SRC_URI = "https://example.com/myapp-${PV}.tar.gz"
SRC_URI:append:qemux86-64 = " file://x86-specific.patch"
SRC_URI:append:raspberrypi4 = " file://rpi-specific.patch"
```

---

## 7. The Source Directory — S, B, and WORKDIR

| Variable    | Meaning                                                                |
|---------------|----------------------------------------------------------------------|
| `WORKDIR`    | The recipe's private build-time scratch root: `tmp/work/<arch>/<pn>/<pv-pr>/`|
| `S`          | "Source directory" — where the unpacked, patched source code lives (relative paths resolve under WORKDIR)|
| `B`          | "Build directory" — where `do_configure`/`do_compile` actually run; equals `S` for in-tree builds, or a separate directory for out-of-tree build systems (CMake, Meson commonly build out-of-tree)|
| `D`          | "Destination directory" — the staging root that `do_install` populates (this later becomes the actual package contents); NOT a real system path, just a temporary install root|

```bitbake
# Default for most tarball releases (matches the typical tar.gz internal dir name)
S = "${WORKDIR}/${BP}"             # BP = "${BPN}-${PV}" (base package name + version)

# Common explicit override when the tarball's internal directory name differs
S = "${WORKDIR}/myapp-${PV}-release"

# For git fetches (no internal versioned directory name)
S = "${WORKDIR}/git"

# Out-of-tree build directory (common for cmake.bbclass, meson.bbclass)
B = "${WORKDIR}/build"
```

### Visualizing the Directory Relationships

```
${WORKDIR}/                              ← tmp/work/<arch>/myapp/1.0-r0/
├── myapp-1.0/                           ← ${S} : unpacked, patched source
│   ├── configure
│   ├── Makefile.am
│   └── src/main.c
├── build/                               ← ${B} : out-of-tree build dir (if cmake/meson)
│   ├── CMakeCache.txt
│   └── myapp                            ← compiled binary lands here
├── image/                                ← ${D} : do_install destination root
│   └── usr/
│       └── bin/
│           └── myapp                    ← "installed" here, NOT on real filesystem yet
├── temp/                                 ← task logs (log.do_compile, log.do_install, etc.)
├── recipe-sysroot/                       ← target sysroot (deps for THIS recipe to build against)
└── recipe-sysroot-native/                ← native sysroot (build-host tools for THIS recipe)
```

### Inspecting These Paths for a Real Recipe

```bash
bitbake -e busybox | grep -E '^(WORKDIR|S|B|D)='
```

---

## 8. Build System Classes (autotools, cmake, meson, make)

### autotools.bbclass

For software using GNU Autotools (`autoreconf`/`./configure`/`make`/`make install`):

```bitbake
inherit autotools

# Pass extra arguments to ./configure
EXTRA_OECONF = "--disable-tests --enable-shared --without-x"

# Pass extra arguments to oe_runmake (the wrapped 'make' invocation)
EXTRA_OEMAKE = "CFLAGS='${CFLAGS} -DEXTRA_DEBUG'"
```

`autotools.bbclass` automatically:
- Runs `autoreconf` if needed (regenerating `./configure` from `configure.ac`)
- Calls `./configure` with the correct cross-compilation `--host`/`--build`/`--target` triplets and standard paths (`--prefix=/usr`, etc.)
- Runs `oe_runmake` (a wrapper ensuring the correct cross-compiler and flags are used)
- Runs `oe_runmake install DESTDIR=${D}` for `do_install`

### cmake.bbclass

```bitbake
inherit cmake

EXTRA_OECMAKE = "-DBUILD_TESTING=OFF -DENABLE_FEATURE_X=ON"
```

Automatically:
- Configures an out-of-tree build in `${B}`
- Passes a cross-compilation **CMake toolchain file** (auto-generated, pointing CMake at the right compiler, sysroot, and `find_package`/`find_library` search paths restricted to the target sysroot)
- Runs `cmake --build` and `cmake --install`

### meson.bbclass

```bitbake
inherit meson

EXTRA_OEMESON = "-Dtests=false -Ddocs=disabled"
```

Similarly handles a generated **cross-file** (Meson's equivalent of a CMake toolchain file) so Meson correctly cross-compiles.

### Plain Makefiles (no inherit, or minimal helper)

```bitbake
# No standard build-system class fits — hand-roll using oe_runmake directly
do_compile() {
    oe_runmake CC="${CC}" CFLAGS="${CFLAGS}"
}

do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${B}/myapp ${D}${bindir}/myapp
}
```

`oe_runmake` is a thin wrapper around `make` that ensures `EXTRA_OEMAKE`, the correct parallel job count (`PARALLEL_MAKE`), and standard cross-compilation environment variables are all correctly passed — always prefer it over calling `make` directly in custom recipes.

### setuptools3.bbclass (Python)

```bitbake
inherit setuptools3

DEPENDS += "python3"
RDEPENDS:${PN} += "python3-core"
```

### cargo.bbclass / cargo-bin.bbclass (Rust)

```bitbake
inherit cargo

SRC_URI += "crate://crates.io/serde/1.0.195"
```

---

## 9. The Standard Task Pipeline in Detail

```
do_fetch
    ↓ (downloads SRC_URI into DL_DIR; network access ONLY allowed here)
do_unpack
    ↓ (extracts tarballs / checks out git into ${WORKDIR}, populating ${S})
do_patch
    ↓ (applies every file://*.patch in SRC_URI, in listed order, via 'git am'-like mechanism)
do_prepare_recipe_sysroot
    ↓ (links/copies DEPENDS' staged sysroot output into recipe-sysroot)
do_configure
    ↓ (./configure, cmake, meson setup — prepares the build system)
do_compile
    ↓ (make, ninja — actually compiles the software)
do_install
    ↓ (make install DESTDIR=${D} — stages output into the destination root)
do_package
    ↓ (splits ${D} into logical PACKAGES based on FILES:* variables)
do_packagedata
    ↓ (records package metadata: deps, file lists — used by other recipes/images)
do_package_write_rpm / _deb / _ipk
    ↓ (generates the actual binary package file(s) in the chosen format)
do_populate_sysroot
    ↓ (stages headers/libs/pkg-config files for OTHER recipes that DEPEND on this one)
do_build
    (top-level meta-task; depends on all of the above — what "bitbake <recipe>" targets by default)
```

### Task Logs

```bash
# Every task's full stdout/stderr is logged separately
ls tmp/work/<arch>/<pn>/<pv-pr>/temp/
log.do_fetch
log.do_unpack
log.do_patch
log.do_configure
log.do_compile
log.do_install
log.do_package

# Quickly view the most recent failure's full log
cat tmp/work/<arch>/<pn>/<pv-pr>/temp/log.do_compile
```

---

## 10. do_configure in Depth

For recipes inheriting `autotools`, `cmake`, or `meson`, you almost never write `do_configure` yourself — you tune it via `EXTRA_OECONF`/`EXTRA_OECMAKE`/`EXTRA_OEMESON`. But understanding what happens underneath matters for debugging:

```bitbake
# autotools.bbclass effectively does (conceptually):
do_configure() {
    cd ${S}
    ./configure \
        --build=${BUILD_SYS} \
        --host=${HOST_SYS} \
        --target=${TARGET_SYS} \
        --prefix=${prefix} \
        --exec_prefix=${exec_prefix} \
        --bindir=${bindir} \
        --sbindir=${sbindir} \
        --libexecdir=${libexecdir} \
        --datadir=${datadir} \
        --sysconfdir=${sysconfdir} \
        --sharedstatedir=${sharedstatedir} \
        --localstatedir=${localstatedir} \
        --libdir=${libdir} \
        --includedir=${includedir} \
        --oldincludedir=${oldincludedir} \
        --infodir=${infodir} \
        --mandir=${mandir} \
        ${EXTRA_OECONF}
}
```

### Custom do_configure (when no build-system class applies)

```bitbake
do_configure() {
    # Hand-written configure logic, e.g., generating a config header
    echo "#define VERSION \"${PV}\"" > ${S}/config.h
}
```

### Adding Steps Before/After the Inherited Configure

```bitbake
do_configure:prepend() {
    # Runs BEFORE autotools.bbclass's do_configure
    sed -i 's/OLD_DEFAULT/NEW_DEFAULT/' ${S}/configure.ac
    # Note: if you edit configure.ac, you may need autoreconf to re-run, which
    # autotools.bbclass handles automatically via AUTOTOOLS_AUTO_RECONF
}

do_configure:append() {
    # Runs AFTER autotools.bbclass's do_configure
    echo "Configuration complete for ${PN} ${PV}"
}
```

---

## 11. do_compile in Depth

```bitbake
# What autotools.bbclass/cmake.bbclass effectively do underneath:
do_compile() {
    oe_runmake
}
```

### Common Custom do_compile Patterns

```bitbake
# Build only a specific Makefile target
do_compile() {
    oe_runmake myapp-binary
}

# Pass extra environment for the build
do_compile() {
    export MY_BUILD_FLAG="1"
    oe_runmake
}

# Compile a standalone C file with no build system at all
do_compile() {
    ${CC} ${CFLAGS} ${LDFLAGS} -o myapp ${S}/main.c
}
```

### Important Compiler/Linker Variables (auto-set by the cross-toolchain machinery)

| Variable     | Meaning                                              |
|---------------|---------------------------------------------------|
| `CC`         | The cross C compiler (e.g., `arm-poky-linux-gnueabi-gcc`)|
| `CXX`        | The cross C++ compiler                                |
| `CFLAGS`     | C compiler flags (optimization, target arch tuning)    |
| `CXXFLAGS`   | C++ compiler flags                                     |
| `LDFLAGS`    | Linker flags                                            |
| `CPPFLAGS`   | C preprocessor flags (include paths, defines)            |
| `AR`         | Cross archiver (for static libraries)                    |
| `LD`         | Cross linker                                              |
| `STRIP`      | Cross strip tool (used by do_package to strip debug symbols)|

These are never hand-set in most recipes — they're correctly populated by the cross-compilation toolchain classes based on `TARGET_ARCH`/`TUNE_FEATURES`/`MACHINE`.

---

## 12. do_install in Depth

`do_install` is where you take the compiled build output and place it into `${D}` (the destination/staging root) using paths that mirror where files should ultimately end up on the **target's real root filesystem** (e.g., `${D}${bindir}/myapp` becomes `/usr/bin/myapp` on the target).

```bitbake
do_install() {
    install -d ${D}${bindir}
    install -m 0755 ${B}/myapp ${D}${bindir}/myapp

    install -d ${D}${sysconfdir}
    install -m 0644 ${S}/myapp.conf ${D}${sysconfdir}/myapp.conf

    install -d ${D}${systemd_system_unitdir}
    install -m 0644 ${WORKDIR}/myapp.service ${D}${systemd_system_unitdir}/myapp.service

    install -d ${D}${libdir}
    install -m 0755 ${B}/libmyapp.so.1.0 ${D}${libdir}/
    ln -sf libmyapp.so.1.0 ${D}${libdir}/libmyapp.so.1
    ln -sf libmyapp.so.1   ${D}${libdir}/libmyapp.so
}
```

### Standard Path Variables (FHS-aligned)

| Variable          | Default Path        | Purpose                                |
|---------------------|----------------------|-------------------------------------------|
| `${prefix}`        | `/usr`               | Root of the installation hierarchy        |
| `${bindir}`        | `/usr/bin`            | User executables                           |
| `${sbindir}`       | `/usr/sbin`           | System admin executables                    |
| `${libdir}`        | `/usr/lib`            | Libraries                                   |
| `${includedir}`    | `/usr/include`        | C/C++ header files                          |
| `${sysconfdir}`    | `/etc`                | Configuration files                          |
| `${datadir}`       | `/usr/share`          | Architecture-independent data                |
| `${localstatedir}` | `/var`                | Variable runtime data                         |
| `${sharedstatedir}`| `/com` (or `/var/lib`)| Shared mutable architecture-independent data|
| `${docdir}`        | `/usr/share/doc`       | Documentation                                 |
| `${mandir}`        | `/usr/share/man`       | Man pages                                     |
| `${systemd_system_unitdir}`| `/lib/systemd/system`| systemd unit files (when `inherit systemd`)|

**Always use these variables instead of hardcoding `/usr/bin` etc.** — this is what allows the same recipe to work correctly even if a distro configuration changes the FHS layout (rare, but supported), and it's considered a hard requirement for upstreamable OpenEmbedded recipes.

### `do_install:append()` for bbappend Customization

```bitbake
# In a .bbappend, adding an extra config file without touching the original recipe
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"
SRC_URI += "file://myapp-override.conf"

do_install:append() {
    install -m 0644 ${WORKDIR}/myapp-override.conf ${D}${sysconfdir}/myapp.conf
}
```

---

## 13. Packaging — PACKAGES, FILES, and Package Splitting

A single recipe can (and often does) produce **multiple binary packages** from one `do_install` output — separating the runtime binary, development headers, debug symbols, documentation, and locale data into independently installable packages. This is standard Linux packaging practice (mirroring how distros split `-dev`, `-dbg`, `-doc` packages).

### Default PACKAGES (set by package.bbclass, inherited by everyone)

```bitbake
PACKAGES = "${PN}-dbg ${PN} ${PN}-staticdev ${PN}-dev ${PN}-doc ${PN}-locale ${PN}-src"
```

You rarely need to redeclare `PACKAGES` from scratch — usually you ADD to it for a recipe-specific extra split:

```bitbake
PACKAGES =+ "${PN}-tools ${PN}-examples"

FILES:${PN}-tools = "${bindir}/myapp-converter ${bindir}/myapp-validator"
FILES:${PN}-examples = "${datadir}/myapp/examples/*"
```

### FILES — What Goes in Each Package

```bitbake
# Default FILES:${PN} (main package) typically includes:
FILES:${PN} = "${bindir}/* ${sbindir}/* ${libexecdir}/* ${libdir}/lib*${SOLIBS} \
                ${sysconfdir} ${sharedstatedir} ${localstatedir} \
                ${datadir}/${PN} ${datadir}/pixmaps ..."

# -dev package conventionally gets headers, .so symlinks (not the versioned .so.X), .pc files
FILES:${PN}-dev = "${includedir} ${libdir}/lib*${SOLIBSDEV} ${libdir}/*.la \
                    ${libdir}/pkgconfig ${datadir}/pkgconfig ${datadir}/aclocal"

# -staticdev gets .a static libraries
FILES:${PN}-staticdev = "${libdir}/*.a"

# -doc gets man pages, info pages, and ${docdir}
FILES:${PN}-doc = "${docdir} ${mandir} ${infodir} ..."
```

### Order Matters — First Match Wins

Packages are matched **in the order listed in `PACKAGES`**, and a file is assigned to the FIRST package (in that order) whose `FILES` glob matches it. This is why more specific packages (like `${PN}-tools`) must come BEFORE the catch-all `${PN}` in the `PACKAGES` list — otherwise the generic main package would claim those files first.

```bitbake
PACKAGES =+ "${PN}-tools"
# ${PN}-tools must come before ${PN} in PACKAGES (achieved via =+ prepend operator)
# so its FILES match is checked first
```

### Empty Package Warnings and ALLOW_EMPTY

```bitbake
# If a declared package ends up with zero matched files, BitBake warns by default
# (and the package manager will typically skip producing it). To explicitly allow
# an intentionally-empty "meta" package (e.g., one that exists only to pull in
# RDEPENDS):
ALLOW_EMPTY:${PN}-meta = "1"
```

---

## 14. Dependencies — DEPENDS, RDEPENDS, and the Full Family

| Variable                | Time      | Meaning                                                       |
|---------------------------|-----------|------------------------------------------------------------------|
| `DEPENDS`                | Build     | This recipe needs another recipe's sysroot output to BUILD       |
| `RDEPENDS:${PN}`         | Runtime   | This package needs another package INSTALLED for it to RUN       |
| `RRECOMMENDS:${PN}`      | Runtime   | Soft/optional runtime dependency — installed if available, build doesn't fail if missing|
| `RSUGGESTS:${PN}`        | Runtime   | Even softer suggestion — package manager hint only                |
| `RCONFLICTS:${PN}`       | Runtime   | Cannot coexist with the named package                              |
| `RPROVIDES:${PN}`        | Runtime   | This package also satisfies another package name (virtual provider at runtime)|
| `RREPLACES:${PN}`        | Runtime   | This package replaces/supersedes the named package on upgrade       |

```bitbake
DEPENDS = "zlib openssl glib-2.0"

RDEPENDS:${PN} = "bash python3-core"
RRECOMMENDS:${PN} = "myapp-plugin-extra"
RCONFLICTS:${PN} = "old-myapp"
```

### Why RDEPENDS Needs `:${PN}` But DEPENDS Doesn't

`DEPENDS` applies to the whole recipe (there's only one build process), so it's a simple variable. `RDEPENDS`, `RRECOMMENDS`, etc. apply **per output package** (since one recipe can produce many packages, each potentially needing different runtime dependencies), so they must specify which package they apply to via the override syntax: `RDEPENDS:${PN}` (the main package), `RDEPENDS:${PN}-dev` (the dev package, if it needs something beyond the auto-generated shlib deps), etc.

### Automatic Shared Library Dependency Detection

You almost never need to manually add `RDEPENDS` for shared library linkage — BitBake's packaging step automatically scans compiled binaries (via `objdump`/`readelf`) for their `DT_NEEDED` entries and maps each needed `.so` back to the package that provides it, auto-populating `RDEPENDS` accordingly. Manual `RDEPENDS` is mainly needed for: scripting language dependencies (a shell script `#!/bin/bash` needing `RDEPENDS += "bash"`), runtime-only tool invocations (a program that `exec()`s another program), and configuration-file-driven dependencies.

---

## 15. Patches and Patch Management

### Adding a Patch to a Recipe

```bitbake
SRC_URI += "file://0001-fix-buffer-overflow.patch"
```

By convention, patches are placed in a `files/` subdirectory next to the recipe:
```
recipes-core/busybox/
├── busybox_1.36.1.bb
└── files/
    └── 0001-fix-buffer-overflow.patch
```

### Patch Format Requirements

Patches must be in **unified diff format** (`diff -u` or `git diff`/`git format-patch` output). Git-format patches (with commit message headers) are preferred since they self-document intent:

```bitbake
# Generating a proper patch from a git-tracked source change:
$ cd source-tree
$ git diff > 0001-fix-buffer-overflow.patch
# OR, better, from a committed change:
$ git format-patch -1 HEAD
```

### Patch Application Order and Failure Debugging

Patches in `SRC_URI` are applied **in the order listed**, each against the result of the previous patch. If a patch fails to apply (commonly after a version upgrade where upstream code shifted), BitBake's `do_patch` fails with a clear error showing which patch and which hunk failed.

```bash
# Debug a failing patch interactively
bitbake -c devshell myrecipe
# Inside devshell, manually try:
patch -p1 --dry-run < /path/to/0001-fix-buffer-overflow.patch
# See exactly which hunks fail and why
```

### patch.bbclass Variables

```bitbake
# Specify a non-default patch strip level (default is -p1, like git)
SRC_URI = "file://weird-patch.patch;striplevel=0"

# Apply a patch only to a specific subdirectory within ${S}
SRC_URI = "file://submodule.patch;patchdir=libs/submodule"
```

---

## 16. PACKAGECONFIG — Optional Feature Flags

`PACKAGECONFIG` is the standard mechanism for letting a recipe's optional build-time features be **toggled by downstream layers/users** without editing the recipe itself — typically mapping to `./configure --enable-X`/`--disable-X` or CMake `-DWITH_X=ON/OFF` flags.

### Defining PACKAGECONFIG Options in a Recipe

```bitbake
# Syntax: PACKAGECONFIG[featurename] = "enable-arg,disable-arg,build-deps,runtime-deps"

PACKAGECONFIG ??= "ssl zlib"

PACKAGECONFIG[ssl]   = "--with-ssl,--without-ssl,openssl"
PACKAGECONFIG[zlib]  = "--with-zlib,--without-zlib,zlib"
PACKAGECONFIG[ldap]  = "--with-ldap,--without-ldap,openldap"
PACKAGECONFIG[lua]   = "--with-lua,--without-lua,lua,lua"
PACKAGECONFIG[debug] = "--enable-debug,--disable-debug,,"
```

Each field (comma-separated):
1. Argument passed to `./configure`/cmake when the feature is **enabled**
2. Argument passed when the feature is **disabled**
3. Extra `DEPENDS` to add when enabled
4. Extra `RDEPENDS` to add when enabled

### Toggling PACKAGECONFIG from a bbappend or local.conf

```bitbake
# Turn ON the ldap feature (adds to the default set)
PACKAGECONFIG:append = " ldap"

# Replace the ENTIRE set (turns OFF zlib/ssl unless re-added)
PACKAGECONFIG = "ssl ldap"

# Turn OFF a specific feature
PACKAGECONFIG:remove = "zlib"
```

This pattern is extremely common in real-world layer customization — e.g., disabling SSL support in curl for a minimal image, or enabling a hardware-specific codec in gstreamer-plugins only for boards that have the corresponding hardware.

---

## 17. Cross-Compilation Variables

| Variable        | Meaning                                                              |
|-------------------|-------------------------------------------------------------------|
| `TARGET_ARCH`     | CPU architecture of the final target (e.g., `arm`, `aarch64`, `x86_64`)|
| `TARGET_OS`       | Target OS string (e.g., `linux`, `linux-gnueabi`)                   |
| `TARGET_SYS`      | Full target system triplet (e.g., `arm-poky-linux-gnueabi`)          |
| `TARGET_PREFIX`   | Cross-toolchain binary prefix (e.g., `arm-poky-linux-gnueabi-`)       |
| `HOST_SYS`        | The system the compiled binary will RUN on — usually same as TARGET_SYS for normal recipes|
| `BUILD_SYS`       | The system doing the COMPILING (the build machine, e.g., `x86_64-linux`)|
| `MACHINE`         | The specific board/machine configuration in use (e.g., `raspberrypi4`)|
| `TUNE_FEATURES`   | CPU tuning features active (e.g., `aarch64 cortexa72`)                |
| `SDKMACHINE`      | The architecture of the machine the SDK itself runs on                |

### The Three-System Model (build / host / target)

This terminology, inherited from GNU Autotools cross-compilation conventions, is essential to understand:

- **build** = the machine BitBake is running compilation ON (your dev PC or build server)
- **host** = the machine the COMPILED OUTPUT will RUN on (for a normal application recipe, this equals the target embedded board; for a `-native` recipe, this equals "build" itself)
- **target** = mostly synonymous with "host" in OpenEmbedded's normal usage, but distinct when cross-cross-compiling a compiler itself (e.g., `gcc-cross-canadian` for SDKs, where build/host/target are all three genuinely different machines)

```bitbake
# A normal application recipe: build=devPC, host=target=embedded board
inherit autotools
# automatically gets correct --build/--host triplets

# A -native recipe (e.g., quilt-native, used as a build-time TOOL):
inherit native
# build=devPC, host=devPC (it RUNS on the dev machine, not the target)
```

---

## 18. Native and NativeSDK Recipes

### -native Recipes

Some tools are needed **during the build process itself**, running on the build host, not on the target device — for example, a code generator, a native compiler used to cross-compile another compiler, or `qemu-native` (used to run target-architecture test binaries during a build via QEMU user-mode emulation).

```bitbake
# myapp-native.bb (or commonly, derived automatically from myapp.bb via BBCLASSEXTEND)
inherit native
```

### BBCLASSEXTEND — One Recipe, Multiple Variants

```bitbake
# In myapp.bb — build BOTH a normal target version AND a native version
# from the same recipe logic
BBCLASSEXTEND = "native nativesdk"
```

This is extremely common for libraries/tools needed both on-target and as a build-time dependency (e.g., a code-generation tool whose target binary the device uses, but a native copy is also needed during other recipes' `do_compile` to generate code on the build host).

```bash
# Build the native variant explicitly
bitbake myapp-native

# DEPENDS on the native variant from another recipe
DEPENDS = "myapp-native"
```

### -nativesdk Recipes

Built to run on the **SDK host machine** (which might differ from BOTH the original build machine and the target — e.g., building an SDK on a Linux build server that will be installed and run on a developer's macOS or Windows-WSL machine). Used for tools shipped inside generated SDKs (`populate_sdk`).

```bitbake
inherit nativesdk
```

---

## 19. Multilib and Multiple Architecture Support

Multilib allows a single image to include packages built for **multiple architectures/ABIs simultaneously** (most commonly: 32-bit and 64-bit variants on the same 64-bit-capable target, similar to `lib` vs `lib32`/`lib64` on desktop Linux distros).

```bitbake
# In local.conf or distro config:
MULTILIBS = "multilib:lib32"
DEFAULTTUNE:virtclass-multilib-lib32 = "x86"

# Recipe-level: build a lib32 variant of a specific recipe
BBCLASSEXTEND = "..."   # multilib is usually driven by global config, not per-recipe
```

```bitbake
# Including both architecture variants of a package in an image:
IMAGE_INSTALL:append = " lib32-mylib mylib"
```

This is a relatively advanced/niche topic mostly relevant to platforms needing to run legacy 32-bit binaries alongside a 64-bit primary userspace.

---

## 20. systemd and SysVinit Service Integration

### systemd.bbclass

```bitbake
inherit systemd

SYSTEMD_SERVICE:${PN} = "myapp.service"
SYSTEMD_AUTO_ENABLE:${PN} = "enable"     # or "disable" — controls default enablement

SRC_URI += "file://myapp.service"

do_install:append() {
    install -d ${D}${systemd_system_unitdir}
    install -m 0644 ${WORKDIR}/myapp.service ${D}${systemd_system_unitdir}/
}
```

`systemd.bbclass` automatically handles:
- Adding the package to the right `FILES:${PN}` for the unit file
- Generating the correct postinst/postrm scriptlets to enable/disable the service via `systemctl` on first install
- Adding `RDEPENDS` on `systemd` core packages as needed

### update-rc.d.bbclass (SysVinit, for systems without systemd)

```bitbake
inherit update-rc.d

INITSCRIPT_NAME = "myapp"
INITSCRIPT_PARAMS = "defaults 90"

SRC_URI += "file://myapp.init"

do_install:append() {
    install -d ${D}${sysconfdir}/init.d
    install -m 0755 ${WORKDIR}/myapp.init ${D}${sysconfdir}/init.d/myapp
}
```

### Supporting Both (common in distros that allow either init system)

```bitbake
inherit systemd update-rc.d

SYSTEMD_SERVICE:${PN} = "myapp.service"
INITSCRIPT_NAME = "myapp"
INITSCRIPT_PARAMS = "defaults 90"
# At image build time, only the relevant scriptlets activate based on
# DISTRO_FEATURES containing "systemd" or "sysvinit"
```

---

## 21. User and Group Management (useradd.bbclass)

For daemons that should run as a dedicated, non-root user:

```bitbake
inherit useradd

USERADD_PACKAGES = "${PN}"
USERADD_PARAM:${PN} = "--system --no-create-home --shell /sbin/nologin myapp-user"
GROUPADD_PARAM:${PN} = "--system myapp-group"

# If the binary needs special capabilities/permissions set at install time
do_install:append() {
    chown myapp-user:myapp-group ${D}${localstatedir}/lib/myapp
}
```

`useradd.bbclass` ensures the specified `useradd`/`groupadd` commands are correctly run **on the target** at first-boot/first-install time (via package postinst scriptlets), not on the build host (which would be meaningless, since the build host's UID namespace has nothing to do with the target's).

---

## 22. QA Checks (insane.bbclass)

`insane.bbclass` (implicitly inherited via `package.bbclass`) runs a battery of automated sanity checks on every recipe's packaged output, catching extremely common, easy-to-miss packaging mistakes:

| Check               | Catches                                                              |
|------------------------|--------------------------------------------------------------------|
| `already-stripped`    | A binary was already stripped before BitBake's own stripping step (suggests wrong build flags)|
| `ldflags`              | Compiled binaries that don't honor `LDFLAGS` (often missing `-Wl,...` security hardening flags)|
| `rpaths`               | Binaries containing bad/build-host-specific `RPATH` entries that would break on-target|
| `useless-rpaths`       | RPATH entries pointing at standard system library directories (redundant)|
| `dev-so`               | A `.so` symlink incorrectly placed in the main package instead of `-dev`|
| `staticdev`            | A `.a` static library incorrectly placed outside `-staticdev`        |
| `packages-list`        | Duplicate package names                                                |
| `files-invalid`        | A `FILES:*` variable referencing a path that doesn't exist after install|
| `installed-vs-shipped` | Files installed to `${D}` but not included in ANY package's `FILES` (silently dropped — usually a packaging bug)|
| `unsafe-references-in-binaries` | Hardcoded build-host paths baked into the binary           |
| `host-user-contaminated`| Files owned by the build-host's actual UID instead of the expected target UID|
| `symlink-to-sysroot`   | A symlink in the package pointing into the build sysroot (build-host-only path) — would break at runtime on target|
| `license-checksum`     | Mismatch between recorded and actual license file checksum (ties to LIC_FILES_CHKSUM)|

### Selectively Disabling a QA Check (use sparingly, with a clear comment why)

```bitbake
INSANE_SKIP:${PN} = "dev-so"
# Reason: this package intentionally ships a .so symlink in the main package
# because <specific justified reason>, not the usual -dev convention.
```

Disabling QA checks should be rare and well-justified — they exist because each one represents a real, previously-encountered class of bug that broke real builds or runtime behavior.

---

## 23. Complete Worked Example — A Real Recipe

```bitbake
SUMMARY = "Lightweight system monitoring daemon"
DESCRIPTION = "myapp-monitor collects system metrics and exposes them via a \
local Unix socket for consumption by other tools."
HOMEPAGE = "https://github.com/example/myapp-monitor"
BUGTRACKER = "https://github.com/example/myapp-monitor/issues"
SECTION = "console/utils"

LICENSE = "GPL-2.0-only"
LIC_FILES_CHKSUM = "file://COPYING;md5=b234ee4d69f5fce4486a80fdaf4a4263"

SRC_URI = "git://github.com/example/myapp-monitor.git;protocol=https;branch=main \
           file://0001-fix-musl-build.patch \
           file://myapp-monitor.service \
           file://myapp-monitor.conf \
           "
SRCREV = "9f3c2a1b8e7d6f5a4c3b2a1908765432fedcba98"
PV = "1.4.0+git${SRCPV}"

S = "${WORKDIR}/git"

inherit cmake systemd useradd pkgconfig

DEPENDS = "zlib libcap"
RDEPENDS:${PN} = "bash"

EXTRA_OECMAKE = "-DBUILD_TESTING=OFF"

PACKAGECONFIG ??= "ssl"
PACKAGECONFIG[ssl] = "-DWITH_SSL=ON,-DWITH_SSL=OFF,openssl"
PACKAGECONFIG[json] = "-DWITH_JSON=ON,-DWITH_JSON=OFF,jansson,jansson"

SYSTEMD_SERVICE:${PN} = "myapp-monitor.service"
SYSTEMD_AUTO_ENABLE:${PN} = "enable"

USERADD_PACKAGES = "${PN}"
USERADD_PARAM:${PN} = "--system --no-create-home --shell /sbin/nologin myapp-monitor"

do_install:append() {
    install -d ${D}${systemd_system_unitdir}
    install -m 0644 ${WORKDIR}/myapp-monitor.service ${D}${systemd_system_unitdir}/

    install -d ${D}${sysconfdir}/myapp-monitor
    install -m 0644 ${WORKDIR}/myapp-monitor.conf ${D}${sysconfdir}/myapp-monitor/

    install -d ${D}${localstatedir}/lib/myapp-monitor
}

FILES:${PN} += "${systemd_system_unitdir}/myapp-monitor.service \
                ${localstatedir}/lib/myapp-monitor"

PACKAGES =+ "${PN}-tools"
FILES:${PN}-tools = "${bindir}/myapp-monitor-cli"
RDEPENDS:${PN}-tools = "${PN}"
```

This single recipe demonstrates: git fetching with a patch, service files, multiple inherited classes, native dependencies, a PACKAGECONFIG feature toggle, systemd integration, dedicated user creation, custom install logic, additional FILES assignment, and a secondary split package — essentially the full toolkit a real-world recipe author uses.

---

## 24. Complete Worked Example — A Python Recipe

```bitbake
SUMMARY = "A small Python utility for parsing sensor logs"
HOMEPAGE = "https://github.com/example/sensorparse"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://LICENSE;md5=e23fadd6ceef8c618fc1cb4339a01e29"

SRC_URI = "git://github.com/example/sensorparse.git;protocol=https;branch=main"
SRCREV = "abc123def456..."
PV = "1.0+git${SRCPV}"

S = "${WORKDIR}/git"

inherit setuptools3

DEPENDS = "python3"
RDEPENDS:${PN} = "python3-core python3-json python3-argparse"

# Often Python recipes need NO custom do_install at all — setuptools3.bbclass
# correctly handles "pip install ." semantics into ${D}, respecting setup.py/
# pyproject.toml entry_points to auto-generate the right /usr/bin scripts.
```

### Python-Specific Patterns

```bitbake
# Forcing a specific Python provider/version interaction
RDEPENDS:${PN} += "python3-core"

# Including optional extras only when a PACKAGECONFIG feature is on
PACKAGECONFIG ??= ""
PACKAGECONFIG[plotting] = ",,,python3-matplotlib"

# For packages using pyproject.toml + a PEP517 build backend instead of setup.py
inherit python_pep517
```

---

## 25. Common Pitfalls and Debugging

### 1. Forgetting LIC_FILES_CHKSUM after a license file changes upstream
After a `devtool upgrade`, if the new version's COPYING file changed even by a single byte (a copyright year bump is enough), the build fails with a checksum mismatch. This is by design — review the actual diff to confirm it's benign before updating the checksum.

### 2. Files installed but missing from any package (installed-vs-shipped QA error)
Happens when `do_install` creates a file but no `FILES:*` variable's glob pattern matches its path. Either it belongs in an existing package's FILES (extend the glob) or needs an explicit additional `FILES:${PN} +=` line.

### 3. PACKAGECONFIG option silently ignored
The feature name in `PACKAGECONFIG[name]` must exactly match what you reference in `PACKAGECONFIG ??=`/`PACKAGECONFIG:append`. A typo means the option block is defined but never activated, and the corresponding configure/cmake argument never gets added — surprisingly common and easy to miss since BitBake doesn't error on unused PACKAGECONFIG option definitions.

### 4. Recipe builds fine standalone but fails when included in an image
Usually a missing `RDEPENDS`. The recipe links against a library that happens to already be present in your dev/test environment's sysroot incidentally (from another recipe's build), but the actual runtime package dependency was never declared, so a minimal image lacking that other package fails at runtime with a missing `.so` error. Always verify with `objdump -p <binary> | grep NEEDED` against the final package's declared RDEPENDS.

### 5. `do_compile` succeeds locally via devshell but fails in the real BitBake task
devshell sets up the environment slightly differently for interactive use; a build that "works in devshell" but fails in the real task often points to a `PATH`/environment variable assumption that doesn't hold during the real, non-interactive task execution. Check `log.do_compile` carefully for the exact failing command and compare environment variables.

### 6. Patch applies cleanly with `patch -p1` manually but fails in do_patch
Usually a striplevel mismatch — BitBake's patch fetcher defaults to `-p1` like git, but a patch generated with a different tool/convention might need `;striplevel=0` or a different value explicitly specified in `SRC_URI`.

### 7. systemd service not starting on first boot despite SYSTEMD_AUTO_ENABLE = "enable"
Check `DISTRO_FEATURES` actually includes `"systemd"` (otherwise `systemd.bbclass`'s scriptlet generation is silently skipped entirely) and that `VIRTUAL-RUNTIME_init_manager` is set to `systemd` in the distro/machine configuration — both conditions are required for systemd integration to be active.

---

## 26. Interview Questions and Answers

**Q1: Walk through what happens, task by task, when you run `bitbake myrecipe` for the very first time.**

**A:** BitBake first parses all configuration files and the recipe itself, resolving the complete dependency graph (both recipe-level DEPENDS and task-level dependencies). Then it executes tasks in dependency order: `do_fetch` downloads everything listed in SRC_URI into DL_DIR (with checksum verification for non-git sources); `do_unpack` extracts/checks out that source into `${S}` under `${WORKDIR}`; `do_patch` applies every patch listed in SRC_URI sequentially; `do_prepare_recipe_sysroot` stages the build-time dependencies (from DEPENDS) into a recipe-specific sysroot; `do_configure` runs the appropriate configuration step (./configure, cmake, meson — whatever the inherited build-system class implements), `do_compile` runs the actual build (make/ninja); `do_install` copies build output into `${D}` using FHS-style paths; `do_package` splits `${D}`'s contents into the declared PACKAGES based on FILES:* matching, running QA checks (insane.bbclass) along the way; `do_package_write_<format>` generates the actual .rpm/.deb/.ipk files; and `do_populate_sysroot` stages this recipe's own headers/libraries so OTHER recipes that DEPEND on it can build against it. Finally `do_build`, the top-level meta-task with no real work of its own, simply depends on all of this having completed — which is what "building the recipe" ultimately means from BitBake's perspective.

---

**Q2: What's the difference between S, B, D, and WORKDIR, and why does this distinction matter?**

**A:** WORKDIR is the recipe's entire private build scratch space root (`tmp/work/<arch>/<pn>/<pv-pr>/`). S ("source") is where the unpacked, patched source code physically lives — usually a subdirectory of WORKDIR. B ("build") is where the actual configure/compile commands run; for in-tree build systems (most Autotools projects) B equals S, but for build systems that conventionally build out-of-tree (CMake, Meson typically do this) B is a separate directory, keeping generated build artifacts cleanly separated from the pristine source tree. D ("destination") is a completely different kind of directory — it's a temporary, fake root filesystem that `do_install` populates using real target-style absolute paths (`${D}${bindir}/myapp` meaning "this file will eventually be `/usr/bin/myapp` on the real target"), which `do_package` later reads from to assemble the actual binary packages. This distinction matters because confusing these (e.g., installing directly into `/usr/bin` on the build host instead of `${D}/usr/bin`) would corrupt the build host's own filesystem and produce no usable cross-compiled package at all — D exists specifically to let the install step run using normal Unix tools (`install`, `cp`) while keeping everything safely contained and relocatable.

---

**Q3: Why does package splitting (PACKAGES/FILES) exist, and how does BitBake decide which files go into which package?**

**A:** Splitting a single piece of upstream software into multiple installable packages (the runtime binary, `-dev` headers/static-link files, `-dbg` debug symbols, `-doc` documentation, `-staticdev` static libraries, `-locale` translation files) lets a minimal embedded image install ONLY the runtime piece it actually needs at runtime, without dragging in development headers or documentation that bloat the image and serve no purpose on a deployed device — while still letting a developer's SDK/devshell environment pull in the `-dev` package when actually building software against that library. BitBake decides package membership during `do_package` by iterating through the `PACKAGES` list IN ORDER, and for each package, checking every file remaining in `${D}` (not yet claimed by an earlier package in the list) against that package's `FILES:<pkgname>` glob patterns — first match wins, and any files not matching ANY listed package's FILES patterns are flagged by the `installed-vs-shipped` QA check as a likely packaging bug (the file was built and installed but will be silently lost since no package will contain it). This is exactly why ordering matters: a more-specific package like `${PN}-tools` must be listed before the generic catch-all `${PN}` package, or `${PN}`'s broad FILES glob (e.g., `${bindir}/*`) would claim those files first.

---

**Q4: Explain PACKAGECONFIG and give a concrete reason a BSP layer might use it.**

**A:** PACKAGECONFIG is a standardized mechanism for a recipe to expose optional, toggleable build-time features without requiring downstream consumers to fork or directly edit the recipe. The recipe author defines each option as `PACKAGECONFIG[name] = "enable-arg,disable-arg,build-deps,runtime-deps"`, and a default active set via `PACKAGECONFIG ??= "feature1 feature2"`. A BSP (Board Support Package) layer, machine configuration, or distro configuration can then append, remove, or completely replace the active set using the normal override operators (`PACKAGECONFIG:append`, `PACKAGECONFIG:remove`) without ever touching the original recipe file. A concrete real-world example: gstreamer1.0-plugins-bad has a PACKAGECONFIG option for a specific hardware video decoder plugin that depends on a vendor-proprietary library only present on certain SoCs. A generic image (qemu, or a board without that hardware) leaves that PACKAGECONFIG option disabled (the default), while a BSP layer for the specific board that DOES have that hardware decoder adds `PACKAGECONFIG:append:pn-gstreamer1.0-plugins-bad = " hwcodec"` in its machine configuration, enabling hardware-accelerated decode only where the corresponding hardware and proprietary library actually exist — all from the exact same shared upstream recipe.

---

**Q5: What does LIC_FILES_CHKSUM actually verify, and why is it not just a formality?**

**A:** It's a cryptographic checksum (commonly MD5, sometimes combined with byte-range specifiers for license text embedded within a larger source file) of the actual license text file(s) referenced from SRC_URI, computed fresh every time the source is fetched/unpacked and compared against the value recorded in the recipe. This is not mere documentation — it's an active integrity gate. If upstream silently changes the license terms between versions (this has genuinely happened — projects relicensing, or in rarer cases a compromised release tarball with subtly altered license text), the build FAILS LOUDLY with a checksum mismatch rather than silently shipping a product under license terms the recipe author never actually reviewed or agreed to comply with. For organizations doing license compliance auditing across thousands of recipes in a large embedded Linux product, this mechanism provides a cheap, automated tripwire: any unexpected license text change anywhere in the dependency tree breaks the build and forces a human to look at exactly what changed and explicitly re-approve it (by updating the checksum after review), rather than that change silently flowing through into a shipped product.

---

**Q6: A recipe builds successfully in isolation but the resulting binary crashes at runtime with a missing shared library error on the target. What's the most likely cause and how do you fix it?**

**A:** This is the classic missing-RDEPENDS bug. The recipe's binary was correctly compiled and linked against some shared library (`DEPENDS` correctly pulled in that library's headers/`.so` for linking at build time), but the corresponding RUNTIME package dependency was never properly declared or auto-detected, so when this package is installed into a minimal target image that doesn't happen to ALSO include that other library's package, the `.so` simply isn't present on the target's filesystem at runtime, and the dynamic linker fails with a "cannot open shared object file" error. In the vast majority of cases this is actually handled automatically — BitBake's packaging step runs `objdump`/`readelf` against every compiled binary in the package, extracts its `DT_NEEDED` entries (the list of shared libraries it dynamically links against), and maps each one back to the package that provides that `.so` file (via a shared library name → package mapping database called `shlibs`), auto-populating `RDEPENDS` accordingly — so this failure mode usually only happens for non-standard cases: a `dlopen()`'d plugin library (loaded at runtime by name, not linked at build time, so objdump can't see the dependency at all) or a shell/Python script invoking another program by name rather than linking against it. The fix is to manually add the missing `RDEPENDS:${PN} += "the-other-package"` for these cases that the automatic shlibs detection mechanism cannot see.

---

**Q7: What's the practical difference between a `-native` recipe variant and a normal target recipe, and why might one recipe need both?**

**A:** A normal recipe's output is built to run on the embedded TARGET device — it's cross-compiled using the target architecture's compiler/toolchain and linked against the target sysroot's libraries. A `-native` variant of the same recipe (created either as a separate recipe inheriting `native.bbclass`, or automatically via `BBCLASSEXTEND = "native"` on the original recipe) is instead compiled to run on the BUILD HOST machine itself, using the build host's own native compiler. A common reason a single piece of software needs both: a code-generation or templating tool whose target-architecture binary the embedded device actually uses at runtime, but where OTHER recipes during the build process also need to invoke that exact same tool (to generate source files, process configuration, etc.) as part of THEIR OWN `do_compile`/`do_configure` step — and since BitBake's build host generally cannot execute target-architecture binaries directly (without QEMU emulation overhead), those other recipes need a native-compiled version of the tool, which runs natively and fast on the build host during the build, completely separate from the target-architecture version that ultimately ships on the device. `quilt-native`, `python3-native`, and many code-generator-style recipes follow exactly this pattern.
