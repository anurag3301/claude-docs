# BitBake — Complete Reference Guide

---

## Table of Contents

1. [Introduction to BitBake](#1-introduction-to-bitbake)
2. [BitBake Architecture Overview](#2-bitbake-architecture-overview)
3. [Metadata File Types](#3-metadata-file-types)
4. [BitBake Syntax — Variables](#4-bitbake-syntax--variables)
5. [BitBake Syntax — Variable Operators](#5-bitbake-syntax--variable-operators)
6. [BitBake Syntax — Functions and Tasks](#6-bitbake-syntax--functions-and-tasks)
7. [Task Dependencies and Execution Order](#7-task-dependencies-and-execution-order)
8. [Classes (.bbclass)](#8-classes-bbclass)
9. [Configuration Files (.conf)](#9-configuration-files-conf)
10. [Include Files and Inheritance](#10-include-files-and-inheritance)
11. [Fetchers — SRC_URI Mechanisms](#11-fetchers--src_uri-mechanisms)
12. [Variable Flags (Varflags)](#12-variable-flags-varflags)
13. [Python Functions in BitBake](#13-python-functions-in-bitbake)
14. [Events and Event Handlers](#14-events-and-event-handlers)
15. [The BitBake Cache and Parsing](#15-the-bitbake-cache-and-parsing)
16. [Provider and RProvider Mechanism](#16-provider-and-rprovider-mechanism)
17. [BitBake Command-Line Interface](#17-bitbake-command-line-interface)
18. [BitBake Server and Hash Equivalence](#18-bitbake-server-and-hash-equivalence)
19. [Shared State Cache (sstate)](#19-shared-state-cache-sstate)
20. [Common Pitfalls and Debugging](#20-common-pitfalls-and-debugging)
21. [Interview Questions and Answers](#21-interview-questions-and-answers)

---

## 1. Introduction to BitBake

**BitBake** is a task-execution engine and build-automation tool originally forked from Portage (Gentoo Linux's package manager) in 2004. It is the core engine that powers the **Yocto Project**, OpenEmbedded, and any embedded Linux build system based on them. BitBake itself knows nothing about "Linux distributions" — it is a generic, metadata-driven task scheduler. All the Linux-specific behavior comes from the metadata (recipes, classes, configuration) supplied by layers like **OpenEmbedded-Core (oe-core)**, **meta-yocto**, and BSP layers.

**What BitBake actually does:**

1. Parses metadata files (`.bb`, `.bbappend`, `.bbclass`, `.conf`, `.inc`)
2. Builds a dependency graph between **tasks** (not just packages)
3. Schedules and executes tasks in the correct order, in parallel where possible
4. Fetches source code from various locations (git, http, local files, etc.)
5. Tracks task output via checksums to avoid re-doing unchanged work
6. Produces packages (.rpm/.deb/.ipk) and root filesystem images

**Why BitBake instead of Make or other build tools:**

- Make builds **files**; BitBake builds **packages from recipes**, each recipe having many discrete tasks (fetch, unpack, patch, configure, compile, install, package)
- BitBake handles **cross-compilation** as a first-class concept (build vs host vs target architectures)
- BitBake has a **shared state cache** that can skip re-building unchanged components even across clean builds or different machines
- BitBake supports **layers** — metadata can be modularly combined, overridden, and extended without modifying upstream recipes

---

## 2. BitBake Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                      bitbake (CLI)                            │
│            Parses commandline, talks to bitbake-server        │
└───────────────────────────┬────────────────────────────────-──┘
                            │ XML-RPC / UNIX socket
┌───────────────────────────▼──────────────────────────────────┐
│                    bitbake-server                              │
│  ┌────────────┐  ┌─────────────┐  ┌──────────────────────┐  │
│  │   Cooker    │  │   Parser    │  │  Task Scheduler        │  │
│  │ (orchestr.) │  │ (.bb/.conf) │  │  (dependency resolver) │  │
│  └────────────┘  └─────────────┘  └──────────────────────┘  │
│         │                │                    │              │
│  ┌──────▼────────────────▼────────────────────▼───────────┐ │
│  │              RunQueue (task execution engine)            │ │
│  │   Spawns worker processes for each scheduled task        │ │
│  └────────────────────────┬───────────────────────────────┘ │
└───────────────────────────┼──────────────────────────────────┘
                            │ fork/exec
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │bitbake-worker│ │bitbake-worker│ │bitbake-worker│  (parallel tasks)
     │ runs do_fetch│ │ runs do_compile│ │runs do_package│
     └────────────┘  └────────────┘  └────────────┘
```

### Key Processes

| Process            | Role                                                          |
|---------------------|----------------------------------------------------------------|
| `bitbake` (client)  | Command-line front-end; sends commands to the server          |
| `bitbake-server`    | Long-lived background process; holds parsed metadata, datastore|
| Cooker               | Core orchestration object inside the server                   |
| Parser               | Reads and evaluates `.bb`/`.bbclass`/`.conf` files into a datastore|
| RunQueue             | Resolves task dependency graph, schedules execution            |
| `bitbake-worker`    | Forked child process that actually executes one task           |

BitBake's client-server architecture (introduced in modern versions) means the **first invocation** of `bitbake <target>` starts a persistent server that keeps the parsed metadata cached in memory — subsequent invocations (e.g., in the same terminal session) reuse this server and skip re-parsing if nothing changed, dramatically speeding up incremental builds.

---

## 3. Metadata File Types

| Extension    | Name                  | Purpose                                                        |
|--------------|------------------------|------------------------------------------------------------------|
| `.bb`        | Recipe                | Describes how to fetch, build, and package one piece of software|
| `.bbappend`  | Append/Bbappend       | Extends or modifies an existing `.bb` recipe without editing it |
| `.bbclass`   | Class                 | Reusable logic/functions/variables inherited by recipes          |
| `.conf`      | Configuration          | Global variable settings (machine, distro, layer, build config) |
| `.inc`       | Include                | Shared snippets included by recipes via `require`/`include`     |

### File Naming Convention

```
<package-name>_<version>.bb

Examples:
  busybox_1.36.1.bb
  linux-yocto_6.6.bb
  myapp_git.bb           ← version "git" means version determined by SRC_URI (git fetcher)
  zlib_1.3.1.bb
```

### Directory Layout (typical layer)

```
meta-mylayer/
├── conf/
│   ├── layer.conf              ← defines this as a layer
│   └── machine/                ← MACHINE configuration files (BSP layers)
├── classes/
│   └── myclass.bbclass
├── recipes-core/
│   └── busybox/
│       ├── busybox_1.36.1.bbappend
│       └── files/
│           └── my-patch.patch
├── recipes-myapp/
│   └── myapp/
│       ├── myapp_1.0.bb
│       └── files/
│           └── myapp-1.0.tar.gz
└── recipes-kernel/
    └── linux/
        └── linux-yocto_%.bbappend
```

---

## 4. BitBake Syntax — Variables

BitBake's metadata language is its own DSL (domain-specific language), syntactically similar to shell and Python combined.

### Variable Assignment

```bitbake
# Simple assignment (immediate — RHS evaluated at parse time if no references,
# but variable expansion of ${VAR} happens lazily at use-time in BitBake)
VAR = "value"

# Append with space separator
VAR += "more"          # VAR becomes "value more"

# Prepend with space separator
VAR =+ "more"           # VAR becomes "more value"

# Append without separator
VAR .= "more"           # VAR becomes "valuemore"

# Prepend without separator
VAR =. "more"           # VAR becomes "morevalue"

# Conditional/weak default — only set if VAR is not already set
VAR ?= "default"

# Weak default at parse time (lowest priority — set if nothing else set it,
# even later in the same file)
VAR ??= "weakest_default"

# Immediate expansion (expand ${OTHERVAR} NOW, not later)
VAR := "${OTHERVAR}"

# Remove a value from a space-separated list variable
VAR_remove = "unwanted_item"
```

### Variable Expansion

```bitbake
A = "hello"
B = "${A} world"     # B expands lazily to "hello world" when referenced

# Order of assignment doesn't matter for lazy (deferred) variables:
C = "${D}"
D = "value"
# C correctly evaluates to "value" because BitBake expands at USE time,
# not at PARSE time, unless ":=" is used.
```

### Variable Flags

```bitbake
# Flags attach metadata to a variable, accessed with [flagname]
do_compile[depends] = "virtual/kernel:do_deploy"
SRC_URI[md5sum]      = "abcdef1234567890"
SRC_URI[sha256sum]   = "0123456789abcdef..."
do_fetch[noexec]     = "1"
```

### Overrides

Overrides let a variable have different values depending on context (machine, distro, target, class):

```bitbake
# Base value
VAR = "default"

# Override for a specific machine
VAR:qemux86-64 = "x86-specific value"

# Override for a specific distro feature
VAR:append = " appended-always"

# Combined override with append
SRC_URI:append:qemux86-64 = " file://x86-only-patch.patch"

# OVERRIDES variable controls which overrides are active
# Typically includes: machine name, distro name, class overrides like "class-target"
```

> **Note:** Yocto switched override syntax from `VAR_override` (underscore, pre-3.4/Honister) to `VAR:override` (colon, 3.4+/Honister onward). Modern recipes use the colon syntax exclusively.

---

## 5. BitBake Syntax — Variable Operators

### Complete Operator Reference

| Operator | Name              | Behavior                                                |
|----------|-------------------|----------------------------------------------------------|
| `=`      | Set                | Assigns value (lazily expanded at use time)              |
| `:=`     | Immediate set      | Assigns value, expanding `${}` references immediately    |
| `?=`     | Default set        | Assigns only if variable is currently unset               |
| `??=`    | Weak default set   | Assigns only if unset, with lowest priority among defaults (evaluated at the very end of parsing) |
| `+=`     | Append (spaced)    | Appends value with a space separator                      |
| `=+`     | Prepend (spaced)   | Prepends value with a space separator                     |
| `.=`     | Append (no space)  | Appends value directly, no separator                      |
| `=.`     | Prepend (no space) | Prepends value directly, no separator                     |
| `:append`| Override-style append | Appends, applied after all `+=` operators, order-independent of file parse order |
| `:prepend`| Override-style prepend | Prepends, same ordering guarantee as `:append`     |
| `:remove`| Override-style remove | Removes matching whitespace-separated items from a list |

### Why `:append`/`:prepend`/`:remove` Differ from `+=`/`=+`

```bitbake
# file A.bb:
VAR = "1"
VAR += "2"

# file A.bbappend:
VAR += "3"

# Result depends on PARSE ORDER for += (file A.bb parsed, then bbappend parsed)
# VAR = "1 2 3"   ← order-dependent, can be fragile

# Using :append instead:
# file A.bb:
VAR = "1"
VAR:append = " 2"

# file A.bbappend:
VAR:append = " 3"

# :append is ALWAYS applied after the base value is fully set, regardless of
# parse order, making bbappend files predictable. This is why bbappend files
# should almost always use :append / :prepend / :remove instead of += / =+.
```

### Example: Removing an Item from a List

```bitbake
DISTRO_FEATURES = "acl alsa bluetooth ext2 ipv4 ipv6 largefile pcmcia usbgadget"
DISTRO_FEATURES:remove = "bluetooth pcmcia"
# Result: "acl alsa ext2 ipv4 ipv6 largefile usbgadget"
```

---

## 6. BitBake Syntax — Functions and Tasks

### Shell Functions (default)

```bitbake
do_mytask() {
    echo "Running in ${PN} version ${PV}"
    install -d ${D}${bindir}
    install -m 0755 ${S}/myapp ${D}${bindir}/
}
```

By default, functions defined with `name() { ... }` are **shell functions** — executed via `/bin/sh`. They run in a generated shell script with all BitBake variables exported as shell variables.

### Python Functions

```bitbake
python do_mytask() {
    import os
    bb.plain("Building %s version %s" % (d.getVar('PN'), d.getVar('PV')))
    if not os.path.exists(d.getVar('S')):
        bb.fatal("Source directory does not exist!")
}
```

Python functions receive the **datastore** `d` as an implicit parameter, giving full programmatic access to all variables (`d.getVar()`, `d.setVar()`, `d.appendVar()`, etc.)

### Declaring a Function as a Task

```bitbake
do_mytask() {
    echo "I am a task"
}

# Make it run as part of the build by adding it to the task list
addtask mytask after do_configure before do_compile
```

### Common Task Reference (oe-core standard tasks)

| Task            | Purpose                                                       |
|-----------------|------------------------------------------------------------------|
| `do_fetch`      | Download source (via SRC_URI fetchers)                          |
| `do_unpack`     | Extract tarballs / check out repos into `${WORKDIR}`             |
| `do_patch`      | Apply patches listed in SRC_URI                                  |
| `do_prepare_recipe_sysroot` | Populate sysroot with dependencies before configure   |
| `do_configure`  | Run the package's configure step (e.g., `./configure`, `cmake`)  |
| `do_compile`    | Run the build (e.g., `make`)                                     |
| `do_install`    | Install build output into `${D}` (the destination/staging dir)   |
| `do_package`    | Split `${D}` into individual binary packages                     |
| `do_package_write_rpm`/`_deb`/`_ipk` | Generate package files in the chosen format        |
| `do_populate_sysroot` | Stage headers/libraries for other recipes to use as deps   |
| `do_deploy`     | Copy final output (kernel images, bootloaders) to deploy directory|
| `do_rootfs`     | (image recipes) Assemble the final root filesystem               |
| `do_image`      | (image recipes) Convert rootfs into final image format (ext4, etc.)|
| `do_build`      | The default, top-level meta-task — depends on everything else    |
| `do_clean`      | Remove build output for this recipe                              |
| `do_cleansstate`| Remove build output AND shared-state cache for this recipe       |
| `do_devshell`   | Drop into an interactive shell with the build environment set up |
| `do_listtasks`  | List all tasks defined for this recipe                           |

### Task Function Anatomy

```bitbake
# Tasks can be overridden, extended, or have pre/post hooks

# Prepend code to run BEFORE the existing do_compile
do_compile:prepend() {
    echo "About to compile..."
}

# Append code to run AFTER the existing do_compile
do_compile:append() {
    echo "Compile finished."
}

# Completely override a task (loses inherited behavior unless you call it explicitly)
do_compile() {
    oe_runmake custom_target
}
```

---

## 7. Task Dependencies and Execution Order

### Intra-Recipe Task Ordering

```bitbake
addtask mytask after do_configure before do_compile
```

This declares: `do_mytask` must run after `do_configure` completes, and must complete before `do_compile` starts.

### Inter-Recipe Dependencies

| Variable     | Meaning                                                              |
|--------------|------------------------------------------------------------------------|
| `DEPENDS`    | Build-time dependency — recipe needs another recipe's `do_populate_sysroot` output to build|
| `RDEPENDS`   | Runtime dependency — package needs another package installed at runtime|
| `RRECOMMENDS`| Runtime soft dependency — preferred but not required at runtime       |
| `RCONFLICTS` | Runtime conflict — cannot be installed alongside this package          |
| `RPROVIDES`  | Declares this package also satisfies another package name's runtime requirement |
| `RREPLACES`  | This package replaces (and can supersede) another package              |

```bitbake
DEPENDS = "zlib openssl"
RDEPENDS:${PN} = "bash python3"
```

### Task-Level Dependencies via Flags

```bitbake
# do_compile of THIS recipe depends on do_populate_sysroot of zlib
do_compile[depends] += "zlib:do_populate_sysroot"

# This is automatically generated from DEPENDS for the standard
# do_configure/do_compile tasks, but custom tasks may need explicit deps.
```

### Task Signature and the Dependency Graph

BitBake builds a complete **task graph** (not just a recipe graph). Each task is a node; edges represent "must complete before." The RunQueue topologically sorts this graph and executes tasks in parallel up to `BB_NUMBER_THREADS`, respecting all dependency edges.

```bitbake
# View the dependency graph for a recipe's tasks
bitbake -g myrecipe
# Generates: task-depends.dot, pn-buildlist, pn-depends.dot

# Visualize with graphviz
dot -Tpng task-depends.dot -o task-depends.png
```

---

## 8. Classes (.bbclass)

Classes contain reusable BitBake code — variables, functions, and task definitions — that recipes pull in via `inherit`.

### Inheriting a Class

```bitbake
inherit autotools pkgconfig

# Multiple classes, space separated
inherit cmake systemd
```

### Important Built-In Classes (oe-core)

| Class               | Purpose                                                            |
|---------------------|----------------------------------------------------------------------|
| `base.bbclass`      | Implicitly inherited by ALL recipes — defines core tasks and behavior|
| `autotools.bbclass` | Handles GNU Autotools (`./configure && make && make install`) packages|
| `cmake.bbclass`     | Handles CMake-based packages                                          |
| `meson.bbclass`     | Handles Meson-based packages                                          |
| `pkgconfig.bbclass` | Adds pkg-config dependency handling                                   |
| `kernel.bbclass`    | Provides kernel-specific build logic                                  |
| `kernel-yocto.bbclass`| Yocto-specific kernel tooling (kernel "features", config fragments)|
| `image.bbclass`     | Core logic for building root filesystem images                       |
| `rootfs-*.bbclass`  | Package-manager-specific rootfs construction (rpm/deb/ipk)            |
| `systemd.bbclass`   | Handles systemd service file installation and enabling                |
| `update-rc.d.bbclass`| Handles SysV init script installation                                |
| `useradd.bbclass`   | Handles creation of users/groups during package install               |
| `allarch.bbclass`   | Marks a recipe as architecture-independent (no per-arch packages)     |
| `native.bbclass`    | Builds a recipe to run on the BUILD host (not the target)             |
| `nativesdk.bbclass` | Builds a recipe to run on the SDK host (relocatable, for the eSDK)    |
| `cross.bbclass`     | Builds cross-compilation toolchain components                         |
| `crosssdk.bbclass`  | Builds the cross-toolchain shipped inside the SDK                     |
| `devshell.bbclass`  | Implements `do_devshell` (interactive debug shell)                    |
| `sanity.bbclass`    | Performs build environment sanity checks                              |
| `license.bbclass`   | Collects and packages license information                             |
| `package.bbclass`   | Core packaging logic — splits `${D}` into output packages             |
| `staging.bbclass`   | Handles sysroot population (`do_populate_sysroot`)                    |
| `insane.bbclass`    | QA checks (the `do_package_qa` task — checks for common packaging mistakes)|
| `bin_package.bbclass`| For recipes that just install pre-built binaries (no compile step)   |
| `external-toolchain.bbclass`| Use a pre-built external toolchain instead of building one     |

### Example Custom Class

```bitbake
# meta-mylayer/classes/myinfo.bbclass

MYINFO_MESSAGE ?= "Built by MyLayer"

python myinfo_print() {
    bb.plain(d.getVar('MYINFO_MESSAGE'))
}

do_compile[postfuncs] += "myinfo_print"
```

```bitbake
# In a recipe:
inherit myinfo
MYINFO_MESSAGE = "Custom build message"
```

### `inherit` vs `inherit_defer`

```bitbake
# Normal inherit — class is processed immediately during parsing
inherit autotools

# Deferred inherit (newer BitBake) — useful to avoid ordering issues
# when a class needs to see final variable values
inherit_defer somelayerclass
```

---

## 9. Configuration Files (.conf)

Configuration files set global build parameters. They are parsed before any recipes and establish the environment all recipes build within.

### Configuration File Hierarchy (parse order)

```
bitbake.conf (from oe-core, the base of everything)
   → bblayers.conf (lists active layers — read first to find all layer.conf)
       → each layer's conf/layer.conf
   → local.conf (user/build-specific settings — build/conf/local.conf)
   → conf/machine/${MACHINE}.conf (target hardware configuration)
   → conf/distro/${DISTRO}.conf (distribution policy configuration)
```

### local.conf — Build-Specific Settings

```bitbake
# build/conf/local.conf

MACHINE ?= "qemux86-64"
DISTRO ?= "poky"

# Parallelism
BB_NUMBER_THREADS = "8"
PARALLEL_MAKE = "-j 8"

# Package format
PACKAGE_CLASSES ?= "package_rpm"

# Disk space monitoring (abort build if low on space)
BB_DISKMON_DIRS = "STOPTASKS,${TMPDIR},1G,100K STOPTASKS,${DL_DIR},1G,100K \
                    ABORT,${TMPDIR},100M,1K ABORT,${DL_DIR},100M,1K"

# Shared download directory (across multiple builds)
DL_DIR ?= "${TOPDIR}/downloads"

# Shared state cache directory
SSTATE_DIR ?= "${TOPDIR}/sstate-cache"

# Extra image features
EXTRA_IMAGE_FEATURES = "debug-tweaks tools-debug"

# Add extra packages to all images
IMAGE_INSTALL:append = " htop nano"
```

### bblayers.conf — Active Layer List

```bitbake
# build/conf/bblayers.conf

POKY_BBLAYERS_CONF_VERSION = "2"

BBPATH = "${TOPDIR}"
BBFILES ?= ""

BBLAYERS ?= " \
  /home/user/poky/meta \
  /home/user/poky/meta-poky \
  /home/user/poky/meta-yocto-bsp \
  /home/user/meta-mylayer \
  "
```

### layer.conf — Per-Layer Definition

```bitbake
# meta-mylayer/conf/layer.conf

BBPATH .= ":${LAYERDIR}"

BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"

BBFILE_COLLECTIONS += "mylayer"
BBFILE_PATTERN_mylayer = "^${LAYERDIR}/"
BBFILE_PRIORITY_mylayer = "6"

LAYERDEPENDS_mylayer = "core"
LAYERSERIES_COMPAT_mylayer = "scarthgap styhead"
```

| Variable                  | Meaning                                                       |
|----------------------------|-----------------------------------------------------------------|
| `BBFILES`                 | Glob patterns telling BitBake where to find `.bb`/`.bbappend` files|
| `BBFILE_COLLECTIONS`      | Registers this layer's symbolic name                            |
| `BBFILE_PATTERN_<name>`   | Regex matching this layer's path (used to map files to collection)|
| `BBFILE_PRIORITY_<name>`  | Priority for resolving conflicts (higher overrides lower)        |
| `LAYERDEPENDS_<name>`     | Other layers this layer requires                                  |
| `LAYERSERIES_COMPAT_<name>`| Declares which Yocto release codenames this layer supports     |

---

## 10. Include Files and Inheritance

### `require` vs `include`

```bitbake
# require: fails the parse if the file is not found
require common.inc

# include: silently continues if the file is not found
include optional-settings.inc
```

### Recipe Versioning with `.inc` Files

A common pattern: shared logic in a `.inc` file, version-specific `.bb` files include it:

```bitbake
# myapp.inc
DESCRIPTION = "My Application"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://COPYING;md5=xxxxx"
SRC_URI = "git://github.com/example/myapp.git;branch=main;protocol=https"
S = "${WORKDIR}/git"
inherit cmake

# myapp_1.0.bb
require myapp.inc
SRCREV = "abc123..."
PV = "1.0"

# myapp_2.0.bb
require myapp.inc
SRCREV = "def456..."
PV = "2.0"
```

### `.bbappend` Mechanics

A `.bbappend` extends an existing `.bb` recipe **without modifying the original file**. This is the cornerstone of layer-based customization.

```bitbake
# meta-mylayer/recipes-core/busybox/busybox_%.bbappend

# % is a wildcard matching ANY version of busybox_<version>.bb

FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

SRC_URI += "file://my-custom.cfg"

do_install:append() {
    install -d ${D}${sysconfdir}
    install -m 0644 ${WORKDIR}/my-custom.cfg ${D}${sysconfdir}/
}
```

**Critical rule:** the `.bbappend` filename must match the `.bb` recipe name (including version, or a `%` wildcard) or BitBake will warn "bbappend has no matching recipe" and silently skip it.

```bitbake
busybox_1.36.1.bb           ←matches→ busybox_1.36.1.bbappend
busybox_1.36.1.bb           ←matches→ busybox_%.bbappend
linux-yocto_6.6.bb          ←matches→ linux-yocto_%.bbappend
```

### FILESEXTRAPATHS and FILESPATH

```bitbake
# FILESPATH tells BitBake where to search for files referenced in SRC_URI
# (file://xxx). Default search includes ${PN}-${PV}, ${PN}, "files" subdirs.

# FILESEXTRAPATHS adds additional search directories (commonly used in bbappends
# to point at the appending layer's own files/ subdirectory)
FILESEXTRAPATHS:prepend := "${THISDIR}/files:"

# := (immediate expansion) is critical here — THISDIR must be evaluated
# at this point in the file, not later, since THISDIR changes per-file.
```

---

## 11. Fetchers — SRC_URI Mechanisms

`SRC_URI` declares where to download source code from. BitBake supports many fetcher "protocols":

### Common Fetcher Types

```bitbake
# Local file (relative to FILESPATH, e.g., a layer's files/ directory)
SRC_URI = "file://mypatch.patch"

# HTTP/HTTPS download
SRC_URI = "https://example.com/source-${PV}.tar.gz"
SRC_URI[sha256sum] = "abcdef0123456789..."

# Git repository
SRC_URI = "git://github.com/example/repo.git;protocol=https;branch=main"
SRCREV = "a1b2c3d4e5f6..."     # Required: exact commit hash to pin

# Git with a tag
SRC_URI = "git://github.com/example/repo.git;protocol=https;tag=v1.2.3"

# Subversion
SRC_URI = "svn://svn.example.com/repo;module=trunk;protocol=https"

# CVS (legacy)
SRC_URI = "cvs://anoncvs@example.com/cvsroot;module=mymodule"

# Local relative path on disk (absolute outside the layer — for local dev)
SRC_URI = "file:///home/user/local-source"

# npm packages
SRC_URI = "npm://registry.npmjs.org;package=mypackage;version=1.0.0"

# Multiple sources combined (tarball + patches)
SRC_URI = "https://example.com/app-${PV}.tar.gz \
           file://0001-fix-build.patch \
           file://0002-add-feature.patch \
           "
```

### Checksums (mandatory for non-git fetches)

```bitbake
SRC_URI[md5sum]    = "d41d8cd98f00b204e9800998ecf8427e"
SRC_URI[sha256sum] = "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855"

# Named checksum (when SRC_URI has multiple files)
SRC_URI = "https://example.com/file1.tar.gz;name=file1 \
           https://example.com/file2.tar.gz;name=file2"
SRC_URI[file1.sha256sum] = "..."
SRC_URI[file2.sha256sum] = "..."
```

BitBake refuses to fetch a remote source without a checksum (unless `BB_STRICT_CHECKSUM` is disabled) — this guarantees build reproducibility and supply-chain integrity.

### Mirrors

```bitbake
# PREMIRRORS — checked BEFORE the original SRC_URI location
PREMIRRORS = "git://.*/.* http://my-mirror.example.com/sources/ \;downloadfilename=${BB_GENERATE_MIRROR_TARBALLS}"

# MIRRORS — checked AFTER the original location fails
MIRRORS += "https://example.com/.* https://backup-mirror.example.com/sources/"
```

### SRCREV — Pinning Git/SCM Revisions

```bitbake
# Pin to an exact commit (required for reproducible builds)
SRCREV = "9f3c2a1b8e7d6f5a4c3b2a1908765432fedcba98"

# Use the tip of a branch (NOT reproducible — for development only)
SRCREV = "${AUTOREV}"

# When AUTOREV is used, PV typically includes a git-derived version string:
PV = "1.0+git${SRCPV}"
```

> **Best practice:** Never ship production builds with `SRCREV = "${AUTOREV}"`. It defeats build reproducibility because the resolved commit can change between builds, silently pulling in new (possibly broken) upstream code.

---

## 12. Variable Flags (Varflags)

Varflags attach metadata to a variable name, distinct from the variable's value:

```bitbake
# Syntax: VARIABLE[flagname] = "value"

# Mark a task as not needing to execute (skip it entirely)
do_fetch[noexec] = "1"

# Declare extra dependency for a task beyond DEPENDS
do_compile[depends] += "otherrecipe:do_populate_sysroot"

# Declare a task depends on a variable's value (forces re-run if VAR changes)
do_compile[vardeps] += "MY_CUSTOM_VAR"

# Mark a task to run on every build regardless of stamps (rarely used)
do_mytask[nostamp] = "1"

# Cache a function/task's checksum dependencies explicitly
do_install[file-checksums] += "${THISDIR}/files/config.txt:True"

# Run a function before/after a given task
do_compile[prefuncs]  += "myprefunc"
do_compile[postfuncs] += "mypostfunc"

# Network access flag (only do_fetch should need network by default)
do_fetch[network] = "1"
```

---

## 13. Python Functions in BitBake

BitBake supports three kinds of Python code:

### Inline Python (anonymous functions, run at parse time)

```bitbake
python () {
    pv = d.getVar('PV')
    if pv.startswith('9999'):
        d.setVar('SRCREV', '${AUTOREV}')
}
```

Anonymous Python functions (no name, just `python () { ... }`) run during **parsing**, immediately after the recipe is parsed — useful for computing derived variables.

### Named Python Tasks

```bitbake
python do_print_info() {
    pn = d.getVar('PN')
    pv = d.getVar('PV')
    bb.plain("Package: %s, Version: %s" % (pn, pv))
}
addtask print_info after do_compile
```

### Python Library Functions (`.bbclass` helper functions)

```bitbake
def get_arch_define(d):
    arch = d.getVar('TARGET_ARCH')
    if arch == 'arm':
        return '-DARM_BUILD'
    return ''

CFLAGS:append = " ${@get_arch_define(d)}"
```

The `${@python_expression}` syntax (called "inline Python expansion") evaluates arbitrary Python and substitutes the result as a string.

### Common `d.*` Datastore API

| Method                       | Purpose                                          |
|-------------------------------|----------------------------------------------------|
| `d.getVar('VAR')`             | Get a variable's expanded value                    |
| `d.getVar('VAR', False)`      | Get unexpanded (raw) value                          |
| `d.setVar('VAR', 'value')`    | Set a variable                                       |
| `d.appendVar('VAR', ' more')` | Append to a variable                                 |
| `d.delVar('VAR')`             | Delete a variable                                    |
| `d.getVarFlag('VAR', 'flag')` | Read a varflag                                       |
| `d.expand('${VAR}')`          | Expand a string containing variable references       |
| `bb.plain(msg)`               | Print message (always shown)                         |
| `bb.note(msg)`                | Print informational message (shown with `-v`)        |
| `bb.warn(msg)`                | Print warning                                         |
| `bb.error(msg)`               | Print error (does not stop the build)                 |
| `bb.fatal(msg)`               | Print error and abort the build immediately            |
| `bb.debug(level, msg)`        | Print debug message at given verbosity level           |

---

## 14. Events and Event Handlers

BitBake fires events during the build process; recipes/classes can register handlers:

```bitbake
addhandler myclass_eventhandler
myclass_eventhandler[eventmask] = "bb.event.BuildStarted bb.event.BuildCompleted"

python myclass_eventhandler() {
    if isinstance(e, bb.event.BuildStarted):
        bb.plain("=== Build starting ===")
    elif isinstance(e, bb.event.BuildCompleted):
        bb.plain("=== Build finished ===")
}
```

### Common Event Types

| Event                          | Fired When                                       |
|----------------------------------|------------------------------------------------------|
| `bb.event.BuildStarted`         | At the very start of a `bitbake` invocation          |
| `bb.event.BuildCompleted`       | When all requested tasks finish                       |
| `bb.build.TaskStarted`          | Before a specific task begins                          |
| `bb.build.TaskSucceeded`        | After a task completes successfully                    |
| `bb.build.TaskFailed`           | After a task fails                                      |
| `bb.event.ConfigParsed`         | After configuration files are fully parsed              |
| `bb.event.RecipeParsed`         | After a single recipe file is parsed                    |
| `bb.event.NoProvider`           | When a dependency cannot be resolved                    |

### Task-Level Event Handlers (simpler syntax)

```bitbake
do_compile[postfuncs] += "notify_compile_done"

python notify_compile_done() {
    bb.plain("Compile step finished for %s" % d.getVar('PN'))
}
```

---

## 15. The BitBake Cache and Parsing

### Parsing Pipeline

```
1. Read bblayers.conf → discover all layers
2. Read each layer's conf/layer.conf → register BBFILES patterns
3. Read bitbake.conf (base configuration, from oe-core)
4. Read conf/machine/${MACHINE}.conf
5. Read conf/distro/${DISTRO}.conf
6. Read local.conf (highest priority user overrides)
7. For each .bb file matching BBFILES:
     a. Parse the recipe (apply require/include, classes via inherit)
     b. Apply any matching .bbappend files
     c. Compute the recipe's full variable set and task list
     d. Cache the parsed result (keyed by file content hash)
```

### Parse Cache (`cache/` directory)

BitBake caches parsed recipe metadata in `<build>/cache/bb_cache.dat` (or similar). On subsequent invocations, if a recipe file's mtime/checksum hasn't changed, BitBake skips re-parsing it — this is why the **second** `bitbake` invocation in a session is much faster to start than the first.

```bash
# Force a full re-parse (e.g., after suspicious caching issues)
bitbake -p          # parse only, populate the cache
rm -rf build/cache  # nuke the cache entirely
```

### PROVIDES, PACKAGES, and the Recipe-to-Package Mapping

```bitbake
# A single recipe can produce MULTIPLE packages
PACKAGES = "${PN} ${PN}-dev ${PN}-dbg ${PN}-doc ${PN}-staticdev ${PN}-locale"

# Default PACKAGES (set by base.bbclass) is usually sufficient; only override
# when you need custom package splitting:
PACKAGES =+ "${PN}-extra-tools"
FILES:${PN}-extra-tools = "${bindir}/extra-tool"
```

---

## 16. Provider and RProvider Mechanism

When multiple recipes can satisfy the same dependency name, BitBake uses **PROVIDES**/**PREFERRED_PROVIDER** to resolve ambiguity:

```bitbake
# A recipe can declare it provides a virtual or alternate name
PROVIDES += "virtual/libgl"

# In machine.conf or distro.conf, choose which recipe satisfies a virtual:
PREFERRED_PROVIDER_virtual/libgl = "mesa"
PREFERRED_PROVIDER_virtual/kernel = "linux-yocto"

# Version pinning when multiple versions of the same recipe exist
PREFERRED_VERSION_linux-yocto = "6.6%"
PREFERRED_VERSION_gcc = "13.%"
```

### Virtual/* Naming Convention

Common "virtual" providers in Yocto allow BSP layers to swap implementations without modifying upstream recipes:

| Virtual Name           | Typically Provided By           |
|--------------------------|----------------------------------|
| `virtual/kernel`        | `linux-yocto`, custom kernel recipe|
| `virtual/bootloader`    | `u-boot`, custom bootloader recipe |
| `virtual/libgl`         | `mesa`                            |
| `virtual/egl`           | `mesa`                            |
| `virtual/xserver`       | `xserver-xorg`                    |

---

## 17. BitBake Command-Line Interface

### Basic Invocation

```bash
# Build a recipe (and everything it depends on) up to do_build
bitbake busybox

# Build multiple targets
bitbake core-image-minimal busybox

# Build only up to a specific task
bitbake -c compile busybox
bitbake -c fetch busybox
bitbake -c devshell busybox

# Force re-execution of a task (ignore stamps)
bitbake -c compile -f busybox

# Clean (remove build output) for a recipe
bitbake -c clean busybox

# Clean including the shared-state cache (deeper clean)
bitbake -c cleansstate busybox

# Show all tasks available for a recipe
bitbake -c listtasks busybox
```

### Useful Flags

| Flag             | Purpose                                                          |
|-------------------|----------------------------------------------------------------|
| `-c <task>`       | Run a specific task instead of the default `do_build`            |
| `-f`              | Force task execution even if up to date                          |
| `-n`              | Dry-run — show what would be built without actually building     |
| `-k`              | Continue building other targets even if one fails                |
| `-v`              | Verbose output                                                    |
| `-D`              | Increase debug verbosity (`-DDD` for more)                       |
| `-e`              | Dump the final expanded environment/variables for a recipe       |
| `-g`              | Generate dependency graphs (.dot files)                          |
| `-u <ui>`         | Select a UI (e.g., `knotty`, `ncurses`, `taskexp`)                |
| `-p`              | Parse all recipes only (populate cache, don't build)              |
| `-s`              | Show recipe versions available                                    |
| `-S <signature>`  | Compare task signatures (useful with sstate debugging)            |

### Inspecting a Recipe's Environment

```bash
# Dump ALL variables and their final values for a recipe
bitbake -e busybox > busybox-env.txt

# Search for a specific variable
bitbake -e busybox | grep ^SRC_URI=
bitbake -e busybox | grep ^WORKDIR=

# Dump the global environment (no recipe — base configuration only)
bitbake -e | less
```

### devshell — Interactive Debug Shell

```bash
bitbake -c devshell busybox

# Drops into a shell with:
#  - Working directory = ${S} (the unpacked source)
#  - PATH includes cross-toolchain
#  - All BitBake environment variables exported
#  - Allows manually running ./configure, make, etc. to debug build failures
```

### Querying Dependencies

```bash
# Show why a package is included (dependency chain)
bitbake-layers show-recipes busybox

# Show recipe dependency tree
bitbake -g core-image-minimal
cat pn-buildlist        # flat list of all recipes that will be built

# Show which layer provides a recipe
bitbake-layers show-recipes -f busybox
```

---

## 18. BitBake Server and Hash Equivalence

### Persistent Server Mode

```bash
# Check server status
bitbake --status

# Explicitly start/stop the server
bitbake --server-only
bitbake -m         # shut down the server

# Force a one-shot run without persisting a server
bitbake --no-setscene busybox
```

### Hash Equivalence (modern BitBake feature)

Hash equivalence allows BitBake to recognize when two different task signatures (e.g., due to a comment change or unrelated metadata edit) produce **functionally identical output**, allowing reuse of cached sstate even when the literal hash differs:

```bitbake
# In local.conf, enable a hash equivalence server (often local, or shared with team)
BB_HASHSERVE = "auto"
BB_SIGNATURE_HANDLER = "OEEquivHash"
```

This dramatically improves build farm efficiency — if a recipe is edited in a way that doesn't change its actual build inputs/outputs (e.g., a recipe comment), downstream recipes don't need to rebuild.

---

## 19. Shared State Cache (sstate)

The shared-state cache (sstate-cache) is BitBake's mechanism to **skip task execution** entirely when a task's inputs are unchanged — even across totally clean builds, different machines on a network, or CI build farms.

### How sstate Works

1. Every task has a **signature** — a hash computed from: the task's function body, all variables it depends on (`vardeps`), and the signatures of all tasks it depends on.
2. Before running a task, BitBake checks if an sstate object with that exact signature already exists (locally in `SSTATE_DIR`, or on a remote `SSTATE_MIRRORS`).
3. If found, BitBake "accelerates" the task — it doesn't run the real task at all, but instead runs a lightweight "setscene" task that just unpacks the cached output (tarball) into the right place.
4. If not found, the real task runs, and its output is THEN stored into `SSTATE_DIR` for future reuse.

```bitbake
# Shared state directory (often shared across multiple build/ directories)
SSTATE_DIR = "${TOPDIR}/sstate-cache"

# Remote sstate mirrors (e.g., a network cache shared by a build team / CI)
SSTATE_MIRRORS = "file://.* http://sstate.example.com/sstate-cache/PATH;downloadfilename=PATH"
```

### Inspecting and Debugging sstate

```bash
# Force ignoring all sstate (rebuild everything from scratch)
bitbake -f core-image-minimal

# See why a particular task didn't reuse sstate (signature mismatch debugging)
bitbake-diffsigs sigdata1 sigdata2

# Explain why two builds produced different results for the same recipe/task
bitbake -S printdiff core-image-minimal
```

### Setscene Tasks

```bitbake
# Tasks ending in _setscene are the "accelerated" version that just
# restores sstate output instead of doing real work
do_compile_setscene
do_populate_sysroot_setscene
do_package_setscene
```

---

## 20. Common Pitfalls and Debugging

### 1. Using `+=` in a `.bbappend` instead of `:append`
`+=` is parse-order dependent. If your bbappend's `+=` runs before the base recipe sets the variable (which can happen due to internal parsing order in some edge cases, or simply makes the result hard to predict), you get unexpected ordering. Always use `:append`/`:prepend` in bbappend files for predictable behavior.

### 2. Forgetting `FILESEXTRAPATHS:prepend := "${THISDIR}/files:"`
Without this in a `.bbappend`, `file://my-patch.patch` in `SRC_URI` won't be found because BitBake only searches the original recipe's layer by default, not the appending layer.

### 3. Missing checksums for non-git SRC_URI
BitBake will halt the fetch with an error demanding `SRC_URI[sha256sum]` for any HTTP/HTTPS/FTP download. Compute it with `sha256sum <file>` after a manual download, or let BitBake suggest it on first failed fetch attempt.

### 4. `SRCREV = "${AUTOREV}"` in production
Defeats reproducibility — the resolved git commit can silently change between builds. Always pin to an exact `SRCREV` hash for release builds.

### 5. bbappend filename mismatch
`mypkg_1.0.bbappend` only matches `mypkg_1.0.bb` exactly. If the recipe gets upgraded to `mypkg_1.1.bb`, the old bbappend silently stops applying with only a warning. Use `mypkg_%.bbappend` (wildcard) to match any version, unless you specifically need version-pinned behavior.

### 6. Stale shared-state cache after recipe logic change
If you change a task's Python logic in a way BitBake's signature computation doesn't detect as a real change (rare, but can happen with certain dynamic Python), sstate might incorrectly reuse stale output. `bitbake -c cleansstate <recipe>` forces a true rebuild.

### 7. Task running with the wrong working directory
Each task's working directory defaults to `${WORKDIR}`, except `do_compile`/`do_configure`/etc. which default to `${S}` via the `B` variable typically being `${S}` or `${WORKDIR}/build` for out-of-tree builds (CMake). Confusion here causes "file not found" errors — always verify with `bitbake -e <recipe> | grep ^WORKDIR=` and `grep ^S=` and `grep ^B=`.

### 8. PARALLEL_MAKE causing flaky builds
Some upstream Makefiles have race conditions when built with `-j N` parallel jobs. Symptom: build fails intermittently, succeeds on retry. Fix per-recipe:
```bitbake
PARALLEL_MAKE = ""    # disable parallel make for this recipe only
```

---

## 21. Interview Questions and Answers

**Q1: What is the difference between BitBake and the Yocto Project?**

**A:** BitBake is a generic task-execution engine — a build tool that parses metadata (recipes, classes, configuration files) and executes tasks in dependency order. It has no inherent knowledge of "Linux" or "embedded systems." The Yocto Project is a much larger collaborative project that provides the actual Linux-specific metadata: OpenEmbedded-Core (the base recipe collection for core utilities, toolchains, and libraries), Poky (a reference distribution combining BitBake + oe-core + meta-yocto), and a vast ecosystem of BSP (Board Support Package) layers and software layers. In short: BitBake is the engine; Yocto/OpenEmbedded is the car built around that engine. You could theoretically use BitBake to build something entirely unrelated to Linux, since it's metadata-driven and generic.

---

**Q2: Explain the difference between DEPENDS and RDEPENDS.**

**A:** DEPENDS is a build-time dependency — it tells BitBake that this recipe needs another recipe's `do_populate_sysroot` output (headers, libraries, pkg-config files) available in the sysroot before this recipe's `do_configure`/`do_compile` can run. For example, a recipe that links against OpenSSL needs `DEPENDS = "openssl"` so the OpenSSL headers and `.so` development files are staged before compiling. RDEPENDS is a runtime dependency — it tells the package manager that the resulting binary package needs another package **installed on the target device** for the program to actually run (e.g., a shared library it dynamically links against, or an interpreter like Python). DEPENDS affects the build sysroot; RDEPENDS affects what gets pulled into the final root filesystem image and what's listed in the package's runtime dependency metadata (RPM/deb/ipk "Requires" field).

---

**Q3: What is the shared-state cache and why is it critical for build performance?**

**A:** The shared-state cache (sstate) lets BitBake skip re-executing a task entirely if it can prove the task's output would be identical to a previously cached result. BitBake computes a cryptographic signature for every task based on the function body, all variables the task depends on, and the signatures of upstream tasks it depends on. Before running a task, it checks if a matching sstate object exists (locally or on a network mirror). If found, instead of running the real (often slow) task, it runs a lightweight "setscene" variant that just unpacks a pre-built tarball into the right location. This means: a clean `rm -rf tmp/` followed by a rebuild can complete in minutes instead of hours, because almost everything is restored from sstate rather than recompiled. It's also why CI build farms can share a network sstate mirror — if one machine already built `gcc-cross` for a given configuration, every other machine and even other developers can reuse that exact output instead of rebuilding GCC themselves.

---

**Q4: Why does BitBake require checksums for SRC_URI fetches but not for git?**

**A:** For tarball/HTTP-based fetches, the only way to guarantee you got the exact bytes you expect (and not a tampered, corrupted, or differently-versioned file) is a cryptographic hash of the downloaded content — hence `SRC_URI[sha256sum]` and `SRC_URI[md5sum]`. Git, however, is inherently content-addressed: the `SRCREV` (commit hash) IS the checksum — a git commit hash is computed from the tree contents, so pinning to a specific `SRCREV` already guarantees byte-for-byte reproducibility of that commit's tree. There is no separate checksum needed because the integrity check is baked into git's object model itself. This is also why `SRCREV = "${AUTOREV}"` is dangerous for reproducibility — it deliberately discards this guarantee by always resolving to "whatever the branch tip currently is," which can change between builds.

---

**Q5: Explain BitBake's variable expansion timing — when does `${VAR}` actually get substituted?**

**A:** BitBake uses **lazy (deferred) expansion** by default for the `=` operator. When you write `B = "${A} world"`, BitBake stores the literal string `"${A} world"` — it does NOT immediately look up A's value. The actual substitution happens only when the variable is **read** (e.g., when a task script is generated, or when `d.getVar('B')` is called). This means variable assignment order in a file usually doesn't matter for plain `=` — you can reference a variable before it's defined later in the same file, and it will still resolve correctly at use-time. The `:=` operator is the exception: it forces **immediate expansion** at the point of assignment, capturing whatever A's value is at that moment. This distinction matters most when a variable like `THISDIR` changes meaning depending on which file is currently being parsed — `FILESEXTRAPATHS:prepend := "${THISDIR}/files:"` must use `:=` precisely because `THISDIR` needs to be captured as "the directory of THIS bbappend file," not lazily re-evaluated later when it might mean something else.

---

**Q6: What happens during BitBake's `do_populate_sysroot` task and why does it matter for cross-compilation?**

**A:** `do_populate_sysroot` takes the relevant build output of a recipe (typically headers, libraries, pkg-config `.pc` files, and CMake config files from `do_install`'s output) and stages them into a shared **sysroot** directory tree that other recipes' build processes can use as if they were installed system-wide. This is the mechanism that makes cross-compilation dependencies work: when recipe B has `DEPENDS = "recipeA"`, BitBake ensures `recipeA:do_populate_sysroot` completes before `recipeB:do_configure` runs, and recipeB's compiler/linker search paths are pointed at the shared sysroot (`STAGING_DIR_HOST`/`STAGING_DIR_TARGET`) where recipeA's headers and libraries now live. Without this staged sysroot mechanism, every recipe would need direct, manually-managed file paths to every dependency's build directory — sysroot population is what allows BitBake's DEPENDS graph to translate into a working, automatically-assembled compiler environment for cross-compilation.

---

**Q7: What's the difference between `bitbake -c clean` and `bitbake -c cleansstate`?**

**A:** `bitbake -c clean <recipe>` removes the recipe's local build output — the working directory contents (`${WORKDIR}`), stamps, and any generated packages — forcing those tasks to re-run on the next build. However, it does NOT touch the shared-state cache. So if an identical sstate object still exists for this recipe's tasks (same signature), the very next build will simply re-accelerate from sstate instead of doing real work — the "clean" was largely cosmetic from a performance perspective. `bitbake -c cleansstate <recipe>` goes further: it removes BOTH the local build output AND the corresponding sstate cache entries for this recipe, forcing genuinely from-scratch execution of every task with no shortcut available. This distinction matters when debugging a suspected sstate corruption issue or when you've made a change that BitBake's signature computation might not have correctly detected (a rare edge case, but `cleansstate` is the reliable "trust nothing, rebuild everything for this recipe" hammer).

---

**Q8: How does BitBake resolve which recipe should provide `virtual/kernel`?**

**A:** Many recipes in OpenEmbedded provide functionally-equivalent implementations of the same role — for example, multiple kernel recipes (`linux-yocto`, `linux-yocto-rt`, a vendor BSP kernel) could all satisfy "the kernel for this machine." Rather than hard-coding a single kernel recipe name everywhere it's referenced (in image recipes, in `DEPENDS`, etc.), OpenEmbedded uses the abstract name `virtual/kernel`. Each candidate kernel recipe declares `PROVIDES += "virtual/kernel"` (often implicitly via `inherit kernel`). The actual choice of which real recipe satisfies `virtual/kernel` for a given build is made via `PREFERRED_PROVIDER_virtual/kernel = "linux-yocto"` set in the machine configuration file (`conf/machine/${MACHINE}.conf`). This indirection is what allows a BSP layer to simply set `PREFERRED_PROVIDER_virtual/kernel` to point at its own custom kernel recipe, and every recipe and image in the system that depends on "the kernel" automatically gets the right one — without any of those dependent recipes needing to know or care about the actual recipe name.
