# Raspberry Pi RDKE Wrynose Migration Review

Last updated: 2026-11-17

This is the cross-repository review tracker for the Raspberry Pi RDKE migration
to Yocto Wrynose. The layers below are separate Git repositories, so PRs should
be prepared per repository. Keep this inventory updated as build triage
continues.

## Current status (2026-11-17)

- The user confirmed both `core-image-minimal` and
  `lib32-core-image-minimal` built successfully.
- The latest `lib32-vendor-test-image` build failed fetching mpg123 after the
  vendor PV pin to 1.29.3 retained OE-Core's 1.33.4 checksum. The upstream
  1.29.3 archive downloaded successfully and its SHA-256 was verified against
  the fetched archive; added a PV-scoped checksum override for 1.29.3. Retry
  `lib32-mpg123:do_fetch` to validate.
- The subsequent `lib32-mpg123:do_compile` failed because mpg123 1.29.3 uses
  old-style signal-handler function pointer types incompatible with POSIX
  `sigaction`. Added a version-scoped patch aligning the wrapper and callbacks
  with `void (*)(int)`; the patch dry-run applies cleanly. Retry
  `lib32-mpg123:do_compile` to validate.
- The latest `lib32-essos:do_compile` failed because 32-bit `_TIME_BITS=64`
  removes the `time` member from `struct input_event` in Wrynose's Linux input
  headers. Updated Essos to use the header's `input_event_sec` and
  `input_event_usec` compatibility macros. The patch dry-run applies to Essos
  2.1.2 and both macros are present in its recipe sysroot; retry compile.
- The latest `lib32-assimp:do_configure` failed because CMake 4 rejects
  Assimp 5.0.1's top-level minimum version of 2.8. Added
  `CMAKE_POLICY_VERSION_MINIMUM=3.5` to the recipe's CMake arguments, following
  the diagnostic's suggested compatibility setting; retry configure.
- The latest `lib32-westeros:do_configure` failed because the recipe formed
  its source directory from `${PN}`, which expands to `lib32-westeros` for
  multilib. Changed `S` to use `${BPN}` so it resolves the unpacked
  `westeros-2.1.4` source directory; retry the configure task to validate.
- The latest `lib32-vendor-test-image` dependency resolution reaches the
  Raspberry Pi Wi-Fi firmware but rejects its `synaptics-killswitch` license
  flag. The product configuration now accepts that flag specifically for the
  RDKE Raspberry Pi build. The subsequent gobject-introspection-native
  configure failed because it found GLib 2.74.6, below its 2.82 minimum. The
  product now prefers OE-Core GLib 2.88.2 for native tools only; retry the
  target to validate. The next run failed unpacking Vulkan headers and loader:
  Wrynose rejects explicit `S = "${UNPACKDIR}/git"` overrides. Removed these
  obsolete assignments from the Vulkan recipes and appends; retry to validate.
- The latest `lib32-vendor-test-image` run failed Breakpad patch-fuzz QA and
  Mesa fetch. Refreshed the Breakpad patch to remove redundant descriptor
  closes and test-only code, and added the verified Mesa 22.0.3 checksum for
  the vendor version pin; retry both tasks to validate.
- The next Mesa run failed applying OE-Core's Freedreno build-path patch:
  its hunk targets a newer Mesa source layout than the vendor-pinned 22.0.3.
  The XD4 reference carries the same newer patch; no matching BPI counterpart
  was found in the checked layers. Adapted the shared patch with version-neutral
  context, preserving the absolute path for build-time generators while
  removing it from runtime `RNN_PATH`. A dry-run against pristine Mesa 22.0.3
  applies without fuzz. Also replaced the deprecated `FILES:${PN}:remove +=`
  assignments; retry `lib32-mesa:do_patch` and the image build to validate.
- The subsequent `spirv-headers-native:do_configure` failure came from the
  shared recipe include setting `S` with `${PN}`, which adds `-native` even
  though unpack names the source directory with `${BPN}`. Updated the include
  to use `${BPN}`; retry the image build to validate.
- The subsequent `rust-llvm-native:do_patch` failure could not resolve the
  `llvm-config.cpp` target for the Yocto environment override patch. Normalized
  the patch paths relative to the recipe's LLVM source directory and set its
  strip level to `1`; retry to validate.
- The latest build failed `fontconfig:do_patch` during patch QA because the
  cache-mtime patches used invalid `Upstream-Status: Appropriate` metadata.
  Changed both to `Pending`. `glslang-native:do_compile` also exposed
  a Vulkan SDK mismatch: glslang 1.4.341.0 compiled against SPIRV-Tools 1.3.260.
  Prefer matching SPIRV-Headers and SPIRV-Tools 1.4.341.0 for Wrynose, both
  native and target; retry the tasks to validate.
- The latest `dbus-native:do_install` failed because the shared dbus append
  unconditionally edits `${sysconfdir}/dbus-1/system.conf`, which native D-Bus
  does not install at that path. Restricted the system-config edit to the
  target variant. The XD4 reference has the same unscoped edit; consider
  carrying this class-specific fix there too.
- The latest build exposed two remaining source-path mismatches. SPIR-V Tools
  used `${PN}` and incorrectly included `-native` in its unpack directory;
  switched both SPIR-V shared includes to `${BPN}`. Rust 1.72 still pointed at
  `${WORKDIR}` instead of Wrynose's `${UNPACKDIR}`. Updated `RUSTSRC` and
  retained the LLVM patch's corrected relative path/strip level. Verified the
  Rust patch applies to the actual unpacked LLVM source without fuzz; retry the
  two failed tasks and image build.
- The next Mesa 22.0.3 patch failure was the LLVM 22 Gallivm iterator fix:
  upstream's hunk uses a qualified type, while this older source uses the
  unqualified `StringMapIterator`. Added a version-specific adaptation using
  `auto`, and made the Mesa 22.0.3 append replace both incompatible upstream
  patch hunks. The adapted Gallivm patch applies without fuzz; retry
  `lib32-mesa:do_patch` and the image build.
- The Mesa retry still failed because the vendor append's `SRC_URI:remove`
  spelled the OE-Core patch filename with an apostrophe (`don\'t`), while the
  actual filename uses a hyphen (`don-t`). The generic patch was therefore
  applied before its Mesa 22.0.3 replacement. Corrected the removal name;
  clean `lib32-mesa`'s work directory before retrying to clear the partial
  patch state. Fontconfig 2.17.1 also exposed stale context
  in the cache-mtime patch. Added a version-specific refreshed patch, retained
  the legacy patch for other Fontconfig versions, and fixed the `FILES:remove`
  operator warning.
- Source changes are committed independently on `feature/rdke-wrynose` in 11
  affected repositories. All 11 feature branches are now pushed: five to
  `rdkcentral` and six to the user's `h1712` forks.
- The top-level Raspberry Pi manifest and both common/config submanifests are
  also updated and pushed on `feature/rdke-wrynose`, with pins for all 11
  changed projects.
- The latest image build exposed three new issues. PulseAudio was pinned to
  15.0 while inheriting the 17.0 recipe checksum; verified the official 15.0
  archive and added its checksum conditionally. Removed the duplicate
  `wayland-protocols-dev` package declaration while retaining its pkg-config
  file assignment. Mesa 22.0.3's bundled C11 threads header also conflicted
  with glibc 2.43's `once_flag` declarations; added a Mesa-version-scoped
  compatibility patch. Rerun the affected fetch, package, and compile tasks
  to validate.
- The new `lib32-pulseaudio:do_package` failure had two causes: PulseAudio's
  modules install under a versioned `pulse-${PV major.minor}/modules`
  directory, while the recipe split packages from the unversioned
  `pulseaudio/modules` path; and the ALSA mixer data FILES rule only named the
  directory. Updated module splitting to use the versioned directory and
  matched the mixer path/profile files. The existing conditional GSettings
  package already claims its helper/schema files. Confirmed the configured
  module and data patterns match the installed files; rerun `do_package` to
  validate BitBake packaging.
- The new ALSA plugins fetch failure is specific to the vendor-pinned 1.2.6
  archive. Verified the upstream download SHA-256 is
  `068818a4b55d8c029daa00015d853d45113f56b224b7c64e1e117988c825b2a0`; made
  the checksum conditional on PV while preserving the OE-Core 1.2.12 checksum.
  FFmpeg's 4.2.2 append also listed `avutil` as a PACKAGECONFIG even though it
  is always built and has no PACKAGECONFIG declaration; removed that invalid
  token. Rerun the ALSA fetch and FFmpeg configure tasks.
- The latest build also failed compiling the pinned BlueZ 5.48 vendor restore
  patch because it called `sendfile()` without including its declaration;
  added `<sys/sendfile.h>`. `cargo-native` failed its Rust compiler probe
  because the native Rust target spec (`x86_64-linux.json`) is staged under
  `${STAGING_LIBDIR_NATIVE}/rustlib`, outside Cargo's task-local target search
  path. Added that directory to `RUST_TARGET_PATH` for native Cargo builds;
  rerun both compile tasks to validate.
- The subsequent BlueZ 5.48 compile passed `src/shared/shell.c` after selecting
  Readline's stdarg declaration but failed at the same `rl_message()` call in
  `tools/btmgmt.c`. Extended the BlueZ compatibility patch to select the
  variadic declaration before that file's Readline include too; rerun compile.
- The latest build failed `lib32-libstd-rs:do_compile` because the staged
  `cargo-native` binary needs `libssh2.so.1`, which is present in the native
  recipe sysroot but not on the runtime loader path. Added the native library
  directory to the libstd compile environment. `lib32-cairo:do_package` also
  failed the no-GPLv3 policy because Cairo's default `trace` option packages
  GPLv3 tools and marks related packages incompatible; disabled tracing,
  removed its now-unused perf-utils package, and assigned the remaining
  staticdev/locale packages their actual permissive Cairo license. Rerun both
  tasks to validate.
- The subsequent libstd compile reached Cargo but its rustc target probe could
  not find `x86_64-linux.json`: `rust.inc` sets `RUST_TARGET_PATH` to the
  recipe-local targets directory, overriding the native spec path used by
  cargo-native. Appended `${STAGING_LIBDIR_NATIVE}/rustlib` for libstd builds
  so Cargo/rustc can resolve the staged host target spec; rerun compile.
- The newest compile still reports rustc cannot resolve `x86_64-linux` despite
  the target spec directory being present in `RUST_TARGET_PATH`. The spec file
  and direct rustc probe both work in the mirrored sysroot; Cargo's rustc probe
  also succeeds when rustc is launched through the uninative loader, so the
  remaining difference is the configured task's child-process environment or
  loader. The Cargo class now logs the selected `rustc` path to help compare the
  next task run; avoid adding another target-path workaround until the child
  environment is identified. BlueZ's `uhid_destroy()` forward declaration and
  file-local helper are verified by a zero-fuzz dry-run against the preserved
  pre-patch source.
- The new `cargo-c-native:do_compile` failure is caused by `cargo` loading
  `libssh2.so.1` from its native recipe sysroot without that sysroot's library
  directory on `LD_LIBRARY_PATH`. Added a native library-path prepend to the
  shared Cargo build function, removing the duplicate libstd-specific setup.
  Verified staged Cargo starts through the uninative loader with this path.
  Rerun both `cargo-c-native:do_compile` and `lib32-libstd-rs:do_compile`;
  libstd's target-spec lookup remains unresolved.
- The latest `lib32-nghttp2:do_configure` failure comes from nghttp2 1.48.0's
  old CMake minimum-version declaration, rejected by CMake 4; set
  `CMAKE_POLICY_VERSION_MINIMUM=3.5` in that version's recipe. Cairo's
  `do_package` failed because multilib script renaming still expected
  `cairo-trace` after the GPLv3 trace option was disabled and the executable
  removed. The prior package removal used `${PN}-perf-utils`, but the recipe
  declares the package literally as `cairo-perf-utils`; remove that literal
  package so multilib cannot rename or package it. The Assimp unpack warning
  also identifies an `${PN}` multilib path mismatch; use `${BPN}` in `S` to
  match the unpacked `assimp-${PV}` directory. Rerun these tasks to validate.
- The latest libstd compile still cannot find the native `x86_64-linux` target
  spec, and the same host-target probe now fails for `cargo-c-native`. Both
  recipes generate only the canonical `x86_64-unknown-linux-gnu` target in
  their task-local `rust-targets` directory, while the installed rustc reports
  its host as `x86_64-linux`. Cargo first queries rustc for that host without a
  `--target` argument. Stage the native sysroot's `${BUILD_SYS}.json` into the
  task-local target directory in the shared Cargo build function, so the host
  target can be resolved across Cargo recipes; remove the libstd-only path
  append. The generated non-host target JSON also encoded pointer and C-int
  widths as numbers, but Rust 1.72 expects strings for these fields. Preserve
  their string values in `rust-target-config.bbclass`; a direct rustc check
  now resolves the host spec and parses the ARM target spec. Also corrected
  Cairo's package removal to match the literal `cairo-perf-utils` entry and
  changed Assimp's source path to `${BPN}` for multilib. Rerun the affected
  Cargo, Cairo, and Assimp tasks to validate.
- The latest `gstreamer1.0-plugins-base:do_patch` passed patch application but
  failed patch-status QA for the Amlogic Opus clipping behavior patch. Marked
  it `Inappropriate` as vendor-specific; rerun the patch task to validate QA.
- The latest build's two HAL header unpack tasks failed Wrynose QA because
  their recipes explicitly set `S` to `${UNPACKDIR}/git`; removed the obsolete
  assignments and rely on BitBake's default `S`. Cargo advanced past the target
  lookup after adding the staged Rust target-spec directory, but then failed
  because its canonical Rust triple has no staged standard library. Native Rust
  libraries are installed for `${BUILD_SYS}`, so configured native Cargo to
  build and install using that target triple. Rerun these tasks to validate.
- The next build reported the same obsolete `${UNPACKDIR}/git` source override
  in Essos resource manager and iarmbus headers; removed both. BlueZ 5.48 also
  failed compiling calls to readline 5.2's `rl_message()` because that header
  exposes an old-style empty parameter declaration unless its stdarg feature
  macros are selected. Added a version-specific patch to select the variadic
  declaration; rerun the affected tasks to validate.
- The latest `lib32-essos:do_unpack` failed the same Wrynose QA check for an
  explicit `S = "${UNPACKDIR}/git"` assignment. Removed the override and rely
  on BitBake's default unpack source directory; rerun `do_unpack` to validate.
- The latest build found the same obsolete git `S` override in the RDK
  GStreamer utility, HDMI CEC, IARM manager, and Device Settings header
  recipes; removed those assignments. Assimp's miniz update patch was
  truncated at its final hunk, which produced patch fuzz; restored the complete
  upstream patch and verified it applies with zero fuzz. Cairo now fails
  installed-vs-shipped QA because `libcairo-fdr.so` remains after removing
  cairo-perf-utils; remove that unused module along with cairo-trace when trace
  is disabled.
- The latest `cargo-c-native` build passed Cargo's host-target probe and lock
  parsing, then failed because `cargo-c` 0.10.21 embeds Cargo 0.95 and declares
  Rust 1.92 as its minimum; its locked `time-core` also uses the Rust 2024
  edition. An initial fallback to `cargo-c` 0.9.30 + Cargo 0.77 failed because
  Cargo 0.77 requires rustc 1.73. Replaced that with `cargo-c` 0.9.27 + Cargo
  0.74, whose crate metadata declares Rust 1.70, regenerated the crate include
  from its version-3 lockfile, and retargeted the riscv32 patch to
  `parking_lot_core` 0.9.8. Verified the release checksum and patch
  applicability. The repeated `lib32-libstd-rs` log shows the target-spec file
  is staged but Cargo's rustc probe still cannot resolve
  `x86_64-linux`. Initially attempted to force `RUST_TARGET_PATH` using Cargo's
  CLI config option, but the latest native cargo log shows inline tables are
  not accepted there. Moved the forced environment setting into Cargo's
  generated `config.toml`, where the `[env]` table syntax is supported; rerun
  cargo-native and libstd to validate.
- The latest `lib32-libstd-rs:do_install` failed after compilation succeeded:
  the install function looked under `${B}/${TARGET_SYS}`, but Cargo writes
  under `${B}/${CARGO_TARGET_SUBDIR}`. Updated the cleanup and copy paths to
  use the shared Cargo target subdirectory. Confirmed it contains `libstd.rlib`
  and that the previous target path does not exist; rerun do_install to validate.
- The latest `rust-native:do_compile` failure is caused by duplicate
  `[target.x86_64-unknown-linux-gnu]` tables in Cargo's legacy config. Rust's
  setup appended its target table on each invocation, so changed it to
  regenerate the config file before writing the linker entry. Rerun
  `rust-native:do_compile` to validate.
- The latest four failures are narrowed to three compatibility fixes and one
  patch refresh. `westeros-sink:do_patch` still targeted `${UNPACKDIR}/git`
  because a vendor dev-layer append overrode `S`; corrected that append to use
  `${UNPACKDIR}/${BPN}-${PV}`. Refreshed the preferred-protection GStreamer
  patch against its exact pre-patch source; its three hunks now apply without
  fuzz. `cargo-c-native` was targeting
  `x86_64-unknown-linux-gnu`, but this Rust 1.72 sysroot provides the native
  standard library and target spec under `${BUILD_SYS}` (`x86_64-linux`).
  Changed its native Cargo target and output subdirectory to `${BUILD_SYS}`.
  Assimp 5.0.1 also needed `<cstdint>` enabled for the fixed-width types in
  `FIReader.hpp`; the new patch applies to the preserved source. Rerun the
  Westeros-sink patch, GStreamer patch, cargo-c compile, and Assimp compile to
  validate in BitBake.
- The subsequent `lib32-librsvg:do_configure` failed when Meson invoked
  `cargo --version`: Cargo could not load `libssh2.so.1` from the recipe's
  native sysroot. Added `${STAGING_LIBDIR_NATIVE}` to the exported loader path
  in `cargo_common.bbclass`, covering direct Cargo probes during configure as
  well as Cargo-dependent build tasks; rerun configure to validate.
- The latest `lib32-vkmark:do_patch` failure came from its `S` path using
  `${PN}`, which expands to `lib32-vkmark` under multilib while the unpacked
  source is named `vkmark-git`; changed `S` to `${BPN}`. Assimp 5.0.1 also
  failed compiling because `MMDPmxParser.h` uses fixed-width integer types
  without including `<cstdint>`; added a focused header patch. Retry the
  vkmark patch and Assimp compile tasks to validate.
- The next `lib32-vkmark:do_configure` failure came from vkmark using the
  `wayland-scanner` pkg-config value `/usr/bin/wayland-scanner`, which is a
  target path rather than the native tool path. Updated vkmark to locate
  `wayland-scanner` through the build task's native `PATH`.
- The latest multilib configure failures for DisplayInfo, RDK GStreamer
  utilities, and systemaudioplatform all came from legacy `${RPI_UNPACKDIR}/git`
  source paths. Wrynose unpacks these repositories under
  `${UNPACKDIR}/${BPN}-${PV}`; updated each recipe's `S`, preserving the
  DisplayInfo and systemaudioplatform subdirectories where applicable. The
  GStreamer utilities source uses a plain Makefile and has no `configure`
  script, so removed the unused `autotools` class inheritance while retaining
  its explicit Makefile build. Retry the three configure tasks to validate.
- The next DisplayInfo and systemaudioplatform configure attempts reached their
  source trees but CMake 4 rejected the projects' `cmake_minimum_required(3.3)`;
  added `CMAKE_POLICY_VERSION_MINIMUM=3.5` to both recipes. Westeros-sink's
  lib32 compile also failed because V4L2 ioctl request macros are unsigned
  values wider than signed `int`; updated its debug wrapper to use `unsigned
  int` for the request parameter and compare the S_FMT request without a
  signed cast. The Westeros patch applies cleanly to the preserved source;
  BitBake configure and compile retries are still needed to validate.
- The latest `lib32-gstreamer1.0-plugins-good:do_compile` failure is caused by
  GStreamer 1.18 using TagLib 1's `TagLib::uint` frame factory overload while
  Wrynose provides TagLib 2.2.1, which instead accepts an ID3v2 header. Added
  a TagLib-major-version conditional patch using the header API for TagLib 2
  and preserving the old call for TagLib 1; retry compile to validate.
- The latest `lib32-sysint-soc:do_install` failure came from its legacy
  `${RPI_UNPACKDIR}/git` source path. The required unit exists under
  `${UNPACKDIR}/${BPN}-${PV}/systemd_units`; updated `S` to that Wrynose
  unpack path. Also fixed the reported missing whitespace around the
  `tvsettings-hal-headers` `SRCREV` assignment.
- The latest six HAL configure failures used `${RPI_UNPACKDIR}/git` although
  their sources are unpacked under `${UNPACKDIR}/${BPN}-${PV}`; updated each
  recipe's `S`. `westeros-sink` could not link `-lwesteros_compositor` because
  the Westeros install append removed `${D}${base_libdir}`; removed that broad
  deletion so the built compositor library remains available for sysroot and
  packaging. Retry all seven tasks to validate.
- The latest `librsvg` configure still could not load native Cargo's
  `libssh2.so.1`; explicitly export the native library directory in its
  configure task where Meson probes Cargo. The devicesettings configure lacked
  the `libudev` pkg-config module, so added `systemd` to its build dependencies.
  The glib-networking compile failure was a vendor patch passing a
  `GTlsConnectionBase *` to a helper requiring `GTlsConnection *`; use the
  concrete client connection cast. Westeros now packages its installed
  systemd unit. Finally, pwrmgr's configure-unsafe QA stems from its configure
  script using `PKG_CONFIG_SYSROOT_DIR` without inheriting the pkgconfig class;
  added that class so the include paths resolve in the target sysroot.
- The latest `mfrlibs` and `deepsleepmgr` compiles still used literal
  `/usr/include/rdk/...` paths because their Autoconf recipes use
  `PKG_CONFIG_SYSROOT_DIR` but did not inherit `pkgconfig`; added that class to
  both recipes. `librsvg` 2.61.3 reaches Cargo's version check but requires
  Rust 1.85.1 (`Cargo.toml` `rust-version`), newer than this build's toolchain.
  Selected OE-Core `librsvg` 2.58.2 instead; its `rust-version` is 1.70.0 and
  its locked crate set is provided with the matching recipe. The subsequent
  2.58.2 configure attempt exposed another incompatibility: it requires Pango
  1.50, while this build provides 1.44.7. The preference is now 2.50.7, whose
  configure requirements are Pango >=1.44 and Rust >=1.40; its release archive
  includes the Cargo vendor tree.
- The new `lib32-librsvg:do_configure` reached `do_qa_configure`, which
  rejected its `AM_GNU_GETTEXT` macros because the recipe did not inherit
  `gettext`; added that class. `lib32-ffmpeg:do_create_spdx` could not resolve
  the legacy `BSD` license identifier to an SPDX license text; replaced it
  with the recognized `BSD-2-Clause` and `BSD-3-Clause` identifiers.
- The subsequent `lib32-librsvg:do_compile` failed when Cargo could not load
  `libssh2.so.1` from its native sysroot. Explicitly prepend
  `${STAGING_LIBDIR_NATIVE}` to `LD_LIBRARY_PATH` in the compile task, matching
  the earlier configure-time runtime-library fix. The next compile reached
  rustc but could not find Rust's custom build-host target `x86_64-linux`;
  stage the Rust compiler's installed `${BUILD_SYS}.json` specification into
  librsvg's `RUST_TARGET_PATH` before invoking Cargo.
- `lib32-mesa:do_populate_lic` reported an updated `docs/license.rst`
  checksum; refreshed the checksum after confirming the source still identifies
  the core Mesa license as MIT. The latest fontconfig license failure came from
  its OSS `.bbappend`, which still overrode the updated base recipe checksums;
  refreshed the three append checksums against the QA-reported source text.

## Historical build-triage log

The following entries record the sequence of earlier build failures and fixes;
the current build status above supersedes their interim status statements.

- The user confirmed BitBake parsing succeeded after adding the targeted
  unmatched-append masks.
- `core-image-minimal` is still being validated. A reported circular SPDX
  dependency between `systemd` and `libcgroup` was addressed by disabling
  libcgroup's optional `systemd` PACKAGECONFIG for this product. To resolve the
  separate `OpenSSL -> systemd -> kmod -> OpenSSL` cycle, the Raspberry Pi
  Wrynose configuration now follows XD4: it selects OpenSSL 3.5.7, removes the
  `ssl-1.1.1` feature and `openssl-1.1.1l` package from this Wrynose image, and
  masks the legacy 1.1.1 recipes and appends. This intentionally drops the old
  compatibility package for Wrynose. A subsequent parse error showed that
  preferred versions alone do not prevent BitBake from parsing legacy recipes;
  those recipes are now masked explicitly. The latest build passed parsing and
  reached `opkg-native:do_patch`, where QA rejected an invalid `Upstream-Status`
  in an RDK opkg patch. That status is now marked `Pending`. The following build
  advanced to `breakpad-native:do_unpack` and rejected the shared Breakpad
  include's obsolete `S = "${UNPACKDIR}/git"` assignment; that override was
  removed so OE-Core's default applies. The following `do_patch` found the LSS
  checkout under the legacy `git/` destination instead of the default source
  directory. Breakpad's secondary Git destinations now use
  `${BB_GIT_DEFAULT_DESTSUFFIX}`. The latest patch-fuzz failure affects four
  legacy Breakpad patches. Following XD4, these are now excluded from both the
  target and native Breakpad recipes for this Raspberry Pi Wrynose machine;
  their custom behaviors are not included in this Wrynose build. The image
  build then reached `libpcre-native:do_populate_lic`, which rejected the
  obsolete `BSD` identifier. The recipe now uses `BSD-3-Clause`, matching
  XD4's `libpcre_8.39.bb`. The following run reached `rpi-bootfiles:do_fetch`;
  a vendor append was overriding the base recipe's matching firmware checksum
  with a stale value. Removed that redundant override so the base recipe's
  checksum for the pinned commit is used. The latest run reached
  `gnutls-native:do_patch`, where two CVE patches had malformed
  `Upstream-Status` syntax. Both now use Yocto's `Backport [URL]` format. The
  following run reached `glib-2.0-native:do_patch`; its CVE patch had the
  upstream merge-request URL on a separate line. The header now uses
  `Backport [URL]` on one line. The next run reached
  `libcgroup-3.2.0:do_patch`, where its legacy libcgroup 0.41 init patch did
  not apply. Added a version-selected 3.2.0 port, retaining the original patch
  for other versions and preserving the RDK systemd service behavior. The
  image build remains to be verified. The following run reported three more
  patch issues: the syslog-ng monotonic timestamp patch targeted 3.36.1 rather
  than 4.11.0, the BusyBox systemd socket-activation patch applied with fuzz,
  and the legacy systemd 230 patch stack failed patch QA. Following XD4, the
  Wrynose product now selects systemd 259.5, removes the `systemd230` feature,
  and masks the legacy 230 recipe and append. The syslog-ng patch was refreshed
  for 4.11.0 and the BusyBox patch hunk was refreshed for 1.37.0. Raspberry Pi
  boot validation with systemd 259.5 is still required. The latest Android
  kernel build reported two missing patch status headers. Both are now marked
  `Inappropriate`: one RDK vendor compatibility patch and one temporary
  AppArmor workaround, retained with user-confirmed product approval. The
  latter disables the AppArmor `no_new_privs` transition check and remains a
  security-sensitive product exception. The subsequent
  `glib-2.0-native:do_populate_sysroot` failure was caused by GDB auto-load
  scripts embedding the build `TMPDIR` in their install path. Added the
  native-only GDB directory cleanup from OE-Core's GLib install logic to the
  RDK GLib 2.74.6 include. The next configure failure showed that
  `shared-mime-info-native` was using `msgfmt` 0.16.1, below Meson's 0.19.7
  minimum for XML translation merging. Its native variant enables NLS to
  request full `gettext-native`; the next run showed that this still resolved
  to the RDK gettext 0.16.1 provider. Wrynose now prefers OE-Core
  `gettext-native` 1.0 to ensure Meson uses a sufficiently recent `msgfmt`.
  The latest build reached `rpi-cmdline:do_compile`, where the vendor append
  looked for `cmdline.txt` under `UNPACKDIR` even though the base recipe
  generates it in `WORKDIR`. Updated the append to modify the generated file
  in `WORKDIR`. The subsequent GCC 15.3 `libgcc-initial:do_configure` failed
  because the Raspberry Pi security-flag removal left
  `-Werror=format-security` enabled after removing `-Wformat`; GCC's configure
  probe treats the resulting warning as an error. Disabled
  `SECURITY_STRINGFORMAT` for `libgcc-initial`, matching OE-Core's GCC
  exception. The next run exposed a second issue: RDKE's `FULL_OPTIMIZATION`
  references `${DEBUG_FLAGS}`, which Wrynose renamed to `DEBUG_LEVELFLAG`, so
  the compiler received the literal `${DEBUG_FLAGS}` as an input file. Added
  the compatibility alias used by XD4 to `rdke-config.inc`. The latest build
  passed that bootstrap and failed in glibc configure because it selected
  vendor kernel headers 5.10.52, below glibc's 5.15 minimum. Wrynose now
  prefers OE-Core's generic Linux libc headers 6.18, matching XD4; this does
  not change the Raspberry Pi runtime kernel version. The next build reached
  `libgcc:do_configure`; its compiler command again contained
  `-Werror=format-security` without `-Wformat`, so the same targeted
  `SECURITY_STRINGFORMAT` exception used for `libgcc-initial` is now applied to
  `libgcc`. The latest build compiled the kernel but failed generating
  `hifiberry-adc.dtbo`: the newer Raspberry Pi BSP default overlay list
  includes six entries not provided by the pinned Android 6.1.77 kernel.
  Removed those unsupported overlays from the Wrynose list only.
  The subsequent build reported both `xorgproto:do_configure` and
  `libtool-cross:do_configure` failing because the compiler received
  `-Werror=format-security` after RDKE removed `-Wformat`. Added the missing
  error-flag token to the existing `SECURITY_CFLAGS` removal in both common
  and Raspberry Pi product configuration.
  The latest build advanced to `bzip2:do_package`, where the distro's GPLv3
  exclusion incorrectly classified `bzip2-staticdev` and `bzip2-locale` using
  the recipe's aggregate license expression. Added package-specific license
  assignments for both packages matching bzip2's `bzip2-1.0.6` license; GPLv3
  remains excluded globally. The next run reached `libidn2:do_package` and
  exposed the same aggregate-license issue for its development, debug,
  documentation, source, and generated locale packages. Followed XD4's
  `libidn2` append: assign the library's dual GPL-2/LGPL-3 license to those
  packages and remove the separate GPLv3 `-bin` package.
  The latest build also reports `kbd:do_package` fatal QA without the
  `kbd-keymaps-pine` package's genuine GPL-3.0-or-later license. Excluded that
  optional map package and its files from Wrynose and removed the dangling
  recommendation, preserving the other kbd packages. Kernel module compilation
  found a Btrfs root-name array too short for the NUL-terminated
  `BLOCK_GROUP_TREE` string; added a narrow kernel patch to increase the array
  by one byte. The latest build failed in `dbus:do_install` because
  `messagebus` was missing from its recipe sysroot. RDKE moves recipe-sysroot
  post-functions to `do_install_ipk_recipe_sysroot`, but the user-add helper
  did not recognize that task name; updated the helper to handle it. GLib's
  target debug files also retained the build path because LTO generates debug
  info at link time; pass the debug-prefix map to target link steps when LTO
  is enabled. The next DBus install passed the ownership step and exposed an
  unconditional install of the CCSP header, which exists only when the
  broadband-specific source patch is applied; scoped the header install to the
  broadband override. The image then failed while creating its tar archive:
  the legacy `tar-native` selected for Wrynose does not support
  `--sort=name`, required by OE-Core's reproducible image command. Prefer the
  OE-Core 1.35 native tar for Wrynose. The first lib32 image build found
  AppArmor's source directory and cross-compilation patch were not multilib
  compatible: `${PN}` changes under multilib, and the patch header named a
  nonexistent `.orig` file. Use the unprefixed `${BPN}` for the source path and
  patch the actual `Makefile.am`. The next parse failed in the stack-layer
  dependency resolver because recipe parse workers do not receive
  `BB_CMDLINE`; now generate `target_pkg_list` from the main cooker's
  `CacheLoadStarted` event and safely handle a missing command line in workers.
  The lib32 build also failed compiling zlib 1.2.11: its `gzguts.h` undefines
  `_FILE_OFFSET_BITS`, conflicting with Wrynose's global `_TIME_BITS=64` flags.
  Disable the time64 define only for the lib32 zlib compilation.
  The next build reached lib32 GNU tar 1.17 and failed its bundled gnulib
  `mktime.c` compile-time check: its 32-bit `long` cannot support that code's
  assumptions with 64-bit `time_t`. Disable `_TIME_BITS` only for this legacy
  lib32 tar build. Perl then attempted to compile `NDBM_File` against the
  legacy GDBM 1.8.3 `ndbm.h`, whose unprototyped functions are treated as
  zero-argument declarations by the newer compiler. The existing disable
  options were placed in `EXTRA_OECONF`, which Perl's custom configure task
  does not consume; pass them via `PACKAGECONFIG_CONFARGS` instead.
  The `lib32-vendor-test-image` parse still selected the RDK OSS
  `librsvg_2.40.21.bb` after the legacy PV overrides were removed; that recipe
  requires `libcroco`, which is not provided by the Wrynose layer set. Removed
  the packagegroup's obsolete direct `libcroco` dependency and explicitly set
  `PREFERRED_VERSION_librsvg = "2.61.3"` initially selected OE-Core, but that
  release requires Rust 1.85.1. Updated the preference to 2.58.2, but its Pango
  1.50 minimum exceeds this build's Pango 1.44.7. The active preference is now
  2.50.7, which supports the available Rust and Pango versions, after the next
  parse selected the vendor 2.40.21 recipe and failed on its unavailable
  `libcroco` dependency.
  The latest build advanced past that dependency and stopped because the
  Raspberry Pi Wi-Fi firmware's `synaptics-killswitch` license flag was not
  accepted. Accepted the flag in the product machine configuration only.
  The following run failed configuring `gobject-introspection-native` 1.86.0:
  it found the RDK GLib 2.74.6, while this release requires GLib >=2.82.0.
  Prefer OE-Core GLib 2.88.2 for the native variant only, leaving target GLib
  unchanged. The next run failed unpacking lib32 Vulkan headers and loader
  because legacy Vulkan recipe appends explicitly set `S` to
  `${UNPACKDIR}/git`. Removed those assignments from the affected Vulkan
  appends and the related Volk recipe so OE-Core's default source directory is
  used.
  The latest lib32 run then found a Breakpad `patch-fuzz` QA failure and Mesa
  checksum mismatch: the Wrynose OE-Core checksum was for Mesa 26.0.5 even
  though the vendor layer pins Mesa to 22.0.3. Removed redundant descriptor
  closes and test-only code from the Breakpad fix, retaining its EOF read
  handling with clean patch context. Added the Mesa 22.0.3 checksum confirmed
  by OE-Core's kirkstone recipe and the downloaded official archive.
  The subsequent kernel-module build found another GCC 15 error in Btrfs:
  `kvcalloc()` arguments were reversed in `send.c`. Added a narrow kernel patch
  to pass element count before element size. The next lib32 patch failure was
  xkeyboard-config's remote color-key patch, which still used 2.11-era
  keycode-file context; XD4 carries the same old patch, while the BPI reference
  has no corresponding override. Rebased it against 2.47. The Cairo fetch
  mismatch came from the vendor PV pin selecting cairo 1.16.0 while inheriting
  the 1.18.4 checksum; added a matching vendor recipe append using the checksum
  verified against OE-Core's dunfell 1.16.0 recipe. The BPI reference has no
  cairo 1.16.0 version override.
  The next run failed in fontconfig install because optional configuration
  files were removed without `-f`; made those removals tolerate files absent
  from fontconfig 2.17.1. Vala's multilib configure could not resolve the
  introspection pkg-config module because its declared GLib minimum is 2.82.0
  while the selected target GLib is 2.74.6. Set Vala's `GI_GIRDIR` override to
  `${libdir}/gir-1.0` so the target library does not embed the staging sysroot;
  keep the vapigen wrapper's GIR search path on `${STAGING_LIBDIR}` for builds.
  The latest build failed configuring multilib `breakpad-wrapper` because the
  recipe hardcoded `S` to `${UNPACKDIR}/git`, while BitBake unpacked the named
  Git source under `sources/breakpad-wrapper-1.0.0`. Removed the stale `S`
  assignment to use BitBake's default source directory, which contains the
  recipe's `configure.ac` and `Makefile.am`.
  The GStreamer 1.18.5 patch stack applied, but patch QA rejected the
  tracer-hook patch for a missing `Upstream-Status`; marked it `Pending`
  because it adds a GStreamer API change with no recorded upstream submission.
  The subsequent lib32 libjpeg failure came from its custom unpack task
  extracting into `${WORKDIR}` although `S` points into `${UNPACKDIR}`; aligned
  the extraction destination with `S`. BlueZ 5.48 stores the firmware fallback
  define in `hciattach_bcm43xx.c` rather than the newer `hciattach.h`, so ported
  the Raspberry Pi firmware-path patch to the 5.48 source location.
  The latest BlueZ 5.48 patch QA report found missing or malformed
  `Upstream-Status` fields and six patches applying with fuzz. Added valid
  statuses, matching XD4's OEM-specific classifications where applicable, and
  refreshed the six hunks against their quilt preimages. All six patches pass
  zero-fuzz dry-runs. The XD4 reference has the corresponding 5.48 patch set;
  the BPI reference has no corresponding 5.48 patch set. The assistant could
  not rerun BitBake's `do_patch` task because BitBake is unavailable in this
  shell.
  The next build reported a Westeros patch lookup failure because the recipe
  pointed `S` at `${UNPACKDIR}/git`, while BitBake unpacked the source at
  `${UNPACKDIR}/westeros-soc-drm-${PV}`. Corrected `S`; both resolution hunks
  dry-run against that source with zero fuzz. Rust 1.72's snapshot installers
  were likewise extracted under `${UNPACKDIR}/rust-snapshot-components`, while
  its setup task searched `${WORKDIR}`. Updated the installer glob to use
  `${UNPACKDIR}`, consistent with the newer XD4 OE-Core Rust recipe. The XD4
  and BPI trees have no corresponding Westeros SoC recipe patch to compare.
  GStreamer 1.18.5 then failed during Meson setup because target
  gobject-introspection 1.86.0 requires GLib >=2.82.0, but the selected target
  GLib is 2.74.6. Opted Raspberry Pi Wrynose out of
  `gobject-introspection-data`, matching the XD4 RDK policy; the OE-Core
  introspection class will disable GIR generation for this build.
  Mesa 22.0.3 next rejected `-Dvulkan-manifest-per-architecture=false`,
  an option supplied by the newer shared `mesa.inc` but absent from Mesa 22's
  Meson options. Removed it conditionally in the Raspberry Pi Mesa append only
  for the vendor-pinned 22.0.3 version; newer Mesa releases retain the option.
  The next configure attempt then rejected `-Dexpat=enabled`, another newer
  Mesa option. Mesa 22.0.3 finds expat unconditionally on Linux, so remove only
  the unsupported PACKAGECONFIG flag for that version while retaining its
  `expat` build dependency. Comparing the complete configure invocation with
  Mesa 22.0.3's declared options found five more unsupported modern flags;
  excluded those features for 22.0.3 only. Also translated the software
  Gallium driver to Mesa 22's `swrast` name and the newer VirtIO Vulkan driver
  list to Mesa 22's `virtio-experimental` name, removing unsupported
  `gfxstream`. The XD4 reference uses the same newer Mesa metadata without the
  Raspberry Pi 22.0.3 pin; no Mesa recipe override was found in the checked BPI
  reference. The next Mesa configure attempt rejected `-Dglvnd=disabled`:
  Mesa 22 declares `glvnd` as a boolean, while the newer shared metadata emits
  `enabled`/`disabled`. Translate these values to `true`/`false` for Mesa
  22.0.3 only. After that fix, Mesa's enabled Raspberry Pi drivers required
  libdrm >=2.4.109, but the selected OSS recipe was 2.4.100. Updated the
  preferred target and SDK libdrm versions to 2.4.131, the available OE-Core
  recipe also used by the XD4 reference; the multilib preference mirrors to
  `lib32-libdrm`. The next run exposed two independent recipe/source mismatches:
  Westeros simpleshell's source is unpacked as
  `westeros-simpleshell-${PV}/simpleshell`, not `git/simpleshell`; corrected
  `S` so autotools can regenerate `configure` from its `configure.ac`.
  Separately, Cairo 1.16.0 source was being passed to OE-Core's Cairo 1.18.4
  Meson recipe, which expects `meson.build`. Aligned the Cairo vendor and
  preferred versions to 1.18.4, matching the XD4 Wrynose preference, and
  corrected the version-specific checksum to the 1.18.4 archive checksum.
  The simplebuffer recipe had the same stale `${UNPACKDIR}/git/simplebuffer`
  source path as simpleshell; changed it to the unpacked
  `westeros-simplebuffer-${PV}/simplebuffer` directory.
  The libjpeg-turbo 2.0.4 CMake project also uses a minimum version below 3.5;
  pass `CMAKE_POLICY_VERSION_MINIMUM=3.5` as recommended by CMake 4, without
  changing the package's declared minimum version.
  Cargo 1.72's native snapshot setup similarly looked for the installer under
  `${WORKDIR}`, while BitBake unpacked it under `${UNPACKDIR}/sources`.
  Updated the installer path to `${UNPACKDIR}`; the install prefix and runtime
  Cargo path remain unchanged.
- Earlier blockers addressed during triage:
  - Replaced the removed `debug-tweaks` image feature with Wrynose's supported
    root-login/password features.
  - Removed unused UEFI bootloader dependencies from the Raspberry Pi WIC
    image, which uses its own VFAT boot partition.
  - Disabled `ptest` for the Raspberry Pi product, following XD4 and BPI and
    avoiding ptest runtime dependencies on GPLv3 `make`.
  - Restored Wrynose `kbd` split packages required by systemd's
    `kbd-consolefonts` recommendation.
  - Changed the deprecated `BB_DISKMON_DIRS` action from `ABORT` to `HALT`.
- Targeted `git diff --check` validations have passed. No full BitBake build
  has been run by the assistant; builds are run by the user in the VS Code
  Docker terminal.

## Change summary

- Ported recipe metadata and overrides for Wrynose/BitBake compatibility across
  the Raspberry Pi vendor, RDK OSS, HAL headers, Bluetooth, and layer-support
  repositories.
- Fixed Git fetcher URLs for pinned sources that need `protocol` and
  `nobranch=1`.
- Followed XD4's Wrynose OpenSSL baseline: prefer version 3.5.7, remove the
  legacy OpenSSL 1.1.1 feature/package for this product, and mask its recipes
  and appends.
  Existing OpenSSL endpoint-logging gates remain for non-Wrynose builds.
- Updated the systemd Git fetch transport from `protocol=git` to
  `protocol=https`, eliminating the insecure-transport warning.
- Corrected the RDK opkg patch's malformed `Upstream-Status` from `NA` to the
  valid `Pending` value required by current patch QA.
- Removed the obsolete `S = "${UNPACKDIR}/git"` assignment from shared Breakpad
  metadata so Wrynose uses OE-Core's default source directory.
- Refreshed Breakpad's child-hang patch for current source context, removing
  duplicate descriptor closes and its disabled test-only kill block while
  retaining EOF handling without patch fuzz.
- Added the Mesa 22.0.3 archive checksum to the Raspberry Pi vendor layer;
  the Wrynose base checksum applies to Mesa 26.0.5, not the vendor pin.
- Replaced the generic Mesa append with a 22.0.3-specific append and source-
  compatible Freedreno path patch, keeping the OE-Core Mesa 26 patch unchanged.
- Updated Breakpad's secondary Git `destsuffix` values to
  `${BB_GIT_DEFAULT_DESTSUFFIX}`, keeping test, protobuf, and LSS sources under
  the default `${S}` tree.
- Matched XD4's Wrynose Breakpad handling by excluding four legacy patches for
  both target and native recipes, rather than carrying their fuzz against the
  newer source.
- Updated libpcre 8.39's license identifier to `BSD-3-Clause`, matching XD4 and
  the Wrynose generic license file.
- Removed the stale Raspberry Pi bootfiles checksum override from the vendor
  append; the base recipe's pinned checksum matches the downloaded archive.
- Corrected malformed upstream status headers on two GnuTLS CVE backports to
  Yocto's `Backport [URL]` format.
- Put the upstream merge-request URL into the GLib CVE patch's `Backport [URL]`
  status header.
- Added a libcgroup 3.2.0-specific port of the RDK init patch and selected it
  only for PV 3.2.0; older recipe versions retain the original patch.
- Followed XD4's Wrynose systemd baseline by selecting 259.5 and masking the
  legacy systemd 230 recipe and append for Wrynose.
- Refreshed the RDK syslog-ng monotonic timestamp patch for 4.11.0 and the
  BusyBox systemd socket-activation patch for 1.37.0.
- Corrected the systemd journald metadata patch's `Upstream-Status` to
  `Backport [URL]`.
- Added `Inappropriate` status headers to both Android kernel patches; the
  temporary AppArmor `no_new_privs` workaround is retained with product
  approval and documented as security-sensitive.
- Removed native GLib 2.74.6 GDB auto-load scripts during install to avoid
  embedding `TMPDIR` paths in the sysroot and failing sstate path QA.
- Enabled NLS for `shared-mime-info-native` so Meson uses full `gettext-native`
  with a sufficiently recent `msgfmt` for XML translation merging.
- Preferred OE-Core `gettext-native` 1.0 on Wrynose after BitBake selected the
  legacy RDK gettext 0.16.1 provider, whose `msgfmt` is too old for Meson's XML
  translation merge.
- Updated the Raspberry Pi command-line append to edit `${WORKDIR}/cmdline.txt`,
  the path produced by the base `rpi-cmdline` recipe, instead of assuming it is
  under `UNPACKDIR`.
- Disabled `SECURITY_STRINGFORMAT` for `libgcc-initial`; its bootstrap sysroot
  lacks libc headers, and the Wrynose product's format-flag removals otherwise
  leave `-Werror=format-security` active on GCC configure probes.
- Disabled `SECURITY_STRINGFORMAT` for `libgcc` as well; its configure command
  retained `-Werror=format-security` after the product removed `-Wformat`.
- Added the XD4 Wrynose compatibility definition for `DEBUG_FLAGS` in the
  common RDKE config, preventing the literal variable reference from reaching
  GCC target compiler probes.
- Preferred OE-Core's generic Linux libc headers 6.18 on Wrynose after glibc
  rejected the vendor 5.10.52 headers as older than its 5.15 minimum; the
  target Raspberry Pi kernel remains 6.1.77.
- Removed six overlays not present in the pinned Android 6.1.77 kernel from
  Wrynose's requested DTBO list, while retaining the shared Raspberry Pi BSP
  defaults for other machines and releases.
- Completed the RDKE security-flag removal by also removing
  `-Werror=format-security`; prior builds showed it left enabled without
  `-Wformat`, breaking compiler probes in `xorgproto` and `libtool-cross`.
- Assigned the bzip2 license specifically to `bzip2-staticdev` and
  `bzip2-locale`; these split packages had inherited the aggregate recipe
  license expression, incorrectly triggering the product's GPLv3 exclusion.
- Followed the XD4 libidn2 package-license mapping: use the library's
  dual GPL-2/LGPL-3 license for its non-binary and generated locale packages,
  and remove its separately licensed GPLv3 `-bin` package.
- Fixed the Android 6.1.77 Btrfs root-name table's undersized string array,
  which caused GCC's `-Werror=unterminated-string-initialization` failure.
- Fixed the Android 6.1.77 Btrfs `kvcalloc()` argument order, which GCC 15
  rejects with `-Werror=calloc-transposed-args`.
- Excluded kbd's GPL-3.0-or-later `keymaps-pine` package and data from Wrynose,
  preserving the GPL-2.0-or-later keymap package and removing its dangling
  recommendation.
- Registered dbus's existing user-add sysroot helper on
  `do_prepare_recipe_sysroot`, after the build showed that `messagebus` was
  absent when `dbus:do_install` attempted to set ownership; taught the helper
  to recognize RDKE's `install_ipk_recipe_sysroot` task.
- Passed debug-prefix maps to target link steps in LTO builds to prevent
  `TMPDIR` from appearing in GLib's target debug information.
- Selected OE-Core `tar-native` 1.35 on Wrynose because the legacy native tar
  does not support `--sort=name` required by reproducible image archives.
- Made AppArmor's source path use `${BPN}` for multilib builds and corrected the
  Perl cross-compilation patch's target filename.
- Generate the stack-layer resolver's target list in the main cooker before
  recipe parsing, rather than trying to read `BB_CMDLINE` from parse workers.
- Disabled `_TIME_BITS=64` specifically for lib32 zlib 1.2.11, whose internal
  header undefines `_FILE_OFFSET_BITS` before including glibc headers.
- Disabled `_TIME_BITS=64` specifically for lib32 GNU tar 1.17, whose bundled
  gnulib `mktime.c` does not support 64-bit `time_t` with a 32-bit `long`.
- Disabled Perl's NDBM/DBM detection through `PACKAGECONFIG_CONFARGS`, which
  Perl's custom configure task actually uses, avoiding the incompatible
  `NDBM_File` build against GDBM 1.8.3 headers.
- Removed the legacy `librsvg` 2.40.21 pins from the OSS and vendor
  configurations, initially preferring OE-Core `librsvg` 2.61.3 over the
  high-priority legacy recipe. After that version failed the Rust toolchain
  minimum, switched to OE-Core 2.58.2, but its Pango 1.50 minimum exceeded
  Pango 1.44.7 in the build. The active preference is now OE-Core 2.50.7,
  whose Rust and Pango minimums are compatible with the configured toolchain.
  Also removed the unavailable `libcroco` package from the OSS packagegroup.
- Accepted the Raspberry Pi Wi-Fi firmware's `synaptics-killswitch` license
  flag only in the RDKE Raspberry Pi product configuration, enabling the
  firmware dependency required by the platform packagegroup.
- Preferred OE-Core GLib 2.88.2 for native tools on Wrynose because
  `gobject-introspection-native` 1.86.0 requires GLib >=2.82.0; the RDK target
  GLib version remains unchanged.
- Removed obsolete `S = "${UNPACKDIR}/git"` assignments from the Vulkan
  headers, loader, tools, Volk, validation-layers, and utility-libraries
  metadata so BitBake Wrynose uses its default Git source directory.
- Restored missing Breakpad and dnsmasq source assets referenced by recipes.
- Removed invalid runtime `RPROVIDES` aliases containing `virtual/` and updated
  affected package dependencies.
- Added the AppArmor `SRCREV_FORMAT` for its two Git sources.
- Added exact, machine-scoped masks for the unmatched `.bbappend` files
  reported during parsing. No blanket dangling-append suppression is used.
- Kept the Raspberry Pi Android kernel at 6.1.77 and GCC at 15.3, per the
  migration decisions.

## Repository inventory

Paths are relative to each repository root. This inventory includes tracked
modifications and new source assets currently in the worktrees.

### `product/meta-product-raspberrypi`

Wrynose machine/product configuration, centralized append masks, ptest
exclusion, XD4's systemd baseline, and the libcgroup/systemd SPDX-cycle
workaround.

- `conf/layer.conf`
- `conf/machine/include/product.inc`
- `conf/machine/raspberrypi4-64-rdke.conf`
- `conf/machine/include/wrynose-unmatched-appends.inc` (new)

### `vendor/meta-vendor-raspberrypi-dev`

Wrynose recipe/layer compatibility, package-provider corrections, AppArmor
source revision format, and local build-template updates.

- `conf/layer.conf`
- `conf/include/vendor_pkg_versions.inc`
- `conf/machine/include/vendor.inc`
- `conf/template/local.conf.sample`
- `conf/templates/rdke/local.conf.sample` (new)
- `recipes-bsp/bootfiles/rpi-cmdline.bbappend`
- `recipes-bsp/bootfiles/rpi-bootfiles.bbappend`
- `recipes-core/packagegroups/packagegroup-vendor-layer.bb`
- `recipes-debug/vendor-tools/network-setup_1.0.bb`
- `recipes-graphics/gpu-layer/gpu-layer.bb`
- `recipes-graphics/westeros-soc/westeros-soc-drm.bb`
- `recipes-graphics/westeros/westeros-sink.bbappend`
- `recipes-halif/deepsleepmgr/deepsleepmgr-hal-raspberrypi4.bb`
- `recipes-halif/devicesettings/devicesettings-hal-raspberrypi4.bb`
- `recipes-halif/hdmicec/hdmicec-hal-raspberrypi4.bb`
- `recipes-halif/iarmmgrs/iarmmgrs-hal-raspberrypi4.bb`
- `recipes-halif/mfrlibs/mfrlibs-hal-raspberrypi4.bb`
- `recipes-halif/miracast/miracast-hal-raspberrypi4.bb`
- `recipes-halif/pwrmgr/pwrmgr-hal-raspberrypi4.bb`
- `recipes-kernel/android/android-raspberrypi.inc`
- `recipes-kernel/android/files/disable_nnp_lsm_check.patch`
- `recipes-kernel/android/files/0001-RDKEVL-6480-RPI-Kernel-ACK6.1-CompilationErrorFix.patch`
- `recipes-kernel/linux-libc-headers/linux-libc-headers.inc`
- `recipes-mac/apparmor/apparmor-vendor.bb`
- `recipes-noop/halservices/audio-service-rpi.bb`
- `recipes-noop/halservices/gst-drm-plugins-rpi.bb`
- `recipes-noop/halservices/secapi2-adapter-rpi.bb`
- `recipes-soc/displayinfo-soc/displayinfo-soc_git.bb`
- `recipes-soc/rdk-gstreamer-utils-platform/rdk-gstreamer-utils-platform.bb`
- `recipes-soc/sysint-soc/sysint-soc.bb`
- `recipes-soc/systemaudioplatform/systemaudioplatform_git.bb`

### `vendor/meta-vendor-raspberrypi-oss`

- `conf/layer.conf`
- `recipes-connectivity/bluez5/bluez5_5.48.bbappend`
- `recipes-graphics/wayland/wayland-default-egl_1.20.0.bb`
- `recipes-graphics/westeros/westeros-sink.bb`

### `vendor/meta-image-support-rdke`

- `conf/layer.conf`

### `common/meta-rdk-oss-reference`

Wrynose recipe/fetcher updates, package and license compatibility, restored
Breakpad/dnsmasq assets, and the `kbd` package split correction.

- `conf/layer.conf`
- `conf/include/package_revisions_oss.inc`
- `recipes-common/lighttpd/lighttpd_%.bbappend`
- `recipes-common/lighttpd/lighttpd_1.4.53.bbappend`
- `recipes-connectivity/networkmanager/networkmanager_1.43.7.bb`
- `recipes-connectivity/openssl/openssl_1.1.1%.bbappend`
- `recipes-connectivity/openssl/openssl_3.0.%.bbappend`
- `recipes-core/ctemplate/ctemplate_git.bb`
- `recipes-core/glib-2.0_2.74.6/glib_2.74.6.inc`
- `recipes-support/shared-mime-info/shared-mime-info_%.bbappend` (new)
- `recipes-core/busybox/busybox/0001-add-ENABLE_FEATURE_SYSTEMD-and-use-it-in-syslogd.patch`
- `recipes-core/kbd/kbd_%.bbappend`
- `recipes-core/libcgroup/libcgroup_%.bbappend`
- `recipes-core/libcgroup/libcgroup/init-3.2.0.patch` (new)
- `recipes-core/packagegroups/packagegroup-oss-layer.bb`
- `recipes-core/systemd/systemd-serialgetty.bbappend`
- `recipes-core/systemd/files/journald-minimal-client-metadata-caching.patch`
- `recipes-core/systemd/systemd_230.bb`
- `recipes-core/systemd/systemd_230.bbappend`
- `recipes-core/volatile-binds/volatile-binds.bb`
- `recipes-devtools/breakpad/2023.06.01/breakpad_2023.06.01.inc`
- `recipes-devtools/breakpad/breakpad_git.bb`
- `recipes-devtools/breakpad/breakpad_git.bbappend`
- `recipes-devtools/breakpad/breakpad/0001-Do-not-add-stack-pointer-to-clobber-list.patch` (new)
- `recipes-devtools/breakpad/breakpad/0001-exception_handler.cc-Match-the-types-for-SIGSTKSZ.patch` (new)
- `recipes-devtools/jsonrpc/jsonrpc_1.3.0.bb`
- `recipes-extended/libmanette/libmanette_1.0.bb`
- `recipes-extended/shadow/shadow_%.bbappend`
- `recipes-gplv2/dosfstools/dosfstools_2.11.bbappend`
- `recipes-gplv2/tar/tar_1.17.bb`
- `recipes-graphics/essosrmgr/essosrmgr.bb`
- `recipes-graphics/vulkan/vulkan-headers_1.4.%.bbappend`
- `recipes-graphics/vulkan/vulkan-loader_1.4.%.bbappend`
- `recipes-graphics/vulkan/vulkan-tools_1.4.%.bbappend`
- `recipes-graphics/vulkan/vulkan-utility-libraries_1.4.%.bbappend`
- `recipes-graphics/vulkan/vulkan-validation-layers_1.4.%.bbappend`
- `recipes-graphics/vulkan/vulkan-volk_1.4.%.bbappend`
- `recipes-graphics/vulkan/vulkan-volk_1.4.304.bb`
- `recipes-mac/apparmor/apparmor_3.1.7.bb`
- `recipes-multimedia/webp/libwebp_%.bbappend`
- `recipes-protocols/net-snmp/net-snmp_5.9.%.bbappend`
- `recipes-protocols/net-snmp/net-snmp_5.9.5.2.bbappend` (new)
- `recipes-protocols/net-snmp/net-snmp-5.9.5.2/DELIA-40163_wrynose.patch` (new)
- `recipes-protocols/net-snmp/net-snmp-5.9.5.2/support-multiple-local-certs_wrynose.patch` (new)
- `recipes-connectivity/openssl/openssl_1.1.1l.bbappend` (new)
- `recipes-support/cimplog/cimplog_1.0.bb`
- `recipes-support/cpeabs/cpeabs_1.0.bb`
- `recipes-support/dnsmasq/dnsmasq.inc`
- `recipes-support/dnsmasq/dnsmasq_%.bbappend`
- `recipes-support/dnsmasq/dnsmasq_2.83.bbappend`
- `recipes-support/dnsmasq/dnsmasq-2.83/init` (new)
- `recipes-support/dnsmasq/dnsmasq/lua-2.83.patch` (new)
- `recipes-support/dnsmasq/files/dnsmasq-noresolvconf.service` (new)
- `recipes-support/dnsmasq/files/dnsmasq-resolved.conf` (new)
- `recipes-support/dnsmasq/files/dnsmasq.conf` (new)
- `recipes-support/duktape/duktape_2.2.0.bb`
- `recipes-support/hidapi/hidapi_0.14.0.bb`
- `recipes-support/libsoup/libsoup_3.6.6.bb`
- `recipes-support/parodus/libparodus_1.0.bb`
- `recipes-support/parodus/parodus_1.0.bb`
- `recipes-support/parodus/paroduscl_1.0.bb`
- `recipes-support/syslog-ng/syslog-ng_%.bbappend`
- `recipes-support/syslog-ng/files/0001-monotonic-timestamp.patch`
- `recipes-support/ucresolv/ucresolv_1.0.bb`
- `recipes-support/wdmp-c/wdmp-c_1.0.bb`
- `recipes-support/webcfg/webcfg_1.0.bb`
- `recipes-support/wrp-c/wrp-c_1.0.bb`
- `recipes-xraudio/tensorflow-lite-lib/xraudio-tensorflow-lite-lib.bb`

### `common/meta-rdk-halif-headers`

- `conf/layer.conf`
- `recipes-rdk-halif-headers/audiocapture/media-utils-headers_git.bb`
- `recipes-rdk-halif-headers/ctrlm/ctrlm-hal-headers.bb`
- `recipes-rdk-halif-headers/ctrlm/ctrlm-irdb-headers.bb`
- `recipes-rdk-halif-headers/ctrlm/xr-voice-sdk-ffv-headers.bb`
- `recipes-rdk-halif-headers/deepsleepmgr/deepsleep-manager-headers.bb`
- `recipes-rdk-halif-headers/devicesettings/devicesettings-hal-headers.bb`
- `recipes-rdk-halif-headers/hdmicec/hdmicecheader_git.bb`
- `recipes-rdk-halif-headers/pwrmgr/power-manager-headers.bb`
- `recipes-rdk-halif-headers/tvsettings/tvsettings-hal-headers_git.bb`
- `recipes-rdk-headers/iarmbus/iarmbus-headers.bb`
- `recipes-rdk-headers/iarmmgrs/iarmmgrs-hal-headers_git.bb`
- `recipes-rdk-headers/rdk-gstreamer-utils/rdk-gstreamer-utils-headers.bb`

### `common/meta-stack-layering-support`

- `classes/base-deps-resolver.bbclass`
- `classes/gir-ipk-qemuwrapper.bbclass`
- `conf/layer.conf`

### `common/meta-openembedded`

- `meta-oe/recipes-extended/ostree/ostree_2026.1.bb`

### `configs/rdke/vendor/meta-rdk-bluetooth`

- `bluez5/bluez5_5.%.bbappend`
- `bluez5/bluez5_5.48.inc`
- `recipes-connectivity/bluetoothsdk/bluetoothsdk_git.bb`
- `recipes-core/basu/basu_0.2.1.bb`
- `recipes-core/sdbus-c++/sdbus-c++-tools_2.1.0.bb`
- `recipes-core/sdbus-c++/sdbus-c++_2.1.0.bb`
- `recipes-multimedia/pipewire/pipewire_1.5.85_git.bb`
- `recipes-multimedia/rmf-pipewire-plugin/rmf-audio-capture-pipewire_1.0.bb`
- `recipes-multimedia/wireplumber/wireplumber_0.5.13.bb`

### `configs/common`

- `conf/rdke-config.inc`

## Build-directory-only settings

These are local build configuration changes, not source-layer PR changes:

- `build-raspberrypi4-64-rdke/conf/local.conf`: replaced `debug-tweaks` with
  `allow-empty-password allow-root-login empty-root-password`; changed all
  `BB_DISKMON_DIRS` `ABORT` actions to `HALT`.

## Review and PR notes

- The source changes span separate Git repositories; review and submit each
  repository independently.
- This tracker is maintained in `product/meta-product-raspberrypi` as the
  cross-repository index.
- Changes are committed to `feature/rdke-wrynose` independently in each
  affected repository.
- Update the status and inventory as subsequent BitBake errors are resolved.
