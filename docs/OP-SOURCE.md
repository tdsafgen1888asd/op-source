# op-source provenance and validation

## Source baselines

| Item | Recorded value |
| --- | --- |
| Repository | `tdsafgen1888asd/op-source` (public, confirmed 2026-10-15) |
| Base | Official `openwrt/openwrt`, branch `main` |
| Base commit | `0ad1348fb2a7f49c9c5941d4d8c98f88b6974e8d` |
| Customization reference | `tdsafgen1888asd/Device-Build`, branch `main` |
| Reference commit | `db96277fbc98282751cb1d70b83bc1cd8156eb4d` |
| Import date | 2026-10-13, Asia/Shanghai |
| Kernel | Linux 6.18.57 |
| Kernel archive SHA256 | `5bd9acdc82ec5efc3790ae840e5aaa40dfde7efff261cf6dde902dc570acf98c` |

The source is an independent snapshot of the official OpenWrt tree with the
listed customizations. Upstream history is available at the recorded base
commit. All source copyright notices and licenses are retained.

## Device-Build customizations

- Import six one-device presets and the manifest-driven device selector, with
  AX3000T stock, TR3000 v1 stock and TR1200 v1 as the default build matrix.
- Port isolated Ubuntu builds, metadata/hash checks and source provenance
  artifacts. Verify actual firmware signatures before uploading artifacts.
- Preserve OpenWrt's device trees, image recipes, drivers, network configuration
  and calibration mappings for these profiles.
- Correct TR1200's `partition@fe0000` address to `0xfe0000`, matching its unit
  address and end-of-flash backup partition.
- Port verified TLS, archive-member validation, private temporary download
  directories and safe cache handling from the audited download helper.
- Generate firmware signing material and install the trust anchor for APK
  builds. Port the optional trusted-signature policy and fail-closed signing
  recipes into current OpenWrt files while retaining upstream's other changes.
- Port the ucert append exit-status and buffered-close error fix, retaining the
  upstream source pin and raising the package release to 2.
- Replace the reference's ImmortalWrt feeds with pinned official OpenWrt feeds.
  Firmware distribution and manufacturer defaults remain OpenWrt.

The inherited PON/Airoha-specific integration is outside the three-device port;
other devices use their definitions from the recorded official OpenWrt base.

## Linux 6.18.55 to 6.18.56

The official source snapshot selects Linux 6.18.55 for MediaTek and Ralink.
Update the generic kernel version/hash to 6.18.56 and retain current official
OpenWrt kernel patches, with these stable-kernel adjustments:

1. Remove `438-v7.3-mtd-spinand-Do-not-update-the-QE-bit-on-devices-without-one.patch`.
   Linux 6.18.56 already limits quad-enable register updates to devices with
   `SPINAND_HAS_QE_BIT` and returns zero from successful chip configuration.
2. Remove `782-v7.3-USB-serial-option-add-Quectel-EG060W.patch`, whose USB serial
   device support is already present in 6.18.56.
3. Remove `787-v7.3-net-usb-qmi_wwan-add-Quectel-EG120K-EA.patch`, whose Quectel
   `2c7c:030b` QMI device entry is already present in 6.18.56.
4. Refresh the LoongArch vDSO prefix-map patch around the stable kernel's
   `CONFIG_CC_OPT_INLINE_MEMSET` line using Device-Build's tested context.

Removing duplicate backports preserves the fixes present in the stable kernel.
The Ralink PWM fixes in current OpenWrt already match the corrected reference.

## Linux 6.18.57 update, 2026-10-15

The manifest and generic kernel pin now select Linux 6.18.57. The source checker
reads the version and SHA256 from the manifest and rejects inconsistent kernel
metadata. Workflow step labels and firmware archive names use the manifest's
kernel version as well.

The official kernel archive was downloaded and verified against kernel.org's
published SHA256. All 592 generic/MediaTek patches and all 498 generic/Ralink
patches applied to the new kernel without rejects. No additional patch changes
were required.

The initial 6.18.56 firmware run compiled all three default devices and passed
the image metadata checks, then failed during standalone signature verification
because ucert could not find `usign`. The verifier now prepends the built host
tool directory to PATH. Regression coverage exercises verification without the
build process's host-tool PATH.

The owner stopped cloud testing after GitHub blocked jobs with an account
billing/spending-limit error, then requested local macOS testing. This update
uses `[skip ci]`; no new GitHub Actions builds are started by this commit.
Future builds can be started manually through **OpenWrt Device Build**.

`docs/OPENWRT-CHANGES.patch` records the original 6.18.56 import. Later source
updates are recorded in Git history.

## mac80211 compatibility fix and preflight, 2026-10-16

Linux 6.18.57 adds `struct sockaddr_unsized`, but backports 7.2 provides its own
definition for kernels older than 6.19. The duplicate definition stops
`net/wireless/core.c` from compiling. The mac80211 build patch now stops defining
the compatibility structure at 6.18.57, and its package release increases to 6.
The separate `kernel_connect` wrapper retains its 6.19 cutoff: that function
still takes `struct sockaddr *` in Linux 6.18.57.

Local preflight on Apple Silicon macOS:

- Verified the backports 7.2 archive against the package's SHA256.
- Reproduced the original duplicate-definition compiler error before patching.
- Applied all 191 mac80211 patches in build-system order without rejects.
- Passed eight AArch64 compiler probes for the socket wrapper, covering version
  cases 6.6.0, 6.12.0, 6.18.0, 6.18.56, 6.18.57, 6.18.58, 6.19.0 and 7.0.0.
- Passed three AArch64 compiler probes for the `kernel_connect` wrapper using
  native declarations from Linux 6.18.56, 6.18.57 and 6.19.
- Passed source checks for Linux 6.18.57 and all six device profiles, 41
  regression tests and shell syntax validation of the release verifier. Five
  signing integration tests were skipped because their temporary native tools
  were unavailable; the previous day's signing results remain recorded below.

The compiler probes use the actual patched backports wrappers, the socket type
from Linux 6.18.57 and the listed native function declarations, with minimal
include stubs and simulated version codes. They verify this compatibility fix;
they do not compile a complete mac80211 module, kernel or firmware image.
Results are recorded in `docs/validation/mac80211-preflight.json`. No additional
errors were found within these checks. This fix uses `[skip ci]` to respect the
owner's request to stop cloud builds.

After updating an existing Linux build tree, clean and rebuild mac80211 before
resuming the firmware build:

```sh
git pull --ff-only
make package/kernel/mac80211/clean
make package/kernel/mac80211/compile -j1 V=s
make -j"$(nproc)"
```

## Local macOS checks on 2026-10-15

- Passed source checks for Linux 6.18.57 and all six device profiles.
- Passed 41 regression tests covering device selection, image metadata/hashes,
  archive extraction, download/cache handling and signature-verification
  orchestration, including the missing host-tool PATH regression.
- Built the repository-pinned json-c, libubox, usign, ucert and fwtool sources
  natively on Apple Silicon, after verifying source checksums and applying the
  package patches. The temporary build used CMake 3.31.6, pkgconf 2.5.1 and the
  macOS 15.4 SDK with Apple Clang 21.
- Passed four real signing-tool integration tests: certificate append and
  verification, failed certificate writes, truncated signatures, and the real
  image signing recipes plus standalone release verification. Verification
  succeeds with the signing tools removed from the inherited PATH and rejects
  tampered images. The fifth test needs Linux's `/dev/full` and was skipped.
  Overall, 45 tests passed and one Linux-only test was skipped.
- Passed `actionlint` 1.7.12 for both active workflows and shell syntax checks
  for the signature verification helper.

The local host is Apple Silicon macOS, with a case-insensitive workspace and
GNU make 3.81. A full native build needs a case-sensitive build volume and the
build prerequisites, including GNU make 4.1 or newer. No Linux VM or container
runtime was found. Full Linux 6.18.57 firmware generation remains unverified.

## Checks on 2026-10-13

- Verified the downloaded official `linux-6.18.56.tar.xz` against the recorded
  SHA256 and kernel.org's published list.
- Applied all 592 active generic/MediaTek patches and all 498 active
  generic/Ralink patches in build-system order, with generic and target file
  overlays. Every patch applied and no `.rej` files remained.
- Passed source checks for all six device profiles.
- Passed 41 local regression tests for selection, image metadata/hashes,
  archive extraction, TLS/cache handling and signing orchestration. The five
  real-host-signing-tool integration tests were skipped locally because their
  compiled tools were unavailable.
- Passed `actionlint` for both active workflows and shell syntax validation for
  the signature verification helper.

Patch applicability and source assertions do not establish successful kernel or
firmware compilation. TR1200 image size,
physical boot, calibration, Wi-Fi, USB and upgrade behavior require completed
firmware builds and hardware testing.

The local checks use original kernel contents of all patch-referenced files,
plus target overlays. They do not compile the kernel. Signed-image integration
CI builds the actual pinned json-c, libubox, usign, ucert and fwtool sources,
applies their package patches and tests successful verification, write/flush
errors, truncated signatures and tampered firmware.
