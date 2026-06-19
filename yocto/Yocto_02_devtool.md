# Yocto devtool — Complete Reference Guide

---

## Table of Contents

1. [Introduction to devtool](#1-introduction-to-devtool)
2. [devtool Architecture and Workspace](#2-devtool-architecture-and-workspace)
3. [devtool add — Creating New Recipes](#3-devtool-add--creating-new-recipes)
4. [devtool modify — Editing Existing Recipes](#4-devtool-modify--editing-existing-recipes)
5. [devtool upgrade — Bumping Recipe Versions](#5-devtool-upgrade--bumping-recipe-versions)
6. [devtool build and Testing Workflow](#6-devtool-build-and-testing-workflow)
7. [devtool deploy-target — Live Device Testing](#7-devtool-deploy-target--live-device-testing)
8. [devtool finish — Completing the Workflow](#8-devtool-finish--completing-the-workflow)
9. [devtool edit-recipe and Recipe Inspection](#9-devtool-edit-recipe-and-recipe-inspection)
10. [devtool sdk-update and Extensible SDK Integration](#10-devtool-sdk-update-and-extensible-sdk-integration)
11. [devtool ide-sdk — IDE Integration](#11-devtool-ide-sdk--ide-integration)
12. [Complete Command Reference](#12-complete-command-reference)
13. [Complete Workflow Examples](#13-complete-workflow-examples)
14. [devtool Configuration](#14-devtool-configuration)
15. [Common Pitfalls and Debugging](#15-common-pitfalls-and-debugging)
16. [Interview Questions and Answers](#16-interview-questions-and-answers)

---

## 1. Introduction to devtool

**devtool** is a command-line tool that streamlines the most common, repetitive Yocto/OpenEmbedded development workflows: adding a new piece of software, modifying an existing recipe's source code, upgrading a recipe to a new upstream version, and testing changes on real target hardware. Without devtool, all of these tasks require manually editing recipe files, manually managing `${WORKDIR}` source trees, manually re-running BitBake commands, and manually copying files to a target device via scp/ssh.

devtool automates this entire loop. It is included as part of the standard Poky/Yocto distribution and is also the primary interface to the **Extensible SDK (eSDK)** — meaning the exact same `devtool` commands work whether you're inside a full Yocto build environment or inside a lightweight eSDK installed on an application developer's machine who doesn't have (or want) the full BitBake build infrastructure.

**Core problems devtool solves:**

| Problem                                                | devtool Solution                          |
|----------------------------------------------------------|----------------------------------------------|
| "I have source code for a new tool, how do I make a recipe?" | `devtool add`                            |
| "I need to debug/modify an existing recipe's source"     | `devtool modify`                            |
| "Upstream released a new version, how do I update the recipe?" | `devtool upgrade`                       |
| "I want to test my change on the actual board without rebuilding the whole image" | `devtool deploy-target`     |
| "I'm done iterating, how do I get a clean recipe back into my layer?" | `devtool finish`                    |

---

## 2. devtool Architecture and Workspace

### The Workspace Layer

devtool operates through a special, auto-managed BitBake layer called the **workspace**, conventionally located at `build/workspace/`. This layer is automatically added to `bblayers.conf` (with the highest priority) the first time you run any `devtool` command that needs it.

```
build/workspace/
├── conf/
│   └── layer.conf              ← auto-generated, marks this as a layer
├── appends/
│   └── busybox_1.36.1.bbappend ← auto-generated bbappend pointing devtool's
│                                   modified recipe at the extracted source
├── recipes/
│   └── myapp/
│       └── myapp_1.0.bb        ← for devtool add: the new recipe lives here
│                                   until devtool finish moves it to a real layer
└── sources/
    ├── busybox/                ← extracted/checked-out source for devtool modify
    └── myapp/                  ← extracted/checked-out source for devtool add
```

Because the workspace layer has the **highest layer priority**, any `.bbappend` it generates automatically overrides the original recipe in whatever layer it came from — without you ever editing that original layer's files directly. This is the central mechanism behind devtool's non-destructive workflow.

### High-Level Workflow Diagram

```
                    ┌─────────────────┐
                    │  devtool add /   │
                    │  devtool modify  │
                    └────────┬─────────┘
                             │
                  Creates workspace recipe +
                  extracts/checks out source to
                  build/workspace/sources/<recipe>/
                             │
                             ▼
                ┌─────────────────────────┐
                │  Edit source code in     │
                │  your favorite editor/IDE│
                └────────┬─────────────────┘
                         │
                         ▼
              ┌────────────────────┐
              │  devtool build      │  ← incremental build using devtool's
              └────────┬────────────┘     extracted source, fast iteration
                       │
                       ▼
           ┌─────────────────────────┐
           │  devtool deploy-target   │  ← push binary directly to running
           └────────┬──────────────-──┘     target device over SSH, test live
                    │
                    ▼ (repeat edit/build/deploy loop as needed)
           ┌─────────────────────────┐
           │  devtool finish          │  ← generate final patches, move recipe
           └─────────────────────────┘     into a real layer, clean workspace
```

---

## 3. devtool add — Creating New Recipes

`devtool add` bootstraps a brand-new recipe from existing source code — whether that source is a local directory, a git repository, or a tarball URL. It attempts to auto-detect the build system (Autotools, CMake, Make, Python setuptools, etc.) and license, generating a starting-point recipe you then refine.

### Basic Usage

```bash
# From a remote git repository
devtool add myapp https://github.com/example/myapp.git

# From a local source directory (e.g., you're actively writing the app)
devtool add myapp /home/user/projects/myapp

# Specify the recipe name explicitly when it can't be inferred
devtool add myapp /home/user/projects/myapp --version 1.0

# From a tarball URL
devtool add myapp https://example.com/releases/myapp-1.0.tar.gz

# Specify which existing recipe in the layer index to base it on (template)
devtool add --template cmake myapp /home/user/projects/myapp
```

### What devtool add Does Internally

1. Fetches/copies the source into `build/workspace/sources/myapp/`
2. Inspects the source tree: looks for `configure.ac`/`Makefile.am` (Autotools), `CMakeLists.txt` (CMake), `setup.py`/`pyproject.toml` (Python), `Cargo.toml` (Rust), etc.
3. Auto-detects `LICENSE` and attempts to compute `LIC_FILES_CHKSUM` by scanning for COPYING/LICENSE files
4. Generates a starter recipe at `build/workspace/recipes/myapp/myapp_1.0.bb` with:
   - `SRC_URI` pointing at the source (or `S = "${WORKDIR}/sources-unpack"` style references for local source)
   - `inherit` of the detected build system class (autotools, cmake, setuptools3, cargo, etc.)
   - A `LICENSE = "CLOSED"` placeholder if it cannot determine the license (flagging it for manual review)
5. Adds the workspace layer to `bblayers.conf` if not already present

### Example Session

```bash
$ devtool add myhello https://github.com/example/myhello.git
NOTE: Fetching https://github.com/example/myhello.git...
NOTE: Source tree extracted to /home/user/poky/build/workspace/sources/myhello
NOTE: Recipe myhello_1.0.bb has been automatically created; further editing
      may be required to make it fully functional

$ devtool edit-recipe myhello
# Opens build/workspace/recipes/myhello/myhello_1.0.bb in $EDITOR

$ cat build/workspace/recipes/myhello/myhello_1.0.bb
SRC_URI = "git://github.com/example/myhello.git;protocol=https;branch=main"
SRCREV = "abc123..."
PV = "1.0+git${SRCPV}"
LICENSE = "CLOSED"   # ← devtool flags this; must fix manually if a real license exists
S = "${WORKDIR}/git"
inherit cmake
```

### Important Flags

| Flag                 | Purpose                                                          |
|------------------------|----------------------------------------------------------------|
| `--version <ver>`     | Explicitly set PV instead of auto-detecting                       |
| `--no-git`            | Don't initialize a git repo in the extracted source tree           |
| `--fetch-dev`         | Fetch a development branch instead of a release tag                |
| `--binary`            | Treat the source as a pre-built binary package (uses bin_package class)|
| `--also-native`       | Also create a `-native` variant recipe for build-host execution     |
| `--npm`               | Create an npm-based recipe (Node.js package)                       |
| `-f, --fetch`         | Equivalent to specifying the URL as the source positional argument |

---

## 4. devtool modify — Editing Existing Recipes

`devtool modify` is used on a recipe that **already exists** somewhere in your layers — the most common devtool command in day-to-day development, used to fix a bug, add a patch, or debug a build failure in an existing piece of software.

### Basic Usage

```bash
# Extract busybox's source for local modification
devtool modify busybox

# Extract source into a specific directory instead of the default workspace location
devtool modify busybox /home/user/busybox-src

# Don't extract source — just create the bbappend pointing at existing source
devtool modify -n busybox

# Modify and immediately apply any existing patches from the original recipe
devtool modify busybox
```

### What devtool modify Does Internally

1. Locates the recipe (e.g., `busybox_1.36.1.bb`) across all configured layers
2. Runs the recipe's `do_fetch` and `do_unpack` (and applies existing `do_patch`) to materialize the full, already-patched source tree
3. Copies/checks out this source tree into `build/workspace/sources/busybox/`
4. Initializes a **git repository** in that source tree, with the pristine (post-patch) state committed as the baseline — this is critical, because it means any further edits you make can be diffed against this baseline to auto-generate new patches later
5. Generates `build/workspace/appends/busybox_1.36.1.bbappend` that:
   - Overrides `SRC_URI`/`S` to point at the workspace source directory instead of the original fetch location
   - Sets `EXTERNALSRC = "${WORKSPACE}/sources/busybox"` (via `inherit externalsrc`)

### The externalsrc Mechanism

```bitbake
# Auto-generated in build/workspace/appends/busybox_1.36.1.bbappend
inherit externalsrc
EXTERNALSRC = "/home/user/poky/build/workspace/sources/busybox"
EXTERNALSRC_BUILD = "/home/user/poky/build/workspace/sources/busybox"
```

`externalsrc.bbclass` is the key enabler: it disables `do_fetch`/`do_unpack`/`do_patch` entirely and instead points `${S}` (and `${B}` for out-of-tree builds) directly at your live, editable source directory on disk. This means every `devtool build` afterward uses your **current, uncommitted edits directly** — no re-fetch, no re-patch, no copy step. You edit a file, run `devtool build`, and only the changed files get recompiled (assuming the underlying build system, like Make or Ninja, supports incremental builds, which almost all do).

### Editing the Source

```bash
$ devtool modify busybox
$ cd build/workspace/sources/busybox
$ vim networking/wget.c
# ... make your fix ...
$ devtool build busybox
```

---

## 5. devtool upgrade — Bumping Recipe Versions

`devtool upgrade` automates the tedious process of updating a recipe to a newer upstream release: fetching the new source, recomputing checksums, attempting to re-apply existing patches, and flagging any patches that no longer apply cleanly (needing manual rebase).

### Basic Usage

```bash
# Upgrade to the latest version devtool can detect (checks SRC_URI's upstream)
devtool upgrade busybox

# Upgrade to a specific version
devtool upgrade busybox --version 1.37.0

# Upgrade to a specific git SRCREV (commit) instead of a tagged version
devtool upgrade myapp --srcrev abc123def456

# Upgrade and specify a different source URI entirely (e.g., upstream moved)
devtool upgrade myapp --srcbranch develop
```

### What devtool upgrade Does Internally

1. Determines the new version/SRCREV (auto-checks upstream via `UPSTREAM_CHECK_URI`/`UPSTREAM_CHECK_GITTAGREGEX` if set in the recipe, or uses the explicitly given `--version`)
2. Fetches the new source version into the workspace
3. Initializes git history exactly like `devtool modify`, but on the NEW version's pristine source
4. Attempts to **re-apply every patch** from the old recipe onto the new source tree, one at a time
5. Reports which patches applied cleanly and which failed (needing manual rebase/resolution)
6. Generates an updated recipe file with the new `PV`, new `SRCREV`/checksums, and updated patch list

### Example Session

```bash
$ devtool upgrade busybox --version 1.37.0
NOTE: Extracting current version source...
NOTE: Fetching upgraded version source (1.37.0)...
NOTE: Applying patches on top of the new version...
NOTE: Patch 0001-fix-buffer-overflow.patch applied successfully
NOTE: Patch 0002-add-custom-applet.patch FAILED to apply — needs manual rebase
NOTE: New recipe created in workspace, patch 0002 needs attention

$ cd build/workspace/sources/busybox
$ git status
# Shows the failed patch as a conflict to resolve manually
$ git am --abort   # or resolve conflicts, then:
$ git apply --reject ../../../meta-mylayer/recipes-core/busybox/files/0002-add-custom-applet.patch
# Manually fix the .rej hunks, then:
$ devtool finish busybox meta-mylayer
```

### Checking for Available Upgrades (without performing one)

```bash
# Check a single recipe for newer upstream versions
devtool check-upgrade-status busybox

# Check ALL recipes in the build for available upgrades (slow — scans everything)
devtool check-upgrade-status -a
```

This relies on recipe metadata like:
```bitbake
UPSTREAM_CHECK_URI = "https://busybox.net/downloads/"
UPSTREAM_CHECK_REGEX = "busybox-(?P<pver>\d+(\.\d+)+)\.tar"
# Or for git-based recipes:
UPSTREAM_CHECK_GITTAGREGEX = "v(?P<pver>\d+(\.\d+)+)"
```

---

## 6. devtool build and Testing Workflow

```bash
# Build the recipe using the current workspace source
devtool build busybox

# Equivalent to: bitbake busybox, but devtool ensures the externalsrc
# mechanism is correctly engaged and gives more focused output

# Build only up to a specific task
devtool build -s busybox      # -s = skip do_install/packaging, just compile
```

### Incremental Builds

Because `externalsrc` bypasses fetch/unpack/patch entirely, and most build systems (Make, CMake+Ninja, Meson+Ninja) support incremental rebuilds, `devtool build` after a small source edit typically only recompiles the changed files — turning what might be a 10-minute full BitBake recipe build into a 5-second incremental one. This is devtool's single biggest productivity advantage over manually editing recipes and re-running `bitbake -f -c compile`.

### Building the Full Image with Your Changes

```bash
# After devtool modify/build, rebuild the full image to include your change
bitbake core-image-minimal

# devtool-modified recipes are automatically picked up because the
# workspace bbappend overrides the original recipe's source location
```

---

## 7. devtool deploy-target — Live Device Testing

This is devtool's most powerful feature for hardware bring-up and iterative debugging: it pushes your locally-built binary package(s) **directly onto a running target device** over SSH/SCP, without rebuilding or reflashing the entire root filesystem image.

### Basic Usage

```bash
# Deploy the built recipe output to a device at a given IP/hostname
devtool deploy-target busybox root@192.168.1.50

# Specify SSH options (e.g., custom port, identity file)
devtool deploy-target busybox root@192.168.1.50 -P 2222

# Dry run — show what WOULD be deployed without actually copying files
devtool deploy-target busybox root@192.168.1.50 --dry-run

# Show what files were previously deployed (for this recipe)
devtool deploy-target -s busybox root@192.168.1.50
```

### What devtool deploy-target Does Internally

1. Looks at the recipe's `do_install` output (everything staged in `${D}`)
2. Computes the list of files that would be installed onto the target's root filesystem
3. Uses `scp`/`rsync` over SSH to copy those exact files to the corresponding paths on the live target device
4. Records a manifest of deployed files on the target (in `/etc/devtool-deploy/` or similar) so a later `devtool undeploy-target` can cleanly remove exactly what was added, restoring the device's prior state

### Requirements on the Target

- The target image must have an SSH server running (e.g., `dropbear` or `openssh-sshd` included via `IMAGE_FEATURES += "ssh-server-dropbear"`)
- Passwordless root login or a working SSH key, OR the `debug-tweaks` `EXTRA_IMAGE_FEATURES` for development images (which sets an empty root password and enables root SSH login — development only, never for production images)

```bitbake
# In local.conf, for development images:
EXTRA_IMAGE_FEATURES = "debug-tweaks"
# debug-tweaks: empty root password, allow root ssh login, "post-install" logging
```

### Removing a Deployed Package

```bash
# Reverts the target to its pre-deploy state for this recipe
devtool undeploy-target busybox root@192.168.1.50

# Remove ALL devtool-deployed packages from a target
devtool undeploy-target -a root@192.168.1.50
```

### Typical Hardware Bring-Up Loop

```bash
devtool modify mydriver-app
# edit source...
devtool build mydriver-app
devtool deploy-target mydriver-app root@192.168.1.50
ssh root@192.168.1.50 "/usr/bin/mydriver-app --test"
# observe failure, edit source again...
devtool build mydriver-app
devtool deploy-target mydriver-app root@192.168.1.50   # only changed files re-copied
# repeat until working
devtool finish mydriver-app meta-mylayer
```

---

## 8. devtool finish — Completing the Workflow

Once you're satisfied with your changes (whether from `devtool add`, `devtool modify`, or `devtool upgrade`), `devtool finish` converts your live-edited workspace source back into a **proper, patch-based recipe** in a real layer — the form needed for a clean, reproducible, committable build.

### Basic Usage

```bash
# Finish and place the result into meta-mylayer
devtool finish busybox meta-mylayer

# Finish a devtool-add'ed brand-new recipe into a layer
devtool finish myapp meta-mylayer

# Specify exactly where in the layer the recipe should land
devtool finish myapp meta-mylayer --recipe-path meta-mylayer/recipes-myapp/myapp
```

### What devtool finish Does Internally

1. Diffs the current state of `build/workspace/sources/<recipe>/` against the git history baseline (created when `devtool modify`/`add` first ran)
2. Generates one or more `.patch` files representing your changes (one patch per git commit if you made multiple commits in the workspace source — **this is why committing your changes incrementally inside the workspace source tree with meaningful commit messages is a best practice**, since each commit becomes a clean, reviewable patch file)
3. For `devtool modify`: updates the original recipe's `SRC_URI` to include the new patches, and removes the workspace bbappend, copying the patches into the target layer's `files/` directory
4. For `devtool add`: moves the entire generated recipe (and any patches) from the workspace into the specified real layer, with proper directory structure (`recipes-<category>/<pn>/<pn>_<pv>.bb`)
5. For `devtool upgrade`: writes out the new `PV`, `SRCREV`/checksums, and rebased patch set into the real layer's recipe
6. Removes the recipe from the workspace, cleaning up `build/workspace/`

### Git Commit Best Practice Inside the Workspace

```bash
$ cd build/workspace/sources/busybox
$ vim networking/wget.c
$ git add networking/wget.c
$ git commit -m "wget: fix buffer overflow in header parsing"
$ vim networking/wget.c
$ git add networking/wget.c
$ git commit -m "wget: add --no-check-certificate option"

$ devtool finish busybox meta-mylayer
# Generates TWO separate patch files, one per commit, each cleanly named
# and placed in meta-mylayer/recipes-core/busybox/busybox/
#   0001-wget-fix-buffer-overflow-in-header-parsing.patch
#   0002-wget-add-no-check-certificate-option.patch
```

This produces a much cleaner, more reviewable result than making all your edits uncommitted and running `devtool finish`, which would squash everything into a single large, hard-to-review patch.

---

## 9. devtool edit-recipe and Recipe Inspection

```bash
# Open the recipe file in $EDITOR (works for both workspace and regular recipes)
devtool edit-recipe busybox

# Specify a particular editor for this invocation
EDITOR=code devtool edit-recipe busybox

# Show the recipe's full file path without opening an editor
devtool edit-recipe -p busybox
```

### Other Inspection Commands

```bash
# List all recipes currently active in the devtool workspace
devtool status

# Show detailed info about a specific recipe (source location, version, etc.)
devtool find-recipe busybox

# Search all configured layers for recipes matching a pattern
devtool search '^lib.*ssl'

# Show the layer a given recipe currently comes from
bitbake-layers show-recipes busybox
```

### devtool status Example Output

```bash
$ devtool status
busybox: /home/user/poky/build/workspace/sources/busybox (modify)
myapp: /home/user/poky/build/workspace/sources/myapp (add)
mydriver: /home/user/poky/build/workspace/sources/mydriver (upgrade, current=1.2, new=1.3)
```

---

## 10. devtool sdk-update and Extensible SDK Integration

The **Extensible SDK (eSDK)** ships `devtool` as its primary developer interface, allowing application developers to use the exact same workflow without needing a full Yocto build environment (no need to clone poky, configure layers, or run multi-hour initial builds).

```bash
# Inside an installed eSDK environment (after sourcing environment-setup-*):
devtool add myapp /home/user/myapp-source
devtool build myapp
devtool deploy-target myapp root@192.168.1.50

# Update the eSDK's own internal sstate/metadata to match a newer image build
devtool sdk-update

# Specify where to pull updates from (a network-published sstate/eSDK update server)
devtool sdk-update http://build-server.example.com/sdk-updates/
```

### eSDK vs Full Build Environment — devtool Differences

| Aspect                        | Full Yocto Build               | Extensible SDK                          |
|---------------------------------|----------------------------------|--------------------------------------------|
| Initial setup time              | Hours (first full image build)  | Minutes (pre-built sstate snapshot installer)|
| Disk space                      | 50GB+                            | A few GB                                   |
| `devtool build` backend         | Local BitBake + local sstate     | Local BitBake + **pre-populated** sstate from the eSDK installer, only missing pieces get built|
| Use case                        | BSP/platform/distro engineers    | Application developers building on top of an already-released platform image |
| Updating to new platform release| `git pull` + rebuild              | `devtool sdk-update` against a published update server|

---

## 11. devtool ide-sdk — IDE Integration

Newer Yocto releases include `devtool ide-sdk`, which generates IDE-specific configuration (VSCode `launch.json`/`tasks.json`, for example) for a workspace recipe — enabling **remote GDB debugging** directly on target hardware from within an IDE, using the correct cross-debugger and sysroot paths automatically.

```bash
# Generate VSCode debug configuration for a devtool-modified recipe
devtool ide-sdk myapp --target root@192.168.1.50 vscode

# This generates .vscode/launch.json and tasks.json inside the workspace
# source directory, pre-configured with:
#  - The correct cross gdb path (e.g., arm-poky-linux-gnueabi-gdb)
#  - gdbserver remote-target connection settings
#  - Correct sysroot paths for source-level debugging
```

This closes the loop from "manually configuring cross-gdb and gdbserver by hand" to "click Debug in VSCode and step through target-board code directly," which previously required substantial manual toolchain and path configuration.

---

## 12. Complete Command Reference

| Command                        | Purpose                                                          |
|-----------------------------------|----------------------------------------------------------------|
| `devtool add`                    | Create a new recipe from source (local dir, git, tarball)        |
| `devtool modify`                 | Extract an existing recipe's source for local editing             |
| `devtool upgrade`                | Bump a recipe to a newer upstream version, rebasing patches        |
| `devtool build`                  | Build a workspace recipe (incremental, fast iteration)             |
| `devtool build-image`            | Build a complete image including all workspace-modified recipes    |
| `devtool build-sdk`              | Build an SDK including workspace changes                            |
| `devtool deploy-target`          | Push built output directly to a running target device over SSH    |
| `devtool undeploy-target`        | Remove previously deployed files from a target device              |
| `devtool finish`                 | Convert workspace changes back into a clean, patch-based recipe in a real layer|
| `devtool reset`                  | Discard workspace changes for a recipe without finishing            |
| `devtool status`                 | List all recipes currently active in the workspace                  |
| `devtool edit-recipe`            | Open a recipe file in $EDITOR                                       |
| `devtool find-recipe`            | Show detailed info about a recipe                                   |
| `devtool search`                 | Search all layers for matching recipes                              |
| `devtool rename`                 | Rename a workspace recipe (and its PN/PV)                           |
| `devtool configure-help`         | Show configure script's available options for a recipe's source     |
| `devtool extract`                | Extract a recipe's source WITHOUT setting up the full workspace machinery|
| `devtool sync`                   | Re-synchronize an extracted source tree with the recipe's current patch set|
| `devtool check-upgrade-status`   | Check if newer upstream versions are available, without upgrading   |
| `devtool sdk-update`             | Update an installed eSDK to a newer published snapshot               |
| `devtool ide-sdk`                | Generate IDE debug configuration (VSCode, etc.) for a workspace recipe|
| `devtool create-workspace`       | Manually create/select a workspace layer (rarely needed — automatic)|
| `devtool latest-version`         | Query the latest known upstream version for a recipe                 |

---

## 13. Complete Workflow Examples

### Example 1: Adding a Brand New Application

```bash
# 1. Add the new application recipe from a git repo
devtool add my-sensor-daemon https://github.com/myorg/sensor-daemon.git

# 2. Inspect/fix the auto-generated recipe
devtool edit-recipe my-sensor-daemon
# Fix LICENSE, LIC_FILES_CHKSUM, add any missing DEPENDS

# 3. Build it
devtool build my-sensor-daemon

# 4. Test on real hardware
devtool deploy-target my-sensor-daemon root@192.168.1.50
ssh root@192.168.1.50 "/usr/bin/sensor-daemon --version"

# 5. Iterate: edit source, rebuild, redeploy
vim build/workspace/sources/my-sensor-daemon/src/main.c
devtool build my-sensor-daemon
devtool deploy-target my-sensor-daemon root@192.168.1.50

# 6. Once satisfied, finalize into your layer
devtool finish my-sensor-daemon meta-mylayer

# 7. Verify the final recipe builds cleanly from scratch (no workspace)
bitbake -c cleansstate my-sensor-daemon
bitbake my-sensor-daemon
```

### Example 2: Fixing a Bug in an Existing Recipe (e.g., dropbear)

```bash
devtool modify dropbear
cd build/workspace/sources/dropbear
vim svr-auth.c
# fix the bug
git add svr-auth.c
git commit -m "dropbear: fix auth timeout race condition"

devtool build dropbear
devtool deploy-target dropbear root@192.168.1.50
# test that ssh login still works correctly and the bug is fixed

devtool finish dropbear meta-mylayer
# Generates meta-mylayer/recipes-core/dropbear/dropbear/0001-dropbear-fix-auth-timeout-race-condition.patch
# and a dropbear_%.bbappend referencing it
```

### Example 3: Upgrading a Recipe with Existing Patches

```bash
devtool check-upgrade-status nginx
# NOTE: Current version: 1.24.0, latest available: 1.26.2

devtool upgrade nginx --version 1.26.2
# patches re-applied automatically where possible

cd build/workspace/sources/nginx
git status   # check for any failed/rejected patches

devtool build nginx
devtool deploy-target nginx root@192.168.1.50
# smoke-test the new nginx version

devtool finish nginx meta-mylayer
```

### Example 4: Abandoning Changes

```bash
# Discard all workspace changes for a recipe without finishing
devtool reset busybox

# Discard ALL workspace recipes at once
devtool reset -a
```

---

## 14. devtool Configuration

devtool reads configuration from `build/conf/devtool.conf` (auto-created) and respects standard BitBake variables:

```bitbake
# build/conf/devtool.conf (auto-managed, rarely hand-edited)
[General]
workspace_path = /home/user/poky/build/workspace
```

### Relevant Variables in local.conf

```bitbake
# Where devtool/SDK look for a default deployment target (optional convenience)
DEVTOOL_DEPLOY_TARGET = "root@192.168.1.50"

# SSH options used by deploy-target (e.g., to disable strict host key checking
# for a frequently-reflashed development board with a changing host key)
SSHTOOL_OPTS = "-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
```

---

## 15. Common Pitfalls and Debugging

### 1. Forgetting to commit changes inside the workspace source before `devtool finish`
`devtool finish` generates patches from git commits in the workspace source tree. If you made changes but never `git add`/`git commit` them, `devtool finish` will either complain there's nothing to finish, or (depending on version) bundle everything as a single uncommitted diff — losing the ability to generate clean, separated patches per logical change.

### 2. `devtool deploy-target` failing with permission/SSH errors
Most often caused by the target image lacking an SSH server, or the image not having `debug-tweaks` enabled for a passwordless root account during development. Check: `ssh root@<target-ip>` manually works before trying `devtool deploy-target`.

### 3. Stale workspace bbappend causing confusing build behavior later
If you forget to run `devtool finish` or `devtool reset` and the workspace layer remains active in `bblayers.conf`, your "production" builds may silently still be using the workspace's modified (possibly uncommitted, in-progress) source instead of the real recipe — leading to "but it works on my machine" bugs in CI. Always run `devtool status` before a clean/release build to ensure the workspace is empty, and remove `build/workspace` from `bblayers.conf` for CI builds entirely.

### 4. `devtool modify` extracting source but builds still seem to use the old version
Verify the bbappend's `EXTERNALSRC` variable actually points where you think:
```bash
bitbake -e busybox | grep ^EXTERNALSRC=
```
If this doesn't show your workspace path, the bbappend isn't being applied — check `BBLAYERS` includes the workspace layer (`bitbake-layers show-layers`).

### 5. `devtool upgrade` patch rebase failures
This is expected and common for any patch that touches code the upstream project itself changed between versions. Resolve manually with standard git tooling inside the workspace source (`git am --show-current-patch`, fix conflicts, `git am --continue`), then proceed to `devtool finish`.

### 6. Incremental builds not actually being incremental
Some build systems (certain Autotools configure scripts, some CMake configurations) regenerate Makefiles on every invocation if `configure.ac`/`CMakeLists.txt` timestamps look newer than cached output, effectively forcing a full reconfigure+rebuild every time. If `devtool build` seems unexpectedly slow on every iteration, check whether the recipe's build system is being needlessly reconfigured — sometimes fixable by touching the right stamp files or by recipe-specific `do_configure[noexec]` tricks during pure source-iteration debugging (re-enable before `devtool finish`).

---

## 16. Interview Questions and Answers

**Q1: What problem does devtool solve that plain BitBake recipe editing doesn't?**

**A:** Without devtool, modifying an existing package's source requires manually running `bitbake -c unpack <recipe>`, finding the extracted source deep in `tmp/work/.../`, editing it there (a path that gets wiped on `bitbake -c clean`), manually generating a diff/patch against the original source, manually adding that patch to `SRC_URI` in a bbappend, and manually testing on hardware via separately-scripted scp commands. This is slow, error-prone, and the build-output source tree isn't meant to be a stable place to do iterative development. devtool automates the entire loop: it extracts source into a stable, git-tracked workspace directory (`devtool modify`), supports fast incremental rebuilds bypassing the normal fetch/unpack/patch tasks entirely via `externalsrc` (`devtool build`), pushes binaries straight to a running target over SSH for live testing (`devtool deploy-target`), and finally auto-generates clean, reviewable patch files from your git commit history when you're done (`devtool finish`). It turns a manual, multi-step, error-prone process into a few short, purpose-built commands.

---

**Q2: Explain how externalsrc.bbclass enables devtool's fast iteration workflow.**

**A:** Normally, a recipe's source code lives in `${WORKDIR}` (inside `tmp/work/`), populated by `do_fetch` (download), `do_unpack` (extract), and `do_patch` (apply patches) — every time you want fresh source after an edit, in principle this whole chain could be re-triggered, and the location is deep inside BitBake's managed temp directories, not somewhere a developer would naturally edit code. `externalsrc.bbclass`, which devtool automatically applies via the bbappend it generates, completely disables `do_fetch`/`do_unpack`/`do_patch` for that recipe and instead points `${S}` (and `${B}` for separate build-directory systems like CMake) directly at an arbitrary, persistent directory on disk — the devtool workspace source tree. Because that directory is never wiped between builds and is exactly where your editor is pointed, `devtool build` (which is really just `bitbake -c compile` plus `-c install`/`-c package` under the hood with externalsrc engaged) goes straight to invoking the underlying build system (make, ninja, etc.) against your current edits, and that build system's own incremental-build logic (only recompiling changed object files) takes over — giving near-native edit-compile-test speed instead of BitBake's full multi-task pipeline overhead.

---

**Q3: What's the difference between `devtool modify` and `devtool add`?**

**A:** `devtool modify <recipe>` operates on a recipe that **already exists** in one of your configured layers — it locates that existing `.bb` file, fully fetches/unpacks/patches its source (so you start from the exact same source state the recipe currently builds, including any already-applied patches), and sets up the externalsrc-based workspace so you can edit and rebuild that existing software (e.g., fixing a bug in busybox or dropbear). `devtool add <name> <source>` is for software that has **no recipe yet** — you give it a source location (local directory, git URL, or tarball URL) and it attempts to auto-detect the build system and license, generating a brand-new starter recipe from scratch in the workspace. In short: `modify` edits something that exists; `add` creates something new. Both eventually converge on the same `devtool build` / `devtool deploy-target` / `devtool finish` workflow once the workspace recipe and source are set up.

---

**Q4: Why does devtool generate one patch per git commit during `devtool finish`, and why does this matter for code review?**

**A:** When `devtool modify`/`devtool add` first extracts source into the workspace, it initializes a git repository in that source tree with the starting (pristine, or already-patched) state committed as the baseline. As you make changes, if you commit them incrementally with `git commit` (rather than leaving everything as one giant uncommitted diff), `devtool finish` walks that commit history and converts each individual commit into its own separate `.patch` file in the final recipe's `files/` directory. This matters enormously for code review and long-term maintainability: a single 500-line patch mixing a bug fix, a new feature, and a refactor is very difficult for a reviewer (or future-you) to understand or selectively revert. Five separate, well-named, single-purpose patches (e.g., "fix race condition," "add config option," "clean up unused variable") are independently reviewable, independently revertable if one causes a regression, and self-documenting through their commit messages — which is exactly the same reasoning that applies to good git commit hygiene in any software project, just applied to the patch-based world of Yocto recipes.

---

**Q5: How does devtool deploy-target avoid needing to rebuild and reflash the entire root filesystem image?**

**A:** `devtool deploy-target` operates at the level of a single recipe's `do_install` output (the files staged in `${D}`) rather than the assembled root filesystem image. It computes exactly which files that recipe would contribute to the final rootfs, then uses `scp`/`rsync` over an existing SSH connection to copy just those files directly into the corresponding paths on a **live, already-running** target device's filesystem — overwriting only what changed. This completely bypasses the much more expensive `do_rootfs`/`do_image` pipeline (which assembles every package into a fresh image, generates the filesystem, and would then require reflashing that whole image to an SD card/eMMC and rebooting the board). The tradeoff is that `deploy-target` changes are not persistent across a full reflash and don't go through the package manager's dependency/conflict resolution the way a real rootfs build does — it's explicitly a fast, temporary, development-time mechanism, not a production deployment method, which is also why `devtool undeploy-target` exists to cleanly restore the device to its pre-deploy state once testing is done.

---

**Q6: What is the Extensible SDK and how does devtool's role differ inside it compared to a full Yocto build environment?**

**A:** The Extensible SDK (eSDK) is a self-contained, installable package generated by `bitbake <image> -c populate_sdk_ext` that bundles a pre-built sstate-cache snapshot, a minimal BitBake installation, and the same `devtool` tool — all aimed at application developers who need to build and test software against a specific platform image WITHOUT needing the full multi-hour initial Yocto build, the full layer/BSP configuration knowledge, or tens of gigabytes of build infrastructure that a platform/BSP engineer would have. Inside the eSDK, `devtool add`/`modify`/`build`/`deploy-target`/`finish` work identically from the command-line interface perspective, but `devtool build`'s underlying BitBake invocation draws from the eSDK's pre-populated sstate-cache for everything that hasn't changed — so building even a fairly involved new recipe is fast because all its dependencies (toolchain, base libraries, etc.) are already cached, rather than needing to be built from scratch. `devtool sdk-update` is the eSDK-specific command with no full-build equivalent — it refreshes the eSDK's bundled sstate/metadata snapshot against a newer published platform release, letting an application developer pick up platform updates without reinstalling the entire eSDK.
