# Raspberry Pi RDKE Wrynose Migration Review

Last updated: 2026-11-13

This is the cross-repository review tracker for the Raspberry Pi RDKE migration
to Yocto Wrynose. The layers below are separate Git repositories, so PRs should
be prepared per repository. Keep this inventory updated as build triage
continues.

## Current status (2026-11-13)

- The user confirmed both `core-image-minimal` and
  `lib32-core-image-minimal` built successfully.
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
  Added a version-specific append and patch that preserves the absolute path
  for build-time generators while keeping it out of the compiled runtime path.
  The refreshed patch applies cleanly without fuzz; retry `lib32-mesa:do_patch`
  and the image build to validate.
- The subsequent `spirv-headers-native:do_configure` failure came from the
  shared recipe include setting `S` with `${PN}`, which adds `-native` even
  though unpack names the source directory with `${BPN}`. Updated the include
  to use `${BPN}`; retry the image build to validate.
- The subsequent `rust-llvm-native:do_patch` failure could not resolve the
  `llvm-config.cpp` target for the Yocto environment override patch. Normalized
  the patch paths relative to the recipe's LLVM source directory and set its
  strip level to `1`; retry to validate.
- The latest build exposed two remaining source-path mismatches. SPIR-V Tools
  used `${PN}` and incorrectly included `-native` in its unpack directory;
  switched both SPIR-V shared includes to `${BPN}`. Rust 1.72 still pointed at
  `${WORKDIR}` instead of Wrynose's `${UNPACKDIR}`. Updated `RUSTSRC` and
  retained the LLVM patch's corrected relative path/strip level. Verified the
  Rust patch applies to the actual unpacked LLVM source without fuzz; retry the
  two failed tasks and image build.
- Source changes are committed independently on `feature/rdke-wrynose` in 11
  affected repositories. All 11 feature branches are now pushed: five to
  `rdkcentral` and six to the user's `h1712` forks.
- The top-level Raspberry Pi manifest and both common/config submanifests are
  also updated and pushed on `feature/rdke-wrynose`, with pins for all 11
  changed projects.

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
  `PREFERRED_VERSION_librsvg = "2.61.3"` to select OE-Core's compatible recipe.
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
  to pass element count before element size.
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
  configurations, explicitly preferred OE-Core `librsvg` 2.61.3 to win over
  the high-priority legacy recipe, and removed the unavailable `libcroco`
  package from the OSS packagegroup.
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
