# Rufus on Linux Implementation Plan

> Direction updated on 2026-09-13: follow `qt6-reuse-assessment.md` for current
> decisions. Develop in the Rufus fork, use Qt6 Widgets, and implement existing
> C reporting entry points through a small C++ adapter with typed Qt signals.
> Lufus is a workflow reference; retaining its worker or global logger is not
> part of the current plan. The original proposal below is retained for context.

## 1. Executive Summary

This document proposes a practical path to build a Linux-native Rufus implementation by:

- reusing the `lufus` Qt application shell as the starting UI,
- extracting the portable image-analysis and media-preparation intelligence from upstream Rufus,
- replacing the current `lufus` backend with a new Linux-specific backend,
- delivering the project in phases so that useful functionality appears early.

The key conclusion from the current codebase is:

- `lufus` is worth keeping as a UI shell and proof of workflow.
- `lufus` is not a strong backend foundation.
- the most valuable code to port from Rufus is not the Win32 UI, but the image-analysis, extraction, patching, WIM handling, and boot-media decision logic.

## 2. Goals

Primary goals:

- Build a reliable Linux desktop application for creating bootable USB media.
- Preserve the parts of Rufus that make it more than "just another USB writer".
- Support both raw-image writing and Rufus-style extraction mode.
- Support modern Windows media preparation, including key Windows customization options.
- Build a backend that is testable, modular, and safe under Linux privilege boundaries.

Secondary goals:

- Reuse as much portable Rufus logic as is sensible.
- Reduce Linux-specific shell-out behavior over time by moving important logic into code.
- Make the resulting codebase easier to maintain than the current Windows-centric monolith.

## 3. Non-Goals for the First Releases

These items should not block the first usable Linux release:

- Full parity with all current Rufus features.
- Windows To Go parity.
- Every advanced filesystem Rufus supports on Windows.
- Microsoft ISO download flows.
- Every long-tail media workaround from day one.
- Perfect feature parity for every Windows 11 customization option.

## 4. Current State Assessment

### 4.1 What to keep from `lufus`

These parts are worth keeping initially:

- `ui/` forms and window flow
- progress and logging widgets
- basic app startup and user interaction structure
- `CMakeLists.txt` and Qt6 direction

Relevant files:

- `lufus/ui/rufuswindow.cpp`
- `lufus/ui/log.cpp`
- `lufus/ui/*.ui`
- `lufus/CMakeLists.txt`

### 4.2 What to replace from `lufus`

These parts should be treated as temporary prototypes and replaced:

- `lufus/rufusworker.cpp`
- `lufus/linux/devices.c`
- `lufus/linux/mounting.c`
- `lufus/linux/partition.c`
- `lufus/linux/fat32.c`
- `lufus/iso.c`

Why:

- the worker path is too narrow and fragile,
- cleanup/error handling is unsafe,
- the implementation assumes simple FAT32 formatting plus mount-and-copy,
- several UI choices are not backed by real implementation,
- the media logic is far smaller than the real Rufus engine.

### 4.3 What to port from upstream Rufus

Highest-value candidates:

- `src/iso.c`
- `src/rufus.h` image report structures and decision macros
- `src/vhd.c` WIM splitting helpers
- `src/parser.c` logic patterns for config patching
- `src/xml.c` where useful for XML support
- `src/syslinux/libinstaller/*`
- `src/ms-sys/*`
- `src/wimlib/*`
- `src/bled/*`

These contain much of Rufus's actual product value:

- image analysis
- extraction decisions
- bootloader detection
- WIM splitting
- configuration rewriting
- media-specific workarounds
- compressed-image support

## 5. Recommended Product Strategy

Build the Linux port as a three-layer system:

- `lufus-ui`: the Qt desktop application
- `rufus-core`: portable media-analysis and write-planning logic
- `linux-platform`: Linux device, filesystem, mount, and privilege services

This should be treated as a partial extraction and partial rewrite, not as a direct port of the current app.

## 6. Proposed Architecture

## 6.1 UI Layer

Responsibilities:

- collect user input,
- display scanned media and analyzed image metadata,
- surface plan choices and warnings,
- show progress/log output,
- invoke backend jobs,
- stay unprivileged.

Suggested initial reuse:

- keep `lufus/ui/`
- keep the basic main window flow
- replace direct backend calls with service interfaces

## 6.2 Core Layer (`rufus-core`)

Responsibilities:

- image inspection
- media capability detection
- write mode selection
- extraction plan generation
- config patching rules
- Windows customization plan generation
- WIM split/update planning
- bootloader and boot-mode reasoning

This layer should be largely platform-neutral.

Suggested submodules:

- `core/image/report.h`
- `core/image/analyze_iso.cpp`
- `core/image/analyze_disk_image.cpp`
- `core/image/compressed_image.cpp`
- `core/write/plan.h`
- `core/write/plan_builder.cpp`
- `core/patch/config_patch.cpp`
- `core/windows/unattend_builder.cpp`
- `core/windows/wim_patch_plan.cpp`
- `core/boot/syslinux_plan.cpp`
- `core/boot/grub_plan.cpp`

## 6.3 Linux Platform Layer

Responsibilities:

- removable device enumeration
- partition creation
- filesystem formatting
- block device raw writes
- target mounting and unmounting
- loop device management when needed
- mount propagation and cleanup
- external tool invocation
- privileged execution

Suggested submodules:

- `platform/linux/device_enumerator.cpp`
- `platform/linux/device_info.cpp`
- `platform/linux/partitioner.cpp`
- `platform/linux/filesystem_formatter.cpp`
- `platform/linux/raw_writer.cpp`
- `platform/linux/mounter.cpp`
- `platform/linux/command_runner.cpp`
- `platform/linux/privileged_helper/`

## 6.4 Privilege Model

Recommended model:

- GUI runs unprivileged.
- privileged helper performs destructive or root-only operations.
- all dangerous actions are represented as explicit commands.
- operations are logged and validated.

Preferred implementation direction:

- `polkit` + D-Bus helper

Acceptable early-stage fallback:

- helper executable launched with `pkexec`

Operations that should require privilege:

- opening block devices for write
- partition table changes
- filesystem formatting
- target mounting/unmounting
- direct raw writes
- loop and device-mapper manipulation

## 7. Reuse Plan for Existing Rufus Code

## 7.1 Directly Reusable or Adaptable

### Image report model

Start from:

- `src/rufus.h`

Port:

- the image report structure
- the decision macros
- the bootability and filesystem compatibility reasoning

This gives the Linux port the same mental model for media analysis.

### ISO/UDF analysis and extraction logic

Start from:

- `src/iso.c`

Port/adapt:

- `check_iso_props()`
- scan-time metadata collection
- config selection heuristics
- Syslinux/GRUB version detection
- EFI bootloader discovery
- special ISO workarounds

Refactor goal:

- separate scanning, extraction, and patching into smaller units
- remove Win32 file APIs
- preserve the logic, not the original shape

### WIM handling

Start from:

- `src/vhd.c`
- `src/wimlib/`

Port/adapt:

- `WimSplitFile()`
- WIM metadata inspection
- WIM update flows for Windows customization

### Syslinux support

Start from:

- `src/syslinux/libinstaller/`

Use for:

- boot sector creation
- ldlinux patching
- version-sensitive Syslinux media handling

### MBR helpers

Start from:

- `src/ms-sys/`

Use for:

- BIOS/MBR boot sector writing where still relevant

### Compressed image support

Start from:

- `src/bled/`

Use for:

- archive and compressed image support beyond plain ISO/DD files

## 7.2 Logic Worth Reimplementing Instead of Porting Literally

- `replace_in_token_data()` from `src/parser.c`
- Windows file and path wrappers
- host-locale and registry reads in `src/wue.c`
- direct Win32 registry hive mount logic

The behavior is worth preserving, but the implementation should be Linux-native.

## 7.3 Do Not Port Directly

- `src/rufus.c`
- `src/ui.c`
- `src/stdlg.c`
- `src/drive.c`
- `src/format.c`
- `src/winio.h`

These are deeply coupled to Win32 and should be treated as design references only.

## 8. Filesystem and Tooling Strategy on Linux

Recommended initial dependency set:

- Qt6
- `libparted` or `sfdisk`/`sgdisk` for partitioning
- `wimlib`
- `ntfs-3g` tools (`mkntfs`, `ntfslabel`, `ntfsfix` only as a limited repair aid)
- `dosfstools`
- `exfatprogs`
- `e2fsprogs`
- `util-linux`
- `lsblk` / `udevadm`
- `polkit`
- `hivex`

Recommended format strategy:

- FAT32: use `mkfs.fat`
- NTFS: use `mkntfs`
- exFAT: use `mkfs.exfat`
- ext2/3/4: later, only if explicitly needed

Recommended device discovery strategy:

- do not scan `/sys/block` by hardcoded major numbers
- prefer `lsblk --json` plus udev attributes
- preserve stable identifiers and transport metadata

## 9. Write Modes

## 9.1 Raw/DD Mode

Use when:

- image is explicitly raw/disk image
- ISO should be written byte-for-byte
- media analysis concludes that extraction mode is unsafe or unsupported

Implementation:

- stream input image to block device
- show progress and estimated remaining time
- verify written prefix or full content where feasible

## 9.2 Extraction Mode

Use when:

- ISO can be safely extracted onto a prepared filesystem
- Rufus-style boot and config patch logic is needed

Implementation:

- analyze ISO/UDF with ported core logic
- choose filesystem and partition plan
- prepare target device
- extract via ported `src/iso.c` logic rather than kernel-mount-and-copy
- patch config files
- write/patch boot components
- apply Windows customization if selected

## 10. Windows Customization Plan

This is one of the strongest Linux-port targets because most of the value lies in media editing, not in Windows-host APIs.

## 10.1 Features to Target First

First-wave Windows customization features:

- bypass Microsoft account requirement
- create local administrator account
- disable data collection defaults
- TPM / Secure Boot / RAM bypass
- silent install
- disable BitLocker defaults
- set internal drives offline
- optional S Mode enforcement
- selected QoL tweaks

These correspond to the flags in `src/rufus.h` and the XML generation logic in `src/wue.c`.

## 10.2 Implementation Strategy

### Step A: Build a Linux-native unattend generator

Port the logic of:

- `CreateUnattendXml()` in `src/wue.c`

Output:

- `Autounattend.xml`
- or `unattend.xml` destined for `Panther`

Refactor the current code so it takes explicit inputs:

- Windows architecture
- selected customization flags
- local username
- optional locale/timezone override
- image index

Do not depend on host Windows registry or host Windows locale APIs.

### Step B: Build a WIM patch pipeline

Use:

- `wimlib` to open, extract, update, and overwrite `boot.wim`
- `hivex` to edit the extracted offline `SYSTEM` hive

This will enable:

- direct `LabConfig` edits for TPM / Secure Boot / RAM bypass
- injection of `Autounattend.xml`
- future direct Windows installer media edits

### Step C: Add media file patching

Add support for:

- `sources/appraiserres.dll` placeholder handling
- future `setup.exe` wrapper logic
- CA 2023 bootloader replacement path

This should be modeled as explicit media patch steps in the write plan.

## 10.3 Windows Customization Phases

Phase W1:

- unattend generator only
- local account
- no-online-account
- privacy and BitLocker defaults

Phase W2:

- `boot.wim` patching with `wimlib`
- TPM / Secure Boot / RAM bypass using offline hive edits

Phase W3:

- `appraiserres.dll` and setup wrapper support
- CA 2023 bootloader replacements
- SkuSiPolicy support

Phase W4:

- revisit Windows To Go

## 11. Detailed Delivery Phases

## Phase 0: Project Restructure

Deliverables:

- create new module boundaries
- isolate existing UI from backend
- add backend interfaces
- add logging/progress abstractions

Acceptance criteria:

- app builds with UI intact
- no direct calls from UI into legacy `lufus/linux/*`

## Phase 1: Device and Image Analysis MVP

Deliverables:

- Linux device enumeration service
- image open/analyze service
- ported image report model
- initial ISO scan metadata

Acceptance criteria:

- app lists removable targets correctly
- app scans ISO and shows bootability, label, Windows/Linux detection, and likely write mode

## Phase 2: Raw Write Path

Deliverables:

- block device raw writer
- progress reporting
- cancellation and cleanup

Acceptance criteria:

- write a known-good raw image to USB
- verify that written media boots when the source image is known-good

## Phase 3: Extraction Mode Core

Deliverables:

- portable extraction engine from upstream Rufus logic
- target preparation for FAT32
- config patching framework

Acceptance criteria:

- Linux ISOs that need config rewriting boot from extracted media
- extracted media no longer relies on kernel-mounting the ISO

## Phase 4: NTFS and Windows Install Media

Deliverables:

- NTFS formatting using `mkntfs`
- WIM split support
- Windows install extraction flow
- basic Windows install USB creation

Acceptance criteria:

- Windows ISO with large `install.wim` is handled correctly
- created NTFS or split-WIM media behaves as expected

## Phase 5: Windows Customization

Deliverables:

- unattend generator
- local account creation support
- online-account bypass
- TPM / Secure Boot / RAM bypass

Acceptance criteria:

- generated media applies selected customizations during Windows setup

## Phase 6: Bootloader and Compatibility Work

Deliverables:

- Syslinux integration
- BIOS/MBR support improvements
- additional GRUB/Syslinux workarounds
- compressed image support

Acceptance criteria:

- broader set of Linux images boot successfully

## Phase 7: Security, Hardening, and Packaging

Deliverables:

- privileged helper
- cleanup guarantees
- packaging
- installable desktop app

Acceptance criteria:

- GUI can be used without running the whole app as root
- failure paths leave the system and devices in a sane state

## 12. Testing Strategy

## 12.1 Automated Tests

Add tests for:

- image report generation
- ISO metadata recognition
- config patch transformations
- WIM split planning
- unattend XML generation
- Windows customization flag mapping

## 12.2 Integration Tests

Use loop devices and fixture images for:

- partition creation
- FAT32 / NTFS formatting
- raw write path
- extraction mode
- WIM patching

## 12.3 Boot Validation Matrix

Validate at least:

- common Ubuntu ISO
- Debian live ISO
- Fedora ISO
- Arch-based ISO
- Windows 10 ISO
- Windows 11 ISO with and without customization

Validate on:

- BIOS-only VM
- UEFI VM
- Secure Boot UEFI VM where relevant

## 13. Security and Reliability Requirements

Must-have requirements:

- GUI does not require full root execution
- all destructive operations require explicit confirmation
- helper commands are structured and validated
- cleanup always runs on cancellation and failure
- temporary directories are uniquely created and removed
- no unsafe use of uninitialized descriptors or handles

## 14. Risks

Major risks:

- underestimating how much of `src/iso.c` is entangled with Win32 helpers
- bootloader behavior differences across Linux distributions
- NTFS boot corner cases
- Secure Boot / CA 2023 edge cases
- overfitting to a few test ISOs
- privilege escalation design growing too late in the project

Mitigations:

- port the core incrementally
- test continuously with a media matrix
- isolate platform-specific code early
- keep feature flags behind capability checks

## 15. Recommended Immediate Next Steps

1. Freeze the current `lufus` backend as reference only.
2. Introduce backend interfaces and route the UI through them.
3. Port the image report model from `src/rufus.h`.
4. Start extracting scan-only logic from `src/iso.c`.
5. Build a Linux device enumeration service using `lsblk` and udev metadata.
6. Implement raw write mode before extraction mode.
7. Build the unattend generator early, because it is high-value and relatively self-contained.
8. Add `wimlib` and `hivex` exploration spikes for `boot.wim` patching.

## 16. Recommended File Layout

Suggested medium-term layout under `lufus/`:

- `ui/`
- `app/`
- `core/`
- `platform/linux/`
- `services/`
- `resources/`
- `tests/unit/`
- `tests/integration/`
- `plans/`

Example:

- `lufus/core/image/`
- `lufus/core/windows/`
- `lufus/platform/linux/devices/`
- `lufus/platform/linux/storage/`
- `lufus/platform/linux/privileged_helper/`

## 17. Final Recommendation

The best path is not to "finish the current `lufus` backend".

The best path is:

- keep `lufus` as the user-facing shell,
- port the portable intelligence from real Rufus,
- rebuild the backend as a Linux-native service stack,
- prioritize raw write mode, extraction mode, NTFS Windows media support, and Windows customization in that order.

This yields a product that is both recognizably Rufus-like and realistically maintainable on Linux.
