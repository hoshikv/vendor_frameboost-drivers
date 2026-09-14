# OPLUS frameboost DLKM for peridot (SM8635 / Redmi Turbo 3)

ColorOS scheduler drivers (`sched_assist`/AFS, `frame_boost`, `qos_sched`,
`sched_tune`, UAG CPUFreq governor + `ua_cpu_ioctl`, `hans`) are shipped on the
OnePlus **SM8635** (`vendor/oplus/kernel/cpu/*`) as out-of-tree scheduler
patches / DLKM modules. On the peridot ColorOS port loading the OnePlus `.ko`
fails against the Xiaomi GKI kernel (vermagic / KMI mismatch). This repo
contains the ported sources that are **built against the exact peridot KMI**
(`6.1.175-android14-11-ga3b9c44908dd-ab13320413`) as loadable modules.

Source: `OnePlusOSS/android_kernel_modules_and_devicetree_oneplus_sm8635`
branch `oneplus/sm8635_b_16.0.0_nord_5`, `vendor/oplus/kernel/cpu/sched/*`
and `vendor/oplus/kernel/cpu/uad*` (GPL-2.0).

## Layout (flat — each module is a top-level folder)

| Folder | Module(s) built | Role |
|---|---|---|
| `eas_opt/` | `oplus_bsp_eas_opt.ko` | Energy/capacity hooks used by the UAG governor |
| `sched_tune/` | `oplus_bsp_schedtune.ko` | sched_tune boosted util/target load (built-in only on OnePlus) |
| `sched_assist/` | `oplus_bsp_sched_assist.ko` | AFS / SchedAssist (sched low-latency, AFS hooks) |
| `frame_boost/` | `oplus_bsp_frame_boost.ko` | Frame boost (AFS-based game/UI boost) |
| `qos_sched/` | `oplus_bsp_qos_sched.ko` | QoS sched plugin |
| `uad/` | `cpufreq_uag.ko`, `ua_cpu_ioctl.ko` | UAG cpufreq governor + userspace ioctl |
| `hans/` | `oplus_hans.ko` | High-frequency alert? HANS spikes/power |

The scheduler root module `sched-walt.ko` comes from the peridot kernel tree
(`kernel/sched/walt`) and must be loaded first.

## Building

Builds are hosted in **`hoshikv/peridot-kernel-build`** (`build.sh`, workflow
`kernel_display_touch_audio.yml`): it clones this repo into a flat `frameboost-drivers`
folder, stages an OPLUS `kernel/oplus_cpu` include tree (symlink farm) so the
OPLUS `#include <../kernel/oplus_cpu/...>` / `"../sched/..."` paths resolve, and
builds each module with `KBUILD_EXTRA_SYMBOLS` against the peridot KMI. There is
no CI in this repository.

Porting notes:

- Modules that were **built-in only** on OnePlus (`sched_tune`, `hans`) get a
  generated `license.c` (`MODULE_LICENSE("GPL")`) folded into the composite
  `<mod>-y` object list so modpost does not reject them.
- `KBUILD_EXTRA_SYMBOLS` uses `walt-extra.symvers` = walt `Module.symvers`
  de-duplicated against `vmlinux.symvers`, because the walt `fixup.o` re-exports
  core GKI symbols (`system_state`, ...) and would otherwise trip modpost's
  `exported twice` check.
- The walt module is named `sched-walt.ko` (dashes, not underscores).

## Install on device

```sh
adb push *.ko /settings/system/startup/ scripts/do_install.sh . 2>/dev/null # or:
adb push *.ko /data/local/tmp/ ; adb shell su -c sh /data/local/tmp/install.sh
# on device:
#   sh install.sh --test      # verify vermagic + insmod in dependency order
#   sh install.sh --persist   # copy into /vendor_dlkm/lib/modules/<kver>/ + modules.load
```

Load order is defined in `install.sh` (`ORDER`): walt first, then eas_opt,
schedtune, sched_assist, frame_boost, qos_sched, uag governor, ua ioctl, hans.

## License

GPL-2.0. Original sources (c) OPPO/OnePlus; ported for peridot.