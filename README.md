# op-source

OpenWrt source for Xiaomi AX3000T, Cudy TR3000 and Cudy TR1200, with
**Linux 6.18.56** and isolated device builds. The intended GitHub repository is
the private `tdsafgen1888asd/op-source` repository.

The complete source is based on official
[OpenWrt main](https://github.com/openwrt/openwrt/tree/0ad1348fb2a7f49c9c5941d4d8c98f88b6974e8d),
commit `0ad1348fb2a7f49c9c5941d4d8c98f88b6974e8d`. Device presets, build
automation, the TR1200 partition correction and audited download/signing fixes
are ported from `tdsafgen1888asd/Device-Build` at
`db96277fbc98282751cb1d70b83bc1cd8156eb4d`. Firmware uses OpenWrt branding and
official OpenWrt package feeds. Both source commits are recorded in
[config/device-build.json](config/device-build.json).

## Devices

| Selector | Device / layout | Target | Image profile |
| --- | --- | --- | --- |
| `ax3000t` | Xiaomi Mi Router AX3000T, stock layout | `mediatek/filogic` | `xiaomi_mi-router-ax3000t` |
| `tr3000` | Cudy TR3000 v1, stock layout | `mediatek/filogic` | `cudy_tr3000-v1` |
| `tr1200` | Cudy TR1200 v1 | `ramips/mt76x8` | `cudy_tr1200-v1` |
| `ax3000t-ubootmod` | AX3000T, OpenWrt U-Boot layout | `mediatek/filogic` | `xiaomi_mi-router-ax3000t-ubootmod` |
| `tr3000-256mb` | TR3000 256MB v1, stock layout | `mediatek/filogic` | `cudy_tr3000-256mb-v1` |
| `tr3000-ubootmod` | TR3000 v1, OpenWrt U-Boot layout | `mediatek/filogic` | `cudy_tr3000-v1-ubootmod` |

The first three selectors form the default build set. Optional variants are
selected explicitly. Choose the profile matching the hardware revision and
installed bootloader/flash layout. The TR3000 256MB selector describes the
separate upstream flash variant.

Each preset selects one device and retains its upstream Wi-Fi, networking,
calibration and upgrade support. MediaTek presets include LuCI HTTPS with
OpenSSL; TR1200 uses LuCI's smaller TLS backend for its 16MB flash. Package and
firmware signing are enabled. Image checks require a matching sysupgrade image,
target, profile, kernel version, file size and SHA256 checksum.

## Compile locally

Use Ubuntu 24.04, GNU make 4.1+, a case-sensitive filesystem and at least 40GB
of free build space. The complete dependency list is in
[the build workflow](.github/workflows/device-build.yml).

```sh
git clone https://github.com/tdsafgen1888asd/op-source.git
cd op-source
cp feeds.conf.release feeds.conf
./scripts/feeds update -a
./scripts/feeds install -a

python3 scripts/device-build.py list
python3 scripts/device-build.py configure ax3000t
make download -j4
make -j"$(nproc)" V=s
python3 scripts/device-build.py check-images ax3000t bin/targets/mediatek/filogic
bash scripts/verify-release.sh bin/targets/mediatek/filogic
```

The private repository requires GitHub authentication to clone. Use `configure
tr3000` or `configure tr1200` for the other default devices. TR1200 images go to
`bin/targets/ramips/mt76x8/`. Use separate checkouts for concurrent device builds
or different target architectures. Existing `.config` files are backed up as
`.config.device-build.backup` by the configuration helper.

Presets are in `configs/devices/`; `common.config` provides shared settings.
For custom packages, run `make menuconfig` after selecting the device and then
`python3 scripts/device-build.py check-config DEVICE` before compiling.

Development feeds are in `feeds.conf.default`. The documented procedure and CI
use `feeds.conf.release`, which pins all five official feeds to immutable
commits. Refresh these pins deliberately and test both target architectures.

## GitHub Actions

**OpenWrt Device Build** compiles the three default devices in independent
Ubuntu 24.04 jobs on relevant pushes to `main` and pull requests. Use
**Actions → OpenWrt Device Build → Run workflow** to select `all` or any of the
six selectors. Each job has its own configuration, toolchain, firmware and
diagnostic artifacts, with a cache for source downloads.

Uploads require valid image metadata and successful verification of every
sysupgrade image's signature using the built host tools. Archives include
checksums, the resolved build configuration, pinned feeds, source commit and
device manifest. CI uses disposable signing keys. For firmware distribution,
manage stable signing keys; different build keys imply different trust anchors.
The optional `REQUIRE_IMAGE_SIGNATURE` setting requires trusted signatures
during sysupgrade and should be used with stable owner-managed keys.

**Source and signing checks** runs the device, download and signing regression
tests, builds the pinned host signing tools and tests actual signing and
tampering rejection. Official OpenWrt's broader workflows are preserved for
reference in `.github/upstream-workflows/`.

## Validation

```sh
python3 scripts/device-build.py check-source
python3 -m unittest discover -s tests -v
```

The Linux 6.18.56 archive was verified against
[kernel.org's checksum list](https://cdn.kernel.org/pub/linux/kernel/v6.x/sha256sums.asc).
All **592 generic + MediaTek patches** and **498 generic + Ralink patches**
apply to the verified kernel sources and target overlays. Source checks pass
for all six profiles; **41 local regression tests pass**. Five tests using
compiled signing tools are delegated to Linux CI. Both active workflows pass
`actionlint`.

Full firmware compilation, image-size checks and physical boot/Wi-Fi/USB/
sysupgrade validation remain to be confirmed. See
[source provenance and validation](docs/OP-SOURCE.md) for the exact changes and
scope. The original upstream documentation is retained in
[README.OpenWrt.md](README.OpenWrt.md).

## License

The OpenWrt build system is distributed under GPL-2.0. Individual files and
packages retain their original licenses and copyright notices. See
[COPYING](COPYING), [LICENSES](LICENSES) and package-specific license files.
