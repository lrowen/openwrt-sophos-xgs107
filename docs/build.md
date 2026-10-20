# Building the images

```sh
build/build.sh            # both; or: build/build.sh npu | x86
```

Output: `dist/` (see below). A cold build takes about an hour per target
(the toolchain dominates), 30 GB of disk each. Nothing outside the repository
is touched: the OpenWrt checkout lives in `build/openwrt/` (ignored by git).

**Host requirements**: Debian/Ubuntu-like Linux and about 8 GB of RAM:

```sh
sudo apt install build-essential git python3 python3-setuptools rsync unzip wget curl \
    file patch flex bison perl gawk libncurses-dev libssl-dev zlib1g-dev e2fsprogs swig
```

(OpenWrt's own list: <https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem>.)
Do not build as root. `build/build.sh` checks all of this before it starts and
prints what is missing.

Two requirements are easy to miss:

- **`gawk`**: OpenWrt needs GNU awk. The default `awk` on Debian/Ubuntu is
  `mawk`, which has no `asort()`, and the feed index step then fails with
  `function asort never defined`. After installing it, make sure `awk --version`
  says GNU Awk (`sudo update-alternatives --set awk /usr/bin/gawk`).
- **`libncurses-dev`**: OpenWrt's prerequisite check fails with
  `Build dependency: Please install ncurses` without it.

If a first run failed before these were installed, the next run clears the stale
feed indexes in `build/openwrt/tmp` and `build/openwrt/feeds/*.index` by itself.
The `WARNING: No feed for package 'libc' found` lines during the feed update are
harmless.

Environment variables (all optional): `OPENWRT_DIR`, `OPENWRT_URL` (a local
mirror is much faster), `JOBS`, `DL_DIR` (shared download cache), `DIST_DIR`.

## What the script does

1. Clones OpenWrt and checks out the pinned commit
   (`OPENWRT_COMMIT` in `build/build.sh`, kernel 6.18.52).
2. `openwrt/apply.sh` puts the project's changes into the tree:
   - `openwrt/patches/*.patch`: edits to existing files (kmod packages,
     target config, board network, image profile, kernel options);
   - `openwrt/npu/`: the device tree, base-files (vNTB init script, LuCI
     page, defaults) and the NPU kernel patches;
   - `openwrt/x86/`: the x86 kernel patches and, for the x86 build only, the
     root filesystem overlay (network defaults, the NTB link supervisor).
3. Updates the feeds (pinned to the commits of `build/feeds.conf`, so the
   package set does not drift), copies `build/config/<target>.config` to `.config`,
   `make defconfig`, `make`.
4. Collects the artifacts and checksums into `dist/`.

To change something, edit the file in `openwrt/` (or the config in
`build/config/`) and rerun; the script reapplies idempotently. To regenerate
`openwrt/patches/` after changing files in the OpenWrt tree, commit them there
and `git format-patch <pinned-commit>..`.

## `dist/`

| File | Used by |
|---|---|
| `npu-p1.img.gz`, `npu-p1.md5`, `npu-p1.sha256` | `install-npu.sh`: ext4 image of eMMC slot p1 (OpenWrt rootfs + `/boot/Image` + DTB) |
| `x86-vmlinuz`, `x86-rootfs.img.gz`, `x86-env.sh` | `install-x86.sh`: kernel, 3.7 GB ext4 root (compresses to ~20 MB), checksums |
| `install-npu.sh`, `install-x86.sh`, `x86-mbr.py`, `uboot-env.txt`, `backup-stock.sh` | copied from `install/` |
| `npu-initramfs-kernel.bin` | RAM-boot test image for the NPU (not used by the installers) |
| `SHA256SUMS` | integrity of the above |

## Kernel patches, in one place

| Patch | Target | Why |
|---|---|---|
| `950-net-dsa-mv88e6xxx-6191x-6193x-no-ptp` | NPU | the 88E6193X has no PTP unit but the driver assumed one, so it fails to probe (TAI period reads 0) |
| `951-PCI-dwc-plat-ep-1M-BAR-align` | NPU | endpoint BARs are 1 MiB granular: without alignment the host reads and writes the wrong memory |
| `952-ntb-netdev-name-ntbN` | both | a stable `ntb0` name |
| `953-ntb-hw-epf-prefer-msi` | x86 | the endpoint raises MSI, the DesignWare core always advertises MSI-X |
| `954-pci-epf-vntb-poll-1ms` | NPU | doorbells from the host are found by polling; 5 ms became 10 ms at HZ=100 |

Details of what each fixed: [npu-x86-link.md](npu-x86-link.md).
