# Rufus Linux / Qt6: reuse assessment

Reviewed on 2026-09-13. This assessment updates the direction in
`rufus-on-linux-implementation-plan.md`: reuse the current Rufus source tree as
the foundation, use Linux services and libraries for platform work, and create
a new Qt6 frontend. Keep Lufus as a reference for workflow and earlier lessons.
The review below records the original source assessment. The subsequent
implementation adds an independent reporting component under `rufus/linux/`;
the existing Windows and Lufus application sources remain unchanged.

## Repository setup

- Fork: https://github.com/mcirsta/rufus (verified parent: `pbatard/rufus`).
- Local checkout: `/home/marius/proj/lufus/rufus`.
- `origin`: `https://github.com/mcirsta/rufus.git`.
- `upstream`: `https://github.com/pbatard/rufus.git`.
- Rufus baseline: `18ae93bf37f94764d77216cff4168e0549a14c4d`.
- Lufus baseline: `22af33084db9e8417804dd93ff0ad65fab908fb4`.
- Rufus is a full-history clone, with `master` matching both remote master
  branches at review time. The original source tree is unchanged.
- Current work is on local branch `codex/linux-qt6-reporting`, with additive
  Linux reporting code and tests under `rufus/linux/`.

The nested checkout is an independent repository, not a submodule. Run future
Rufus git commands inside it; do not add the directory to Lufus as an embedded
repository. The layout below describes the intended port. The independent
CMake entry point, reporting job, C ABI adapter and Widgets integration tests
now exist; the full application and disk backend remain future work.

## Recommendation for Lufus

Lufus already targets Qt6 Widgets. It configured and built successfully with
Qt 6.11.2, GCC 16.2.1 and libparted 3.7 on this machine. Its limitations are in
the application architecture and incomplete behavior, rather than Qt6 build
compatibility. The project is small enough that retaining its tightly coupled
controller and backend would probably cost more than replacing them.

| Existing part | Evidence | Reuse decision |
| --- | --- | --- |
| Main window and Designer forms | `ui/rufuswindow.ui` presents the familiar device, image, filesystem, partition and progress workflow. Some controls have no implementation behind them. | Keep the useful workflow and wording as design references. Build a fresh frontend. |
| Window controller | `ui/rufuswindow.cpp:79` starts jobs directly; `iso_path` is not initialized in the constructor; Start does not prevent concurrent jobs. `sourceCombo` is populated but never used to select a write mode. | Replace with an explicit job controller and typed models. |
| Background worker | `rufusworker.cpp:47` declares uninitialized descriptors; both error and scan cleanup use the full descriptor set. It keeps pointers into mutable window data, always calls `format_fat32`, and hardcodes the label to `GALA`. | Replace orchestration and ownership. A thread is still useful, but this worker is not a reusable engine. |
| Logging and progress | `ui/log.cpp:23` correctly recognizes the need to signal the GUI, but depends on global `logptr`, transferred `char*` buffers and shared progress state. | Replace with typed Qt signals from a job object. Keep only a small C ABI adapter where reused C code requires it. |
| Device discovery | `linux/devices.c:23` identifies candidates using the first character of the major number and the removable bit. Device numbers are narrowed to eight bits. `DeviceComboBox` rescans synchronously whenever opened. | Replace with UDisks2 device objects and change notifications through QtDBus. |
| Mounting and copying | `linux/mounting.c:241` cleans global paths and descriptors unconditionally; loop creation hardcodes device number `(7,0)`. Copy traversal results are ignored and some failure paths do not reach the worker correctly. | Replace with library-based image access and platform services for the target. |
| Partitioning and FAT32 formatting | `linux/partition.c` manually orchestrates libparted; `linux/fat32.c` constructs filesystem metadata itself with 512-byte assumptions. | Reuse mature Linux partitioning and filesystem tools/services instead of maintaining another formatter. |
| ISO recognition | `iso.c` mostly collects EFI and large-file flags from a mounted tree. Many named boot markers are unused; the report is not a working equivalent of current Rufus image analysis. | Use current Rufus analysis and media rules as the source of truth. |
| Build setup | `CMakeLists.txt` already finds Qt6 Widgets and enables AUTOMOC/AUTOUIC; the README still describes qmake and Qt5. | Keep CMake/Ninja, but create independent targets in the Rufus fork. |

These are source-review findings, not reproduced disk-operation failures. No
image was mounted and no device was formatted or written during this review.

## Modern Qt6 frontend

Use **Qt6 Widgets with C++ models/controllers**, following the user's desktop
UI preference. Rufus has a small, conventional dialog-style interface: its
Win32 edit fields, combo boxes, checkboxes, progress bars and dialogs map
naturally to standard widgets. Use layouts and the desktop palette for modern
scaling and theme behavior. QML, mobile layouts and animated transitions are
outside this design. [Qt Widgets](https://doc.qt.io/qt-6/qtwidgets-index.html).

Designer `.ui` forms remain an option. Reuse individual Lufus layouts or controls
where useful, while keeping its old worker and global logger out of the new
application. Widgets does not require retaining those architectural problems.

The new frontend should have a compact main window: image selection with an
analysis summary, a device selector showing identity and capacity, appropriate
defaults for the selected image, and expandable advanced options. Present a
concrete erase/write summary before starting. Show phase, bytes, speed and a
log, with clear cancellation and completion states. Unsupported options should
not look operational.

Keep disk work out of widget classes. Use C++ value types for image reports
and write plans, a `QAbstractListModel` for devices, and a controller with explicit states such
as analyzing, ready, preparing, writing, verifying and finishing. Job input must
be an immutable snapshot; a device disappearing must invalidate the selection.
Use queued signals for progress and completion. Theme palettes, scalable
layouts, translations and keyboard navigation should be part of the first UI.

### C integration and job notifications

Use a C++ `QObject` job for blocking library work, moved to a worker thread,
with typed signals such as `progressChanged`, `logMessage` and `finished`.
Connect these to the GUI controller/model with queued connections. Convert
borrowed C strings to owned `QString` values before queuing them. The UI model
owns presentation state; C code does not receive widget pointers.
[Qt worker objects and queued connections](https://doc.qt.io/qt-6/qthread.html#details).

The C-to-C++ boundary and the thread boundary are separate. C++ calls C APIs
directly. When a C library reports progress using a function pointer plus an
opaque context, a small adapter forwards that notification as a Qt signal.
Where unchanged Rufus files already call named logging/progress functions,
the Linux build can instead supply matching C-linkage implementations in C++.
This avoids adding a new callback framework to upstream. Calls without a
context still require an explicit routing/lifetime decision, such as a scoped
sink for one serialized job; do not hide that limitation behind a global GUI
pointer or assume the legacy core is reentrant.

The adapter must remain alive until the operation and any library callbacks
have stopped. A blocking worker cannot process a queued cancel slot while
inside a synchronous C call: use cooperative interruption checked by the
library callback or work loop. Throttle progress notifications. Libraries
without suitable callbacks need an operation-specific solution, not a claim
that Qt signals automatically make them cancellable. Existing asynchronous
D-Bus and process APIs can use their own Qt notifications directly.

## What Rufus can actually share

The distinction is between reusing existing source files, sharing logic after
small structural changes, and replacing Windows services with Linux services.
Do not describe all three as already portable.

An object-compilation probe used unchanged files from the baseline with the
host C compiler and `-std=gnu11`. It did not link an application, run parsers,
install bootloaders, or demonstrate boot compatibility.

| Rufus component | Observed portability | Direction |
| --- | --- | --- |
| `src/cregex_compile.c`, `cregex_parse.c`, `cregex_vm.c` | All three compiled unchanged. | Compile the originals when shared media/config logic needs them. |
| `src/syslinux/libinstaller/{fs,setadv,syslxmod}.c` | All three compiled unchanged, with packed-member alignment warnings. Logging and bootloader data still need integration. | Reuse source and supply the necessary callbacks/data; verify architecture assumptions before claiming support beyond the tested host. |
| `src/syslinux/libfat/` | Four files compiled directly. `cache.c` lacked declarations for `_mm_malloc`/`_mm_free`; it compiled unchanged when the host compiler included `mm_malloc.h`. | Existing read callbacks are a useful boundary. The forced include is only a host probe; choose a portable allocation solution for supported architectures. |
| `src/ms-sys/{fat12,fat16,fat32,ntfs,partition_info,br}.c` | All six compiled unchanged. `file.c` instead uses Windows handles and Rufus I/O. | Strong candidate for build-time replacement of `file.c`, preserving the existing boot-sector routines and data. Respect its `FILE*`/`FAKE_FD` contract consistently across Linux callers. |
| `src/iso.c` and report/decision definitions in `src/rufus.h` | `iso.c` directly includes Windows headers, uses Win32 file operations, global state and UI callbacks. `rufus.h` itself includes `windows.h`. | Preserve media recognition and patching rules; first identify a small separable unit and its I/O/reporting boundary. Whole-file unchanged Linux reuse is not established. |
| `src/parser.c`, `src/xml.c`, `src/wue.c` | Even apparently generic helpers include Rufus/Windows headers. Windows customization also reads host settings and manipulates images. | Share useful algorithms through small explicit boundaries where practical; replace host services and provide explicit configuration inputs. |
| `src/vhd.c` and `src/wimlib/` | Rufus already calls wimlib for WIM splitting. Its bundled build selects Windows sources and defines `_RUFUS` integrations. | Prefer the Linux wimlib library while retaining Rufus's media decisions. Audit local library modifications before assuming equivalence. |
| `src/libcdio/` and `src/bled/` | Rufus carries its own versions and build configuration. Full Linux portability was not tested. | Compare required behavior and Rufus changes before choosing the bundled source or system libraries. Avoid replacing image behavior merely because another library has a similar API. |
| Win32 UI, discovery and formatting | `src/rufus.c`, `ui.c`, `dev.c`, `drive.c`, `format.c` and related modules are Windows implementations, with some product policy mixed in. | Leave these implementations in place. Recover needed policy separately and use Linux services for the platform operations. |

In total, **16 translation units compiled directly**, and a seventeenth compiled
with a compiler-supplied allocation header. This supports selective reuse, not
a claim that the Rufus engine already builds on Linux.

## Build-time separation with minimal upstream disruption

Start with additions under a Linux directory and an independent CMake entry
point. The existing Visual Studio and MinGW builds should not acquire Qt,
Linux dependencies or a new build-system requirement.

```text
rufus/
  src/                       existing Rufus sources and bundled libraries
  res/                       existing resources and boot assets
  linux/
    CMakeLists.txt           independent Linux entry point
    cmake/                  explicit lists of reusable upstream source files
    qt/                     Widgets frontend, C++ jobs, models and controllers
    platform/               UDisks2, library/tool and I/O adapters
    tests/                  ordinary image-file fixtures and integration tests
```

Linux targets should compile reusable `../src/...` files directly. For example,
select the six existing `ms-sys` algorithm files plus a Linux I/O implementation;
the Windows build continues selecting the original `ms-sys/file.c`. No platform
conditionals are needed inside those algorithm files. Keep Qt at the frontend
boundary and expose plain C interfaces/callbacks to reused C code.

For tightly mixed modules such as `iso.c`, build selection alone cannot replace
individual Win32 calls or static/global UI dependencies. A small neutral header
and a mechanical extraction may be the best long-term solution. That would be
an explicit, separately reviewed exception to the untouched-source preference.
Do not introduce a broad Win32 emulation layer, duplicate the whole ISO engine,
or generate edited copies using text replacement to pretend the boundary is
free. A shadow `rufus.h` on an include path also cannot reliably replace the
quoted same-directory include in the original source.

Keep initial work additive. Propose any unavoidable shared-code change as a
small behavior-preserving patch with Windows verification. Maintain the Linux
build, tests, packaging and dependencies in the fork; upstream acceptance is a
future maintainer decision. Preserve provenance and existing license notices
when reusing code and assets.

## Linux services before new implementations

| Operation | Proposed implementation |
| --- | --- |
| Device discovery and hotplug | UDisks2 over QtDBus, including identity, transport, size and mount state. |
| Authorization, mount/unmount and opening a write target | Use UDisks2's existing service and authorization model where it covers the operation. Its Block interface includes `OpenForRestore` and formatting methods. |
| Partition layout | Assess UDisks2 partition APIs first; use libfdisk or a controlled `sfdisk` invocation for layouts/options it cannot express. Keep one selected implementation for each supported operation initially. |
| Filesystem creation | UDisks2-backed formatting or established `mkfs.fat`, `mkntfs`, and `mkfs.exfat` tools. Preserve Rufus-specific boot record work separately from filesystem creation. |
| WIM inspection/splitting/updating | Linux wimlib; retain Rufus rules for choosing operations and parameters. |
| ISO/UDF reading and extraction | Rufus's existing libcdio-based approach, subject to the bundled-versus-system audit. Preserve its media-specific decisions. |
| Raw writing | A small job implementation using an authorized descriptor, correct partial-I/O handling, progress, cancellation, flush and verification. Avoid introducing an independent disk I/O framework. |
| Operations not covered by existing services | Add a narrowly scoped privileged helper only after identifying a concrete gap. The GUI stays unprivileged. |

The UDisks2 API already supplies disk, filesystem, partition and job interfaces;
using it can reduce custom privileged code.
[UDisks2 API](https://storaged.org/doc/udisks2-api/latest/),
[Block interface](https://storaged.org/doc/udisks2-api/latest/gdbus-org.freedesktop.UDisks2.Block.html).
Linux wimlib supports image inspection and modification through its library and
CLI. [wimlib](https://github.com/ebiggers/wimlib).
Use established filesystem and partitioning projects as dependencies:
[dosfstools](https://github.com/dosfstools/dosfstools),
[exfatprogs](https://github.com/exfatprogs/exfatprogs),
[util-linux](https://github.com/util-linux/util-linux).

Existing services still need application glue: stable target validation,
single-job ownership, dependency/capability checks, state transitions, error
propagation and cancellation. That is necessary port code; it should not expand
into replacements for existing filesystem, image or partition libraries.

## First implementation milestones

1. Add the independent Qt6 frontend, device model and job controller. Use
   read-only discovery and simulated job progress to verify the UI states.
2. Build selected unchanged Rufus modules as Linux targets. Add one useful
   image-analysis path that proves preservation of Rufus behavior; this should
   happen early so the project does not become only a generic raw writer.
3. Add a real raw-image job with verification against ordinary temporary image
   files first, then explicit opt-in integration tests for disposable devices.
4. Add FAT32/UEFI extraction with Rufus media analysis and Linux services,
   including large-WIM handling. Test resulting images in virtual machines.
5. Add NTFS/UEFI assets, BIOS/Syslinux, more image quirks and Windows customization
   incrementally. Expand the supported-media matrix only with boot tests.

For each reused component, record its upstream revision, whether its source is
unchanged, required adapters, and behavioral tests. For structural sharing
patches, run Windows checks as well as Linux checks. Avoid implementing every
possible platform backend or feature before the first useful shared path works.

## Verification performed

Lufus configured and built successfully, using Ninja's default parallelism:

```sh
cmake -S /home/marius/proj/lufus -B /tmp/lufus-qt6-assessment -G Ninja -DCMAKE_BUILD_TYPE=Debug
cmake --build /tmp/lufus-qt6-assessment
```

Rufus object-compilation logs and objects are under
`/tmp/rufus-reuse-assessment.kNtjIu`. Each file was compiled with `cc -std=gnu11
-c`, the relevant library include directory and an output path there. The
additional `cache.c` probe used `-include mm_malloc.h`. No source edits were
needed for these probes. Packed-member warnings remain in Syslinux; no runtime,
cross-architecture, boot or linked Linux Rufus validation has been performed.

The subsequent reporting implementation configured and built successfully with
the installed Qt 6.11.2. Its CTest suite passed all seven behavioral test cases:
C-to-Widgets delivery and message ownership; progress control values and
64-bit counters; coalescing and failure/completion order; serialization; phase
initialization order; nested and repeated execution; and receiver destruction.
Build instructions and integration contracts are in `rufus/linux/README.md`.
These are runtime checks of the reporting component, not full Rufus media or
boot validation.
