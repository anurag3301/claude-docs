# Yocto SDK & Extensible SDK (eSDK) — Complete Reference Guide

---

## Table of Contents

1. [Introduction — Why SDKs Exist Separately from the Full Build](#1-introduction--why-sdks-exist-separately-from-the-full-build)
2. [Standard SDK vs Extensible SDK — Key Differences](#2-standard-sdk-vs-extensible-sdk--key-differences)
3. [Generating a Standard SDK](#3-generating-a-standard-sdk)
4. [Standard SDK Contents and Structure](#4-standard-sdk-contents-and-structure)
5. [Using a Standard SDK](#5-using-a-standard-sdk)
6. [Generating an Extensible SDK](#6-generating-an-extensible-sdk)
7. [Extensible SDK Contents and Structure](#7-extensible-sdk-contents-and-structure)
8. [Using the Extensible SDK with devtool](#8-using-the-extensible-sdk-with-devtool)
9. [CMake/Autotools/pkg-config Integration with the SDK](#9-cmakeautotoolspkg-config-integration-with-the-sdk)
10. [SDK Toolchain Variables](#10-sdk-toolchain-variables)
11. [Customizing SDK Contents](#11-customizing-sdk-contents)
12. [SDK Updates — sstate-Based eSDK Refresh](#12-sdk-updates--sstate-based-esdk-refresh)
13. [Container-Based SDK Distribution](#13-container-based-sdk-distribution)
14. [IDE Integration](#14-ide-integration)
15. [SDK Versioning and Compatibility Management](#15-sdk-versioning-and-compatibility-management)
16. [Complete Worked Example — SDK-Based Application Development Workflow](#16-complete-worked-example--sdk-based-application-development-workflow)
17. [Common Pitfalls and Debugging](#17-common-pitfalls-and-debugging)
18. [Interview Questions and Answers](#18-interview-questions-and-answers)

---

## 1. Introduction — Why SDKs Exist Separately from the Full Build

Building a complete embedded Linux image from scratch requires the full Yocto/BitBake build environment: cloning Poky and every needed layer, a multi-hour-capable build machine, tens of gigabytes of disk space, and genuine familiarity with the layer/recipe/BitBake ecosystem. This is entirely appropriate for the engineers actually maintaining the platform/BSP — but it's a completely disproportionate barrier for an APPLICATION developer who simply needs to write, cross-compile, and test ONE application against an already-defined, already-released target platform, without needing (or wanting) to understand or touch the underlying build system at all.

The **SDK** (and its more capable sibling, the **Extensible SDK / eSDK**) solves exactly this problem: `bitbake <image> -c populate_sdk` (or `populate_sdk_ext`) packages everything an application developer needs — a working cross-compiler toolchain, a matching target sysroot (headers and libraries for everything in that specific image), and supporting tooling — into a single, self-contained installer script that runs independently of the full Yocto build environment.

```
Full Yocto Build Environment                    Generated SDK
(BSP engineers, platform team)                  (Application developers)
─────────────────────────────                  ─────────────────────────
- Clone poky + all layers                       - Run one installer script
- Hours-long initial build                       - Minutes to install
- 50GB+ disk space                                - A few GB disk space
- Deep BitBake/recipe knowledge needed             - Just needs to know gcc/cmake basics
- Builds the ENTIRE platform image                  - Builds/cross-compiles ONE application
```

---

## 2. Standard SDK vs Extensible SDK — Key Differences

| Aspect                          | Standard SDK (`populate_sdk`)               | Extensible SDK / eSDK (`populate_sdk_ext`)        |
|------------------------------------|---------------------------------------------|----------------------------------------------------|
| Core contents                     | Cross-toolchain + fixed target sysroot       | Cross-toolchain + sysroot + bundled BitBake + pre-populated sstate-cache + devtool|
| Installer size                    | Smaller (typically hundreds of MB)            | Larger (often several GB, due to bundled sstate)     |
| Adding a NEW recipe/dependency    | Not possible — sysroot is fixed at SDK generation time; must request an updated SDK from the platform team| Possible directly: `devtool add` works inside the eSDK, building genuinely new recipes using the bundled BitBake + cached sstate|
| Modifying an existing package's source | Not directly supported within the SDK itself | Fully supported: `devtool modify` works exactly as it would in a full build environment|
| Updating to a newer platform release | Reinstall a newly-generated SDK from scratch | `devtool sdk-update` can incrementally refresh against a published update server|
| Build tooling included            | Just the cross-compiler/binutils/cmake-toolchain-file/pkg-config — uses your own native build commands directly | Full embedded BitBake installation, enabling genuine recipe-based builds, not just direct compiler invocation|
| Best suited for                    | Simple, single-application cross-compilation against a FIXED, known target API/ABI surface | Iterative application development, adding new dependencies, modifying platform packages, full devtool workflow|

In short: the Standard SDK is a lean, traditional cross-toolchain-plus-sysroot package — exactly what most people picture when they hear "cross-compilation SDK." The Extensible SDK is a much more capable, heavier package that essentially gives an application developer a scoped-down, fast-starting version of the FULL Yocto build capability, specifically themed around one already-built platform image.

---

## 3. Generating a Standard SDK

```bash
# From within a full Yocto build environment, after building the image
# you want an SDK that matches:
bitbake core-image-minimal
bitbake core-image-minimal -c populate_sdk
```

```bash
# Output:
ls tmp/deploy/sdk/
poky-glibc-x86_64-core-image-minimal-cortexa72-toolchain-5.0.sh
```

### What Gets Built

`populate_sdk` triggers (among other things):
- `gcc-cross-canadian` — a compiler that runs on the SDK's HOST architecture (commonly the same x86_64 as the build machine, but could differ) and targets the embedded TARGET architecture
- `nativesdk-*` variants of various supporting tools (cmake, pkg-config, etc.) — built to run on the SDK host
- A target sysroot populated with headers/libraries for every package actually present in the source image (`core-image-minimal` in this example) — meaning the SDK's available API surface exactly matches what's genuinely on the target device, not some larger generic set

---

## 4. Standard SDK Contents and Structure

After running the generated installer:

```bash
./poky-glibc-x86_64-core-image-minimal-cortexa72-toolchain-5.0.sh
# Prompts for installation directory, default: /opt/poky/5.0/
```

```
/opt/poky/5.0/
├── environment-setup-cortexa72-poky-linux        ← source this before building
├── site-config-cortexa72-poky-linux
├── sysroots/
│   ├── x86_64-pokysdk-linux/                       ← native/HOST-architecture tools (the compiler itself runs here)
│   │   └── usr/bin/
│   │       └── cortexa72-poky-linux/
│   │           ├── aarch64-poky-linux-gcc
│   │           ├── aarch64-poky-linux-g++
│   │           └── ...
│   └── cortexa72-poky-linux/                         ← TARGET sysroot (headers/libs the compiler targets)
│       └── usr/
│           ├── include/                                ← every header from every package in the source image
│           └── lib/                                     ← every library from every package in the source image
└── version-cortexa72-poky-linux
```

### environment-setup-* — The Critical Activation Script

```bash
source /opt/poky/5.0/environment-setup-cortexa72-poky-linux
```

This script, which MUST be sourced into your shell before any cross-compilation work, sets:
- `CC`, `CXX`, `LD`, `AR`, `STRIP`, etc. — pointing at the correct cross-toolchain binaries
- `CFLAGS`, `LDFLAGS` — matching the target's architecture/ABI/hardening settings
- `PKG_CONFIG_PATH`/`PKG_CONFIG_SYSROOT_DIR` — so `pkg-config` correctly resolves against the TARGET sysroot rather than the build host's own libraries
- `OECORE_TARGET_SYSROOT`/`OECORE_NATIVE_SYSROOT` — explicit sysroot path references
- A correctly pre-configured CMake toolchain file path (`OECORE_TARGET_SYSROOT/usr/share/cmake/OEToolchainConfig.cmake` or similar) for `cmake`-based projects

---

## 5. Using a Standard SDK

```bash
source /opt/poky/5.0/environment-setup-cortexa72-poky-linux

# Plain Makefile project
make CC=$CC

# Autotools project
./configure --host=aarch64-poky-linux
make
make install DESTDIR=/tmp/staging

# CMake project — the environment script sets up a toolchain file automatically
mkdir build && cd build
cmake .. -DCMAKE_TOOLCHAIN_FILE=$OECORE_NATIVE_SYSROOT/usr/share/cmake/OEToolchainConfig.cmake
make

# Direct gcc invocation
$CC $CFLAGS myapp.c -o myapp $LDFLAGS

# Verify the resulting binary is genuinely cross-compiled for the target
file myapp
# myapp: ELF 64-bit LSB executable, ARM aarch64, ...
```

### Deploying the Resulting Binary to a Target Device

```bash
scp myapp root@192.168.1.50:/usr/bin/
ssh root@192.168.1.50 "/usr/bin/myapp"
```

---

## 6. Generating an Extensible SDK

```bash
bitbake core-image-minimal -c populate_sdk_ext
```

```bash
ls tmp/deploy/sdk/
poky-glibc-x86_64-core-image-minimal-cortexa72-toolchain-ext-5.0.sh
```

The `-ext` suffix in the generated installer filename is the conventional marker distinguishing an Extensible SDK installer from a Standard SDK installer.

### Generation Time and Size Tradeoffs

```bitbake
# By default, populate_sdk_ext bundles a substantial slice of the sstate-cache
# needed to rebuild most of the SDK's own content from source, enabling
# devtool add/modify to work efficiently offline. This makes the eSDK
# installer significantly larger than a Standard SDK. You can trim this:

SDK_INCLUDE_PKGDATA = "1"     # Include package metadata for devtool search
SDK_INCLUDE_TOOLCHAIN = "1"   # Ensure the full toolchain ships even with a minimal sstate set
SDK_EXT_TYPE = "full"         # "full" (default, bundles sstate) vs "minimal" (smaller, requires network access to an sstate mirror for devtool operations)
```

```bitbake
# A "minimal" eSDK trades installer size for requiring network connectivity
# to a published sstate mirror when devtool needs to build something not
# already locally cached:
SDK_EXT_TYPE = "minimal"
SDK_LOCAL_CONF_WHITELIST = "SSTATE_MIRRORS"
```

---

## 7. Extensible SDK Contents and Structure

```bash
./poky-glibc-x86_64-core-image-minimal-cortexa72-toolchain-ext-5.0.sh
# Default install location: /opt/poky/5.0/
```

```
/opt/poky/5.0/
├── environment-setup-cortexa72-poky-linux
├── sysroots/                                   ← same concept as Standard SDK
├── .devtoolbase                                  ← internal bookkeeping
├── conf/
│   ├── local.conf                                  ← a real, functioning BitBake build configuration!
│   └── bblayers.conf
├── layers/                                          ← actual layer copies (meta, meta-poky, etc.) — a real, if scoped-down, Yocto build tree
├── sstate-cache/                                      ← the bundled, pre-populated shared-state cache
└── bitbake -> layers/poky/bitbake/bin/bitbake          ← yes, a REAL bitbake binary, fully functional
```

This is the key structural distinction from a Standard SDK: an Extensible SDK installation is, under the hood, genuinely a real (if intentionally scoped and pre-cached) BitBake build environment — complete with its own `conf/local.conf`, `conf/bblayers.conf`, actual layer copies, and a working `bitbake` binary. `devtool` commands run inside an eSDK are NOT some separate, simplified reimplementation — they invoke this exact same underlying BitBake machinery used in a full build environment, just heavily accelerated by the bundled sstate-cache so that most needed build output is already cached rather than requiring genuine from-scratch compilation.

---

## 8. Using the Extensible SDK with devtool

```bash
source /opt/poky/5.0/environment-setup-cortexa72-poky-linux

# All the standard devtool workflow (full detail in the devtool document)
# works identically inside an eSDK:

devtool add my-sensor-app https://github.com/example/sensor-app.git
devtool build my-sensor-app
devtool deploy-target my-sensor-app root@192.168.1.50

devtool modify busybox
cd $(devtool find-recipe -p busybox 2>/dev/null || echo "workspace/sources/busybox")
# edit source
devtool build busybox
devtool deploy-target busybox root@192.168.1.50

devtool finish my-sensor-app meta-mylayer
```

### Why This Is Powerful — Adding a Genuinely New Dependency

```bash
# An application developer, working purely from an installed eSDK with NO
# access to (or knowledge of) the full Yocto build environment, can add a
# completely new library dependency that wasn't part of the original image:

devtool add my-new-lib https://github.com/example/new-lib.git
devtool build my-new-lib
# my-new-lib is now genuinely built and available in the eSDK's sysroot,
# usable as a DEPENDS for other workspace recipes, exactly as if a platform
# engineer had added it via a full build environment

devtool add my-app /home/dev/my-app-source
# Edit my-app's recipe to add: DEPENDS += "my-new-lib"
devtool build my-app
devtool deploy-target my-app root@192.168.1.50
```

A Standard SDK simply CANNOT do this — its sysroot is a fixed, point-in-time snapshot generated once at `populate_sdk` time; there is no mechanism within a Standard SDK to build and integrate a genuinely new dependency that wasn't already present in the original source image.

---

## 9. CMake/Autotools/pkg-config Integration with the SDK

### CMake Toolchain File

```cmake
# OEToolchainConfig.cmake (auto-generated, referenced via CMAKE_TOOLCHAIN_FILE)
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)
set(CMAKE_C_COMPILER aarch64-poky-linux-gcc)
set(CMAKE_CXX_COMPILER aarch64-poky-linux-g++)
set(CMAKE_FIND_ROOT_PATH ${OECORE_TARGET_SYSROOT})
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

The `CMAKE_FIND_ROOT_PATH_MODE_*` settings are critical — they ensure `find_package()`/`find_library()` calls search ONLY within the target sysroot for libraries/packages (preventing CMake from accidentally linking against the BUILD HOST's own native libraries, a classic, hard-to-diagnose cross-compilation bug if misconfigured), while still allowing `find_program()` to search the normal host PATH for build-time tools like code generators that genuinely need to run on the build host.

### pkg-config Cross-Compilation Setup

```bash
echo $PKG_CONFIG_SYSROOT_DIR
# /opt/poky/5.0/sysroots/cortexa72-poky-linux

echo $PKG_CONFIG_PATH
# /opt/poky/5.0/sysroots/cortexa72-poky-linux/usr/lib/pkgconfig:...

pkg-config --cflags --libs glib-2.0
# Correctly returns TARGET sysroot include/lib paths, not the build host's
# own (likely different-version, different-architecture) glib-2.0
```

---

## 10. SDK Toolchain Variables

| Variable                    | Meaning                                                            |
|---------------------------------|------------------------------------------------------------------|
| `SDKMACHINE`                    | Architecture the SDK ITSELF runs on (the developer's machine)       |
| `SDK_ARCH`                      | Computed SDK host architecture identifier                            |
| `SDKPATH`                        | Default installation path baked into the generated installer script   |
| `SDKEXTPATH`                      | Default installation path specifically for the Extensible SDK         |
| `SDKIMAGE_FEATURES`               | IMAGE_FEATURES-style toggle specifically for what gets included in the SDK's target sysroot (e.g., `dev-pkgs dbg-pkgs`)|
| `TOOLCHAIN_HOST_TASK`              | The set of packages installed into the SDK's HOST-side tooling        |
| `TOOLCHAIN_TARGET_TASK`             | The set of packages whose headers/libraries populate the SDK's TARGET sysroot|
| `TOOLCHAIN_OUTPUTNAME`               | Controls the generated installer script's filename                      |
| `SDK_TITLE`/`SDK_VERSION`/`SDK_VENDOR`| Branding metadata shown in the installer's interactive prompts          |
| `SDK_NAME`                            | Computed full SDK identifier string                                      |
| `SDK_DIR`                              | Build-time staging directory for SDK construction (before final installer packaging)|
| `SDK_EXT_TYPE`                          | "full" vs "minimal" — controls whether sstate is bundled or fetched on demand|
| `SDK_LOCAL_CONF_WHITELIST`               | Which `local.conf` variables are permitted to be customized within an installed eSDK|
| `SDK_UPDATE_URL`                          | Default URL `devtool sdk-update` checks for published updates           |

---

## 11. Customizing SDK Contents

```bitbake
# Extend what's available in the SDK's target sysroot beyond just what's
# in the source image itself — e.g., extra development headers an app
# developer would commonly need that aren't part of the runtime image:
TOOLCHAIN_TARGET_TASK:append = " glib-2.0-dev libcurl-dev"

# Include debug symbols and full headers for everything in the image
# (useful for application developers who need to debug against system libraries)
SDKIMAGE_FEATURES += "dev-pkgs dbg-pkgs"

# Add an entirely custom tool to the SDK's HOST-side toolset
TOOLCHAIN_HOST_TASK:append = " nativesdk-cmake nativesdk-make"

# Customize the installer's branding
SDK_VENDOR = "-mycompanysdk"
SDK_TITLE = "MyProduct Application Development SDK"
SDK_VERSION = "${DISTRO_VERSION}"
```

### Per-Image SDK Customization

Since SDK generation is always tied to a SPECIFIC image recipe's package set, different images naturally produce different SDKs:

```bash
bitbake my-product-image -c populate_sdk
bitbake my-product-image-dev -c populate_sdk_ext
```

A common pattern: generate a Standard SDK from a lean PRODUCTION image (smaller, more restrictive sysroot matching exactly what ships) for final-validation/integration purposes, and a richer Extensible SDK from a DEVELOPMENT image variant (with `dev-pkgs`/`dbg-pkgs` IMAGE_FEATURES) for day-to-day application development.

---

## 12. SDK Updates — sstate-Based eSDK Refresh

```bash
# Inside an already-installed eSDK, after the platform team has published
# a newer build (new sstate-cache snapshot, possibly a new image/recipe set):
devtool sdk-update http://build-server.example.com/sdk-updates/cortexa72/

# Or, if SDK_UPDATE_URL was baked in at generation time:
devtool sdk-update
```

This refreshes the eSDK's bundled sstate-cache and layer metadata to match a newer platform build, WITHOUT requiring the application developer to uninstall and reinstall an entirely new eSDK package from scratch — meaningfully reducing the friction of staying current with platform updates over the lifetime of a project.

### Publishing eSDK Updates (Platform Team Side)

```bitbake
# In local.conf, when generating the eSDK that will later serve as an
# update source:
SDK_UPDATE_URL = "http://build-server.example.com/sdk-updates/cortexa72/"
```

```bash
# The platform team's CI publishes both a full eSDK installer (for fresh
# installs) AND an incremental sstate update feed (for devtool sdk-update)
# to the same well-known internal URL after every relevant platform build.
```

---

## 13. Container-Based SDK Distribution

A common modern alternative/complement to the traditional shell-script installer is packaging the SDK environment inside a container image, sidestepping host-OS compatibility concerns (an installer script built against a specific glibc version on the SDK host can have compatibility issues on a sufficiently different developer machine; a container sidesteps this entirely by bundling its own consistent userspace):

```dockerfile
FROM ubuntu:22.04
COPY poky-glibc-x86_64-core-image-minimal-cortexa72-toolchain-5.0.sh /tmp/
RUN /tmp/poky-glibc-x86_64-core-image-minimal-cortexa72-toolchain-5.0.sh -d /opt/sdk -y
ENV PATH="/opt/sdk/sysroots/x86_64-pokysdk-linux/usr/bin:${PATH}"
ENTRYPOINT ["/bin/bash", "-c", "source /opt/sdk/environment-setup-cortexa72-poky-linux && exec bash"]
```

```bash
docker build -t my-product-sdk:5.0 .
docker run -it -v $(pwd):/workspace my-product-sdk:5.0
# Inside the container: a fully working, consistent cross-compilation
# environment, with the host project source mounted in via -v
```

This approach is increasingly popular for CI pipelines specifically — a containerized SDK gives perfectly reproducible, version-pinned build environments across every CI run and every developer's machine, eliminating "works on my machine" SDK-installation drift entirely.

---

## 14. IDE Integration

### VSCode with the eSDK (via devtool ide-sdk)

```bash
# (Full detail in the devtool document, Section 11)
devtool ide-sdk myapp --target root@192.168.1.50 vscode
# Generates .vscode/launch.json + tasks.json with correctly configured
# cross-gdb path, gdbserver remote connection, and sysroot-aware source
# debugging — directly usable from within an eSDK installation
```

### Eclipse / Qt Creator Cross-Toolchain Configuration

Both Eclipse (historically, via the Yocto Project Eclipse plugin) and Qt Creator support manually pointing their build/debug configuration at an installed SDK's `environment-setup-*` script and cross-gdb binary — letting a full graphical IDE workflow (build, run, breakpoint debugging) work directly against target hardware, using the exact same underlying cross-toolchain the SDK provides.

```
# Qt Creator: Tools → Options → Kits → Add
#   Compiler: /opt/poky/5.0/sysroots/x86_64-pokysdk-linux/usr/bin/.../aarch64-poky-linux-gcc
#   Debugger: /opt/poky/5.0/sysroots/x86_64-pokysdk-linux/usr/bin/.../aarch64-poky-linux-gdb
#   Sysroot:  /opt/poky/5.0/sysroots/cortexa72-poky-linux
```

---

## 15. SDK Versioning and Compatibility Management

```bitbake
# Ensure generated SDK installers carry a clear, traceable version identifier
SDK_VERSION = "${DISTRO_VERSION}-${DATETIME}"
```

### Why SDK/Platform Version Alignment Matters

An SDK's target sysroot is a snapshot of headers/libraries EXACTLY matching the image it was generated from. Using an SDK generated from an OLDER platform release to build an application intended for a NEWER deployed platform release (or vice versa) risks ABI mismatches — a library function signature change, a new required header, a removed deprecated API — that would compile/link successfully against the (stale) SDK sysroot but fail or behave incorrectly when the resulting binary actually runs against the genuinely different library versions present on a differently-versioned real target device.

```bash
# A disciplined practice: tag/version-track SDK releases alongside platform
# image releases, and have application developers explicitly track which
# SDK version their application was last validated against, treating an
# SDK upgrade with the same care as any other dependency version bump.
```

---

## 16. Complete Worked Example — SDK-Based Application Development Workflow

### Platform Team Side (generates and publishes the SDK)

```bash
# In the full Yocto build environment:
bitbake my-product-image
bitbake my-product-image -c populate_sdk_ext

# Publish both the full installer and an incremental update feed:
cp tmp/deploy/sdk/*.sh /internal-artifact-server/sdk-releases/v1.2/
SDK_UPDATE_URL="http://build-server.internal/sdk-updates/my-product/" \
    bitbake my-product-image -c populate_sdk_ext
rsync -a tmp/deploy/sdk-update-feed/ /internal-artifact-server/sdk-updates/my-product/
```

### Application Developer Side (uses the SDK, no full Yocto environment needed)

```bash
# One-time setup:
wget http://internal-artifact-server/sdk-releases/v1.2/my-product-toolchain-ext-1.2.sh
chmod +x my-product-toolchain-ext-1.2.sh
./my-product-toolchain-ext-1.2.sh -d ~/my-product-sdk -y

source ~/my-product-sdk/environment-setup-cortexa72-poky-linux

# Day-to-day development:
devtool add my-gateway-app https://github.com/mycompany/gateway-app.git
devtool build my-gateway-app
devtool deploy-target my-gateway-app root@192.168.1.50

# Add a needed dependency the original platform image didn't include:
devtool add libmodbus https://github.com/stephane/libmodbus.git
devtool build libmodbus
# Edit my-gateway-app's recipe: DEPENDS += "libmodbus"
devtool build my-gateway-app
devtool deploy-target my-gateway-app root@192.168.1.50

# Test, iterate...

# Periodically refresh against the platform team's latest published build:
devtool sdk-update

# Once satisfied, hand off the finished recipe(s) back to the platform team
# for inclusion in the official product layer (the developer can either
# run devtool finish themselves if they have layer write access, or submit
# the generated patches/recipe for the platform team to integrate):
devtool finish my-gateway-app meta-mycompany-apps
devtool finish libmodbus meta-mycompany-apps
```

---

## 17. Common Pitfalls and Debugging

### 1. Forgetting to source environment-setup-* in a new shell
Exactly analogous to forgetting `oe-init-build-env` in a full build environment — every new terminal session needs the SDK's environment script re-sourced, or `$CC`/`$CXX`/cross-tooling simply won't be on `PATH`/correctly configured.

### 2. Linking against the build host's native libraries instead of the target sysroot
Usually a CMake `find_package`/`find_library` misconfiguration (missing or incorrect `CMAKE_FIND_ROOT_PATH_MODE_*` settings) or a hand-written Makefile not respecting `$PKG_CONFIG_SYSROOT_DIR`/`$PKG_CONFIG_PATH`. Symptom: the binary compiles and links successfully on the build host but fails immediately with a "wrong ELF class" or illegal-instruction error when actually run on the target, because it accidentally linked against the build host's own x86_64 libraries instead of the target's aarch64 ones.

### 3. Using a Standard SDK and then wanting to add a new dependency
This is simply not supported by design — a Standard SDK's sysroot is fixed at generation time. The fix is either requesting/generating an updated SDK (with the new dependency now included in the source image) from the platform team, or switching to an Extensible SDK, which DOES support this via `devtool add`.

### 4. eSDK installer is enormous and slow to distribute/install
Consider `SDK_EXT_TYPE = "minimal"` (trading installer size for requiring network access to an sstate mirror during `devtool` operations) if the development team has reliable, fast access to a shared internal network sstate mirror — the "full" bundled-sstate default prioritizes complete offline capability over installer size, which isn't always the right tradeoff for every team's actual working conditions.

### 5. SDK and deployed target platform versions silently drift apart
Without disciplined SDK/platform version tracking (Section 15), an application built against a stale SDK can pass all local testing (against the SDK's own bundled sysroot/QEMU, if used) but fail in subtle, hard-to-reproduce ways once deployed against a genuinely different-versioned real target. Treat SDK version alignment with the same rigor as any other build dependency version management.

### 6. devtool deploy-target failing inside an eSDK specifically
The exact same SSH/connectivity requirements as the full-build-environment devtool (covered in the devtool document) apply identically inside an eSDK — verify SSH connectivity to the target manually first, and confirm the target image has an SSH server and (for development) appropriate `debug-tweaks`-style permissive authentication enabled.

---

## 18. Interview Questions and Answers

**Q1: Explain the fundamental architectural difference between a Standard SDK and an Extensible SDK, beyond just "the eSDK is bigger."**

**A:** The fundamental difference is that a Standard SDK is a STATIC SNAPSHOT — a cross-compiler toolchain plus a fixed, point-in-time target sysroot, generated once and then immutable; using it is conceptually identical to using any traditional, pre-built cross-compilation toolchain package, just one that happens to have been correctly generated to exactly match a specific Yocto-built image's package set. An Extensible SDK, by contrast, is architecturally a genuinely real, functioning (if intentionally scoped and heavily pre-cached) BitBake build environment — it includes actual layer copies, a real `conf/local.conf`/`bblayers.conf`, and a working `bitbake` binary, with a substantial slice of the platform's `sstate-cache` bundled in specifically so that this embedded build environment can perform REAL, incremental BitBake builds (adding genuinely new recipes via `devtool add`, modifying existing platform recipes' source via `devtool modify`) quickly, without needing network access to rebuild things from scratch every time. The practical consequence of this architectural difference is capability: a Standard SDK can only ever compile/link application code against whatever was already baked into its sysroot at generation time, while an Extensible SDK can genuinely extend and modify the platform's own recipe set on the fly, using the exact same underlying mechanism (BitBake + devtool) a full platform build environment would use — just scoped to one application developer's machine and accelerated by pre-cached build output rather than requiring the full, multi-hour, tens-of-gigabytes platform build infrastructure.

---

**Q2: Why does the SDK's target sysroot need to exactly match the specific image it was generated from, rather than just being some generic "ARM Linux" sysroot?**

**A:** An application's cross-compiled binary needs to link against EXACTLY the same library versions, with exactly the same compiled-in features/ABI, that will actually be present on the real target device at runtime — any mismatch (a different library major version with a changed function signature, a library compiled with a different set of optional features enabled/disabled, a different glibc version with subtly different symbol versioning) risks either an outright link/runtime failure (a missing symbol) or, more insidiously, a successful link against a subtly INCOMPATIBLE library version that produces incorrect behavior only discovered much later, possibly only in specific edge-case runtime conditions. Because Yocto already custom-builds the ENTIRE software stack for a specific image — not just the application layer, but the C library, every system library, every dependency, often with product-specific configuration/patches applied — a generic "ARM Linux sysroot" sourced from anywhere else (a generic prebuilt cross-toolchain distribution, a different Linux distribution's ARM packages) would almost certainly NOT exactly match this specific, customized software stack. Generating the SDK's sysroot directly FROM the actual built image (`populate_sdk` literally examines the source image's installed package set and stages exactly those packages' headers/libraries into the SDK) is what guarantees the SDK's API/ABI surface is a perfect, trustworthy match for what the application will actually encounter at runtime on a real device running that exact image.

---

**Q3: A team is deciding between Standard SDK and Extensible SDK for their application development team. What questions would help determine which is the better fit?**

**A:** Several practical questions matter: (1) **Will application developers ever need to add NEW dependencies/libraries not already present in the platform image, or modify existing platform package source as part of their normal workflow?** If yes, only the Extensible SDK supports this — a Standard SDK would force every such need to go back through the platform team requesting and republishing an updated SDK, creating friction and a dependency bottleneck. If application developers genuinely only ever need to compile against a FIXED, well-defined, rarely-changing platform API surface (a common, deliberate API-stability design choice for some product architectures), a Standard SDK's simplicity and smaller size may be entirely sufficient and preferable. (2) **How comfortable is the application development team with BitBake/devtool concepts, versus wanting the simplest possible "just run gcc/cmake" experience?** A Standard SDK's usage model (source the environment script, run your normal build commands) requires zero BitBake/Yocto-specific knowledge beyond initial setup; an Extensible SDK, while still accessible, does introduce devtool-specific workflow concepts that benefit from at least some familiarity. (3) **What are the team's installer size/network-access constraints?** A Standard SDK is meaningfully smaller and has zero ongoing network dependency once installed; an Extensible SDK (especially in its default "full" mode) is larger to distribute but works entirely offline afterward, while a "minimal" eSDK variant trades a smaller installer for requiring ongoing network access to an sstate mirror. In practice, many organizations end up providing BOTH — a Standard SDK for simple, stable-API application builds and CI pipelines, and an Extensible SDK specifically for more exploratory/iterative application development work where the ability to add dependencies and modify platform source on the fly provides real, frequently-used value.

---

**Q4: Why is it important for CMake-based projects using the SDK to have CMAKE_FIND_ROOT_PATH_MODE_* correctly configured, and what specifically goes wrong if it isn't?**

**A:** CMake's `find_package()`/`find_library()`/`find_path()` commands, by default (without cross-compilation-aware configuration), search the BUILD HOST's own standard system paths (`/usr/lib`, `/usr/include`, etc.) — which is exactly the CORRECT behavior for a normal, non-cross-compiling build, but catastrophically WRONG for cross-compilation, since the build host's own libraries are almost certainly a different architecture (and likely different versions/feature sets) than what the TARGET device actually needs. The `CMAKE_FIND_ROOT_PATH` variable (set to the target sysroot path) combined with the `CMAKE_FIND_ROOT_PATH_MODE_*` settings (`ONLY` for libraries/includes/packages, but `NEVER` for programs) tells CMake's find-commands to search EXCLUSIVELY within the target sysroot for libraries and headers (preventing accidental discovery of build-host libraries), while still allowing `find_program()` to search the normal host PATH for build-time-only tools (a code generator, a templating tool) that genuinely need to RUN on the build host during the build process itself, not be linked into the final target binary. If this is misconfigured (or simply absent, if a project's CMakeLists.txt was written without cross-compilation in mind and the SDK's auto-generated toolchain file isn't actually being used), the symptom is exactly the kind of "compiles and links fine on the build host, then fails immediately at runtime on the actual target" bug described in the Common Pitfalls section — the build silently succeeded by accidentally linking against the WRONG (build-host-architecture) version of a library, producing a binary that's either outright unable to run on the target (wrong ELF architecture) or, in subtler cases, linked against incompatible library versions, producing incorrect runtime behavior.

---

**Q5: How does devtool sdk-update avoid requiring application developers to reinstall their entire eSDK every time the underlying platform is updated, and why does this matter practically?**

**A:** `devtool sdk-update` works by fetching an incremental UPDATE to the eSDK's bundled `sstate-cache` and layer metadata from a published update feed (a URL the platform team's CI publishes to after each relevant platform build), rather than requiring a complete reinstallation of the entire several-gigabyte eSDK package from scratch. Because the eSDK's core architecture is genuinely a real, if scoped, BitBake build environment with its own local sstate-cache, "updating" it conceptually means: fetch the newer sstate objects corresponding to whatever changed in the underlying platform build (an updated library version, a new kernel build, a newly added recipe), merge them into the eSDK's existing local sstate-cache and layer checkout, without disturbing anything else that hasn't changed. This matters practically because application developers, especially on a team actively iterating on both application code AND the underlying platform simultaneously (a common situation during active product development), would otherwise face significant friction if every platform update required a multi-gigabyte, multi-minute-to-install fresh eSDK download — `sdk-update`'s incremental nature means staying current with the platform's latest build is a quick, low-friction, frequent operation rather than a heavyweight, dreaded chore that teams might be tempted to defer (with the corresponding risk of developing and testing against an increasingly stale, divergent platform snapshot, exactly the SDK/platform version drift problem discussed in Section 15).
