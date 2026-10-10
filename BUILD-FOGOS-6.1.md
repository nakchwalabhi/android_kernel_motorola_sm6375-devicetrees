# Building a bootable fogos 6.1 (blair) — build kit

Two routes. **Route A** builds the whole LineageOS 23.1 ROM and gives you a
matched, bootable 6.1 system (the reliable path). **Route B** builds only the
kernel side standalone (no AOSP framework) — faster to iterate, but not a
complete bootable system on its own.

Verified facts used below (from your 6.1 kernel tree, branch `lineage-23.1`):
- `build.config.constants`: `BRANCH=android14-6.1`, `CLANG_VERSION=r487747c`
- `build.config.msm.blair`: `MSM_ARCH=blair`, `VARIANTS=(consolidate gki)`,
  `BOOT_IMAGE_HEADER_VERSION=4`, `BUILD_VENDOR_DLKM=1`, `PREPARE_SYSTEM_DLKM=1`,
  `BUILD_INIT_BOOT_IMG=1`, `DT_OVERLAY_SUPPORT=1`
- The kernel repo ships **no** `arch/arm64/boot/dts/vendor` — the devicetrees
  repo must be merged in.

Build host requirements (either route): Linux x86_64, **≥16 GB RAM** (32 GB
comfortable), **≥250 GB free disk**, Ubuntu 22.04/24.04, fast internet.

---

## Route A — full LineageOS 23.1 ROM (recommended for a bootable system)

This produces boot.img / init_boot.img / vendor_boot.img / dtbo.img /
vendor_dlkm.img / vbmeta.img all built from the same 6.1 sources, so the GKI
kernel and its DLKM modules match by construction.

### 1. Host deps
```
sudo apt update
sudo apt install -y git-core gnupg flex bison build-essential zip curl \
  zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses5-dev \
  x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils \
  xsltproc unzip fontconfig python3 python3-pip ccache rsync
mkdir -p ~/bin && curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo && export PATH=~/bin:$PATH
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
```

### 2. Sync the 6.1 tree
```
mkdir -p ~/fogos6 && cd ~/fogos6
repo init -u https://github.com/LineageOS/android.git -b lineage-23.1 --git-lfs
# drop the local manifest (next section) into .repo/local_manifests/
repo sync -c --force-sync --no-clone-bundle --no-tags -j8
```

### 3. Local manifest — `.repo/local_manifests/fogos-6.1.xml`
Use the org's `6.1/lineage-23.1` manifest, with the three repos you own
substituted. Keep the rest pointing at Fogos-Playground.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <remote fetch="https://github.com/Fogos-Playground" name="fogos-plygrnd" revision="lineage-23.1" />
  <remote fetch="https://github.com/nakchwalabhi"     name="nakchwal" />

  <!-- your forks (the port) -->
  <project path="device/motorola/fogos"
           name="android_device_motorola_fogos-6.1" remote="nakchwal" revision="6.1/fogos-port" />
  <project path="kernel/motorola/sm6375"
           name="android_kernel_motorola_sm6375-6.1" remote="nakchwal" revision="6.1/fogos-port" />
  <project path="kernel/motorola/sm6375-devicetrees"
           name="android_kernel_motorola_sm6375-devicetrees" remote="nakchwal" revision="6.1/fogos-port" />

  <!-- rest from the org (6.1 line) -->
  <project path="device/motorola/sm6375-common"  name="android_device_motorola_sm6375-common"  remote="fogos-plygrnd" revision="6.1/lineage-23.1" />
  <project path="kernel/motorola/sm6375-modules" name="android_kernel_motorola_sm6375-modules" remote="fogos-plygrnd" />
  <project path="vendor/motorola/sm6375-common"  name="proprietary_vendor_motorola_sm6375-common" remote="fogos-plygrnd" revision="6.1/lineage-23.1" />
  <project path="vendor/motorola/fogos"           name="proprietary_vendor_motorola_fogos"          remote="fogos-plygrnd" revision="6.1/lineage-23.1" />

  <!-- CAF HALs on the sm6375 (blair) base -->
  <project path="device/qcom/sepolicy_vndr/sm6375" name="android_device_qcom_sepolicy_vndr" remote="fogos-plygrnd" revision="lineage-23.1-caf-sm6375" />
  <project path="hardware/qcom-caf/sm6375/audio/agm"   name="android_vendor_qcom_opensource_agm"     remote="fogos-plygrnd" revision="lineage-23.1-caf-sm6375" />
  <project path="hardware/qcom-caf/sm6375/audio/pal"   name="android_vendor_qcom_opensource_arpal-lx" remote="fogos-plygrnd" revision="lineage-23.1-caf-sm6375" />
  <project path="hardware/qcom-caf/sm6375/audio/primary-hal" name="android_hardware_qcom_audio-ar"  remote="fogos-plygrnd" revision="lineage-23.1-caf-sm6375" />
  <project path="hardware/qcom-caf/sm6375/dataipa"     name="android_vendor_qcom_opensource_dataipa" remote="fogos-plygrnd" revision="lineage-23.1-caf-sm6375" />
  <project path="hardware/qcom-caf/sm6375/data-ipa-cfg-mgr" name="android_vendor_qcom_opensource_data-ipa-cfg-mgr" remote="fogos-plygrnd" revision="lineage-23.1-caf-sm6375" />
  <project path="hardware/qcom-caf/sm6375/display"     name="android_hardware_qcom_display"          remote="fogos-plygrnd" revision="lineage-23.1-caf-sm6375" />
</manifest>
```
(If you have not forked the org's `local_manifests`, the org's own
`6.1/lineage-23.1/manifest.xml` is a complete working set — start from it and
swap the three `nakchwal` entries in.)

### 4. Build
```
source build/envsetup.sh
breakfast fogos          # or: lunch lineage_fogos-userdebug
mka bacon -j$(nproc)     # full ROM; or 'mka bootimage dtboimage vendor_bootimage' for images only
```

### 5. Outputs — `out/target/product/fogos/`
`boot.img`, `init_boot.img`, `vendor_boot.img`, `dtbo.img`, `vbmeta.img`,
`vendor_dlkm.img`, and `lineage-23.1-*-fogos.zip`.

### 6. Flash
```
fastboot flash boot         boot.img
fastboot flash init_boot    init_boot.img
fastboot flash vendor_boot  vendor_boot.img
fastboot flash dtbo         dtbo.img
fastboot --disable-verity --disable-verification flash vbmeta vbmeta.img
fastboot reboot fastboot    # fastbootd for the logical partition
fastboot flash vendor_dlkm  vendor_dlkm.img
fastboot reboot
```
Or `adb sideload lineage-23.1-*-fogos.zip` from recovery.

---

## Route B — standalone kernel (no AOSP framework)

For iterating on the kernel alone. Produces `Image` + DTBs + DLKM modules, which
you then repack into the device images. You still need Google's kernel build
tooling and the matching clang — that is *not* the ROM source.

### 1. Layout
```
mkdir -p ~/blair && cd ~/blair
git clone https://github.com/nakchwalabhi/android_kernel_motorola_sm6375-6.1   msm-kernel -b 6.1/fogos-port
git clone https://android.googlesource.com/kernel/build                        build
git clone https://github.com/nakchwalabhi/android_kernel_motorola_sm6375-devicetrees -b 6.1/fogos-port devicetrees
git clone https://github.com/nakchwalabhi/android_kernel_motorola_sm6375-6.1   modules -b 6.1/fogos-port  # optional (DLKM)
# merge the DTs into the kernel tree:
cp -r devicetrees/qcom msm-kernel/arch/arm64/boot/dts/vendor/
# fetch the prebuilt clang (CLANG_VERSION=r487747c) into:
#   prebuilts/clang/host/linux-x86/clang-r487747c/
#   build/kernel/build-tools/   (dtc etc., from kernel/build's build-tools)
```

### 2. Build
```
ROOT_DIR=$PWD BUILD_CONFIG=msm-kernel/build.config.msm.blair build/build.sh
```
(Also supports Bazel: `msm-kernel/build_with_bazel.py` / `tools/bazel`.)
Outputs: `Image`, `Image.lz4`, `*.dtb`/`*.dtbo`, and the vendor/system DLKM `.ko`.

### 3. Package + flash
Repack the stock images with `magiskboot` (or Android Image Kitchen) — no
`mkbootimg` parameter fiddling:
```
magiskboot unpack stock/boot.img        && magiskboot repack boot.img        new-boot.img
magiskboot unpack stock/vendor_boot.img && magiskboot repack vendor_boot.img new-vendor_boot.img
```
Put the new `Image` + DTBs into boot/vendor_boot, the new `.ko` set into
vendor_dlkm. Then flash as in Route A step 6, and **test with
`fastboot boot new-boot.img` first** (boots to RAM, writes nothing).

---

## Reference implementation — Samsung-QTI gta9pwifi (blair 6.1)

The Galaxy Tab A9+ WiFi (`gta9pwifi`) is the one SM6375 device already running
the blair 6.1 base. Its orgs are a working reference to diff against:

- `Samsung-QTI` — device tree (`device_samsung_gta9pwifi`), the 6.1 `manifest`
  (`blair.xml` + `blair-kernel.xml`), and the kernel build scripts
  (`android_kernel_build`, branch `kernel.lnx.6.1.r46-rel`).
- `Samsung-QTI-Blair` — the blair 6.1 kernel module repos: audio-kernel-ar,
  bt-kernel, camera-kernel, display-drivers, dsp-kernel, graphics-kernel,
  mm-drivers, mmrm-driver, dataipa, data-ipa-cfg-mgr, datarmnet(-ext),
  dataservices, touch-drivers, video-driver, synx-kernel,
  wlan/{fw-api,platform,qca-wifi-host-cmn,qcacld-3.0}, plus
  `android_kernel_platform_build_config` and
  `android_kernel_platform_external_dtc`.

What to take from it:

- **Kernel-build tooling.** `android_kernel_build` is the `kernel/build` piece
  Route B needs; `android_kernel_platform_external_dtc` provides `dtc`.
- **Prebuilt-kernel model.** gta9pwifi builds the kernel as a prebuilt "kernel
  platform" (`TARGET_USES_KERNEL_PLATFORM := true`,
  `KERNEL_PREBUILT_DIR := device/qcom/blair-kernel`) instead of compiling it
  inline like the fogos org — a cleaner separation for Route B.
- **Blair DLKM source set.** A complete cross-check for the fogos modules repo.
- **Boot-image load geometry.** gta9pwifi sets it explicitly
  (`BOARD_KERNEL_OFFSET 0x8000`, `BOARD_RAMDISK_OFFSET 0x02000000`,
  `BOARD_KERNEL_TAGS_OFFSET 0x01e00000`, `BOARD_DTB_OFFSET 0x01f00000`); fogos
  does **not**, in either the 5.4 or the 6.1 tree — it relies on mkbootimg
  defaults and the 5.4 tree booted that way, so this is *probably* fine. If a
  hand-built boot image refuses to boot, compare these values first.
- It also runs `androidboot.selinux=permissive`, i.e. that is normal for a
  blair bring-up, not a fogos-specific bug.

---

## Caveats (both routes)

- **The 6.1 branch is WIP.** The org's own last commits on it read "STOP LOOKING
  AT THIS WIP SHIT". Expect build/boot failures and iterate.
- **Vendor blobs are still the 5.4 set** (`proprietary-files.txt` is byte-identical
  to the `lineage-23.2` branch). Some HALs may misbehave against the 6.1 kernel.
- **SELinux is off** in the 6.1 device trees (sepolicy includes commented out,
  `androidboot.selinux=permissive`). Boots, but not shippable.
- **DT verify list** (from `FOGOS-6.1-PORT.md`): confirm the fogos overlay node
  labels exist in the 6.1 `blair.dtsi`, and reconcile the UFS phy compatible.
- **AVB** is on — always `--disable-verity --disable-verification` for a custom vbmeta.
- Keep the **stock boot/init_boot/vendor_boot/dtbo/vbmeta** dumped before flashing.

## Why this wasn't built in the assistant sandbox

The sandbox is 2 CPU / 2 GB RAM with no compiler toolchain (no clang/gcc/make)
and no sudo — it cannot host an AOSP or kernel build. This kit is written to run
on a real build host.
