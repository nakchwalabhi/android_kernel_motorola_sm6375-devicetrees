# fogos on blair 6.1 — Device Tree Port Bundle

Everything here is derived from the two trees I audited:

- **source (working 5.4):** `Fogos-Playground/android_kernel_motorola_sm6375` @ `lineage-23.2`
  → `arch/arm64/boot/dts/vendor/qcom/` (fogos DTs already exist there, under `blair-*` names)
- **target (6.1):** `Fogos-Playground/android_kernel_motorola_sm6375-devicetrees` @ `lineage-23.1`
  → `qcom/` (blair DTs; `blair-malmo-*` is the working reference port)

The good news: the 5.4 tree already has a **complete fogos DT set under `blair-*`
naming**, and the 6.1 blair DT repo uses the same `blair` base. So this is a
**move/retarget**, not a rewrite. The files below are the result.

---

## Where each file goes

Drop into `android_kernel_motorola_sm6375-devicetrees` (`lineage-23.1`):

| File in this bundle | Destination in repo |
|---|---|
| `qcom/blair-moto-fogos-base.dts` | `qcom/blair-moto-fogos-base.dts` |
| `qcom/blair-fogos-evt-overlay.dts` | `qcom/blair-fogos-evt-overlay.dts` |
| `qcom/blair-fogos-dvt1-overlay.dts` | `qcom/blair-fogos-dvt1-overlay.dts` |
| `qcom/blair-pm7250b-fogos-common-overlay.dtsi` | `qcom/` (same name) |
| `qcom/blair-fogos-common-overlay.dtsi` | `qcom/` (same name) |
| `qcom/blair-fogos-fmr-overlay.dtsi` | `qcom/` (same name) |
| `qcom/fogos-charger.dtsi` | `qcom/` |
| `qcom/fogos-display.dtsi` | `qcom/` |
| `qcom/fogos-touchscreen.dtsi` | `qcom/` |
| `qcom/fogos-sensors.dtsi` | `qcom/` |
| `qcom/fogos-fps-overlay.dtsi` | `qcom/` |
| `qcom/fogos-wl286xx.dtsi` | `qcom/` |
| `qcom/blair-thermal-overlay-pm6125.dtsi` | `qcom/` |
| `qcom/display/blair-sde-display-mtp-pm6125.dtsi` | `qcom/display/` |
| `qcom/audio/blair-audio-moto-fogos-common.dtsi` | `qcom/audio/` |
| `qcom/camera/blair-camera-sensor-mot-fogos-overlay.dtsi` | `qcom/camera/` |
| `Makefile.fogos-snippet` | paste the block into `qcom/Makefile` |

Drop into `android_device_motorola_fogos` (`6.1/lineage-23.1`):

| File | Destination |
|---|---|
| `configs/moto-blair-fogos.config` | `arch/arm64/configs/vendor/ext_config/moto-blair-fogos.config` in the **kernel** repo, replacing the 1-line stub |
| `modules.load` | `modules.load` (device tree) |

---

## File mapping (5.4 → 6.1) and what changed

| 5.4 source | 6.1 target | change |
|---|---|---|
| `blair-moto-fogos-base.dts` | `blair-moto-fogos-base.dts` | `#include "blair.dtsi"` → `"blair-moto-common-base.dtsi"`; kept the `mdss_dsi_phy0` tweak |
| `blair-fogos-evt-overlay.dts` / `-dvt1` | same names | retargeted include to the pm7250b fogos overlay |
| `blair-pm6125-fogos-common-overlay.dtsi` | `blair-pm7250b-fogos-common-overlay.dtsi` | renamed; includes `blair-pm7250b-moto-common-overlay.dtsi` (the 6.1 equivalent) |
| `blair-fogos-common-overlay.dtsi` | same name | **include lines retargeted** (`holi-*` → `blair-*`); body preserved |
| `holi-audio-fogos-overlay.dtsi` | `audio/blair-audio-moto-fogos-common.dtsi` | verbatim (only `dt-bindings` includes) |
| `camera/blair-camera-sensor-mot-fogos-overlay.dtsi` | same path | verbatim (only `dt-bindings` includes) |
| `holi-thermal-overlay-pm6125.dtsi` | `blair-thermal-overlay-pm6125.dtsi` | verbatim (only a `dt-bindings` include) |
| `display/holi-sde-display-mtp-pm6125.dtsi` | `display/blair-sde-display-mtp-pm6125.dtsi` | include retargeted to `blair-sde-display.dtsi` |
| `fogos-*.dtsi`, `blair-fogos-fmr-overlay.dtsi` | same names | verbatim |

---

## Verified vs needs-work

**Structurally verified (safe to start from):**
- The base DTB and the two plugin overlays — correct `model`/`compatible`/`msm-id`
  (`507`/`578`) / `board-id`, matching the 5.4 fogos values.
- The pm7250b fogos overlay: the 5.4 `blair-pm6125-moto-common-overlay.dtsi` is
  effectively identical to the 6.1 `blair-pm7250b-moto-common-overlay.dtsi`
  (both configure PM7250B), so this layer ports cleanly.
- Audio and camera fogos overlays — no repo-internal includes, copy as-is.
- The Makefile block — mirrors the working `CONFIG_MALMO_DTB` block exactly.

**Needs your eyes before it compiles (I could not build/flash):**
1. **Node labels.** `blair-fogos-common-overlay.dtsi` overrides ~18 nodes by
   label (`wcd937x_codec`, `pm7250b_vib`, `sdhc_1/2`, `ufsphy_mem`, `ufshc_mem`,
   `pmk8350_vadc`, `pmk8350_adc_tm`, `pmr735a_*`, `qupv3_se7_i2c`,
   `pm7250b_vadc`, `pm7250b_adc_tm`, `thermal_zones`, `usb2_phy0`, …). Confirm each
   exists with the same name in the 6.1 `blair.dtsi` / `blair-moto-common-base.dtsi`.
2. **UFS phy compatible.** The ported overlay still says
   `qcom,ufs-phy-qmp-v4-yupik` (5.4); the 6.1 base uses `...-v4-blair`. Reconcile.
3. **Display include path.** 6.1 restructured `qcom/display/` (there is a nested
   `display/display/`). Fix the `blair-sde-display*.dtsi` include path.
4. **PMIC split.** fogos references `pm6125_adc_tm` / `pm6125_adc_tm_iio` in the
   thermal zones (via the pm6125 overlays). Confirm the 6.1 blair pm6125 path
   (`blair-pmic-overlay-pm6125.dtsi`, `blair-regulators-pm6125.dtsi`) provides them.
5. **moto-blair-fogos.config.** On 5.4 most fogos drivers were in-tree; on blair
   6.1 they are DLKMs. Only the truly in-kernel symbols belong in the kernel
   fragment — the rest must be selected by the `sm6375-modules` Kconfigs. The file
   lists both groups and flags the DLKM ones.
6. **modules.load.** Reconciled = `sm6375-common/modules.load.vendor_dlkm`
   (authoritative for blair) + 19 fogos-only modules appended
   (`fpc1020_mmi`, `chipone_tddi_v2_mmi`, `ilitek_v3_mmi`, `sx937x_sar`… wait —
   see the appended block). Review the tail; drop anything the common list covers.

---

## How to wire it (order)

1. Copy all DT files above into the devicetrees repo `lineage-23.1`.
2. Paste `Makefile.fogos-snippet` into `qcom/Makefile` next to the malmo block.
3. Replace the 1-line `moto-blair-fogos.config` in the kernel repo with
   `configs/moto-blair-fogos.config`.
4. Replace `modules.load` in the fogos device tree.
5. Build just the DTBs first:
   `m CONFIG_FOGOS_DTB=y` then check `out/.../dtbs/qcom/blair-moto-fogos-base.dtb`
   and the two `.dtbo` overlays are produced.
6. Boot and read `dmesg`; fix the label/compatible mismatches from the list above
   one at a time.

The remaining blockers outside the DT (blob re-extraction, re-enabling SELinux,
touch HAL) are covered in `fogos-6.1-bringup-plan.md`.
