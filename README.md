# oriole-fart12-build

Builds a `userdebug` AOSP 12.1.0_r2 (SP2A.220305.013.A3) image for **Pixel 6 (oriole)**
with **fart12-lite** patches integrated. Targets `com.mustafahameed.mirmazacademy`
(owner's own published app) for packed/resolved-DEX dump extraction.

Adapted from [ZuoqTr/redfin-fart12-build](https://github.com/ZuoqTr/redfin-fart12-build)
(Pixel 5). Key differences vs upstream:

- `lunch aosp_oriole-userdebug` — the Pixel 6 device tree at this tag is
  `device/google/raviole` (+ `gs101`, `gs101-sepolicy`, `gs-common`,
  `raviole-kernel`), and android-12.1.0_r2 is literally the Pixel 6/6 Pro tag
  (build SP2A.220305.013.A3), so the factory image matches the source exactly.
- Factory bundle: `oriole-sp2a.220305.013.a3-factory-8bea92d1.zip`
  (bootloader-oriole-slider-1.1-8118264 + radio-oriole-g5123b-97927-220225-b).
- Swap is recreated UNCONDITIONALLY at 16GB: 2025+ runner images ship a ~3GB
  `/swapfile`, which silently defeated upstream's Bug-36 "8GB swap" guard and
  was the root cause of their exit-143 OOM kills during soong analysis.
- `product.img` + `system_ext.img` are flashed from the factory build too
  (upstream didn't ship them — coming from an Android 15 install that would
  leave A15 product/system_ext under an A12 system and never boot).
- `f1rt.config` retargeted to the owner's app (FART activates only when
  `packageName == processName` of the launched process).

## What you get

GitHub Release artifact per run:

```
oriole-fart-<sha>.zip
├── system-aosp-fart.img      (CUSTOM — AOSP 12.1.0_r2 with FART12 ART/framework/libcore)
├── boot.img                  (FACTORY)
├── vendor_boot.img           (FACTORY — kernel lives here on A12)
├── vendor.img                (FACTORY)
├── product.img               (FACTORY)
├── system_ext.img            (FACTORY)
├── dtbo.img                  (FACTORY)
├── bootloader-oriole-*.img   (FACTORY — slider-1.1, 12L era)
├── radio-oriole-*.img        (FACTORY)
├── vbmeta.img                (FACTORY — flashed with verity+verification disabled)
├── f1rt.config               (mirmazacademy dump config)
├── flash-all.sh              (one-shot flash wrapper incl. fastboot -w)
└── build.log                 (full m output)
```

## Flash procedure — READ THE ANTI-ROLLBACK WARNING FIRST

Pixel 6 received a bootloader **anti-rollback** in the May 2025 security
update. A device that has booted any mid-2025+ Android 15 bootloader will
refuse to boot this Android 12 build **even with the bootloader unlocked**.
Check `fastboot getvar version-bootloader` before starting; if the bootloader
flash below is rejected, stop and restore stock instead.

```bash
unzip oriole-fart-<sha>.zip -d oriole-fart
cd oriole-fart
# device in fastboot mode (power + vol-down, or adb reboot bootloader)
./flash-all.sh    # flashes bootloader first, ends with fastboot -w (WIPE)
```

After flash, push the FART config and launch the target app:

```bash
adb root
adb push f1rt.config /data/local/tmp/f1rt.config
adb shell chmod 644 /data/local/tmp/f1rt.config
adb shell monkey -p com.mustafahameed.mirmazacademy -c android.intent.category.LAUNCHER 1
adb logcat -s zskkk        # watch FART hooks fire
sleep 60
adb pull /data/data/com.mustafahameed.mirmazacademy/zskkk/
```

Dumped files include `*_dexfile.dex`, `*_classlist.txt`, `*_deep_dexfile.dex`,
`*_dexfile_repair.dex`, `*_dexfile_execute.dex`, `*_classlist_execute.txt`.

## Build pipeline

Three-job GitHub Actions workflow:

| Job | Purpose | Wall-clock |
|---|---|---|
| `sync` | `repo init` against `android-12.1.0_r2`, sync AOSP tree | 10-20 min |
| `build` | Apply FART patches, `lunch aosp_oriole-userdebug`, `m systemimage` | 5-8h cold |
| `assemble` | Download oriole factory blob, build `flash-all.sh`, upload Release | 5-10 min |

**Expect to run the workflow twice.** GitHub-hosted jobs are hard-capped at 6h;
a cold `-j2` systemimage build can exceed that. Run 1 gets as far as it can
(saving ccache + soong bootstrap), run 2 resumes warm and completes.

## Trigger

`workflow_dispatch` only. Manual run from Actions tab. Inputs:

- `aosp_tag` — default `android-12.1.0_r2`

## Local submodule

Pinned to `Zskkk/fart12-lite` commit `7671fe3b95d28162e1b2024262ddf4fd8fe4077b`.

```bash
git clone --recurse-submodules https://github.com/creativexanas/oriole-fart12-build.git
```

To bump the pin:

```bash
cd fart12-lite && git fetch && git checkout <new-sha> && cd ..
git add fart12-lite && git commit -m "bump fart12-lite pin"
```

## Files

- `.github/workflows/build.yml` — 3-job pipeline
- `.github/workflows/pr-lint.yml` — payload + JSON + YAML validation
- `f1rt.config` — mirmazacademy dump config (`isDeep: true`)
- `flash-all.sh.template` — substituted at build time
- `fart12-lite/` — submodule (pinned upstream)

## License

Authorized security research on the owner's own application and device only.
Not for production deployment, not for third-party targets.
