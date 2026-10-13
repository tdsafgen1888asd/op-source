# op-source provenance and validation

## Source baselines

| Item | Recorded value |
| --- | --- |
| Repository | `tdsafgen1888asd/op-source` (private) |
| Base | Official `openwrt/openwrt`, branch `main` |
| Base commit | `0ad1348fb2a7f49c9c5941d4d8c98f88b6974e8d` |
| Customization reference | `tdsafgen1888asd/Device-Build`, branch `main` |
| Reference commit | `db96277fbc98282751cb1d70b83bc1cd8156eb4d` |
| Import date | 2026-10-13, Asia/Shanghai |
| Kernel | Linux 6.18.56 |
| Kernel archive SHA256 | `4633bbda2ab95eaf2a7af4041236451d6759c72e6946265531af4b3aacefbf89` |

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

## Checks on 2026-10-13

- Verified the downloaded official `linux-6.18.56.tar.xz` against the recorded
  SHA256 and kernel.org's published list.
- Applied all 592 active generic/MediaTek patches and all 498 active
  generic/Ralink patches in build-system order, with generic and target file
  overlays. Every patch applied and no `.rej` files remained.
- Passed source checks for all six device profiles.
- Passed 41 local regression tests for selection, image metadata/hashes,
  archive extraction, TLS/cache handling and signing orchestration. The five
  real-host-signing-tool integration tests require Linux CI and were skipped
  locally because their compiled tools were unavailable.
- Passed `actionlint` for both active workflows and shell syntax validation for
  the signature verification helper.

Patch applicability and source assertions do not establish successful kernel or
firmware compilation. The local macOS host has GNU make 3.81; Linux configuration
resolution and full builds are delegated to GitHub Actions. TR1200 image size,
physical boot, calibration, Wi-Fi, USB and upgrade behavior require completed
firmware builds and hardware testing.

The local checks use original kernel contents of all patch-referenced files,
plus target overlays. They do not compile the kernel. Signed-image integration
CI builds the actual pinned json-c, libubox, usign, ucert and fwtool sources,
applies their package patches and tests successful verification, write/flush
errors, truncated signatures and tampered firmware.
