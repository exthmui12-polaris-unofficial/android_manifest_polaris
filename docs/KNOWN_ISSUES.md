# Known Issues

Keep this file short and evidence-based. Record only information useful across future debugging sessions.

## Current

No confirmed open issue is recorded here. Use logs from the latest device test as the source of truth.

## Resolved runtime issues

### Network time did not synchronize

Status: RESOLVED

Symptom: With automatic time enabled and a working default network, the device did not correct its clock.

Evidence: The installed `framework-res.apk` selected `time.android.com` because no SIM MCC was active. `NetworkTimeUpdateService` had no NTP cache, `TimeDetectorStrategy` had no network suggestion, and logcat showed repeated `SocketTimeoutException` failures for every NTP request. Temporarily setting `Settings.Global.NTP_SERVER` to `ntp.aliyun.com` immediately produced an NTP result and corrected the clock from `00:30` to `11:48`.

Cause: The default `time.android.com` endpoint was unreachable on the test network. The existing China-specific fallback only applied under `mcc460`, so it did not help Wi-Fi-only / no-SIM operation; its `ntp.ntsc.ac.cn` endpoint was also unreliable in direct reachability tests.

Fix: `vendor/exthm/overlay/common/frameworks/base/core/res/res/values/config.xml` now sets the global default to `ntp.aliyun.com`, and `values-mcc460/config.xml` uses the same endpoint.

Verification: The incremental `framework-res` build completed successfully with CPUs restricted to `0-11`. The rebuilt APK contains `ntp.aliyun.com` for both the default and `mcc460` configurations. Runtime A/B testing with the same server populated the NTP cache and caused `TimeDetectorStrategy` to set the clock.

## Resolved issues

### Boot blocked by an unsupported VR HAL declaration

Status: RESOLVED IN SOURCE

Symptom: Four DropBox system-server watchdog traces showed the main thread blocked in `VrManagerService.initializeNative()` while waiting for `android.hardware.vr@1.0::IVr::getService()`.

Evidence: The final vendor image did not contain `/vendor/lib64/hw/vr.sdm845.so`, while the inherited PixelExperience SDM845 configuration declared `android.hardware.vr@1.0::IVr/default` and attempted to package the VR implementation, service, and `vr.sdm845` module.

Cause: Polaris does not implement Android's Persistent/Daydream VR HAL. The inherited declaration was stale and made system-server wait for a service that should not exist on this device. This HAL is separate from ordinary sensor-based/Cardboard-style applications.

Fix: Permanently removed the unsupported VR HAL declaration from `device/xiaomi/sdm845-common/manifest.xml` and removed the VR implementation, service, and `vr.sdm845` package entries from `device/xiaomi/sdm845-common/sdm845.mk`. Do not restore the Qualcomm CAF VR repository or these declarations without new device-specific evidence that Polaris requires them.

Verification: The source and merged device manifest no longer require the VR HAL. Runtime boot verification still requires rebuilding the affected image and testing it on the device.

### Settings resources dependency

Status: RESOLVED

`device/xiaomi/sdm845-common/parts/Android.bp` was changed from `org.pixelexperience.settings.resources` to the exTHmUI-provided `org.lineageos.settings.resources`. Do not revert without evidence that the PixelExperience module exists.

### Bluetooth duplicate module and QTI API compatibility

Status: RESOLVED

Polaris retains `TARGET_USE_QTI_BT_STACK := true`. A `soong_namespace {}` was added to `packages/apps/Bluetooth/Android.bp`, and AOSP Bluetooth tests are excluded when the QTI stack is active, preventing AOSP and QTI from both exporting a Make-visible `Bluetooth` module. Required QTI framework/native API alignment was also added in `frameworks/base` and `system/bt`.

Files changed: `packages/apps/Bluetooth/Android.bp`, `packages/apps/Bluetooth/tests/Android.mk`, `device/xiaomi/sdm845-common/BoardConfigCommon.mk`, plus the corresponding QTI API compatibility changes in `frameworks/base` and `system/bt`.

Verification: Bluetooth module builds and the complete ROM build succeeded.

### Duplicate libplatformconfig module

Status: RESOLVED

The incompatible module declaration in `hardware/qcom-caf/sdm845/media/libplatformconfig/Android.mk` is now guarded by `ifneq ($(QCPATH),)`, avoiding the duplicate module in the current Qualcomm source layout.

### Location SELinux neverallow

Status: RESOLVED

Removed `qmux_socket(vendor_location_app)` from `device/qcom/sepolicy_vndr/legacy/vendor/common/location_app.te`, resolving the Android 12 neverallow conflict.

### Kernel toolchain compatibility

Status: RESOLVED

`device/xiaomi/polaris/BoardConfig.mk` now selects `aarch64-linux-gnu-` and `ld.lld`. The obsolete `-enable-trivial-auto-var-init-zero-knowing-it-will-be-removed-from-clang` flag was removed from `kernel/xiaomi/polaris/Makefile`, and the required Prelude Clang files were added under `prebuilts/clang/host/linux-x86/clang-prelude/`.

### XiaomiParts thermal resource

Status: RESOLVED

Added the missing `thermal_summary` resource through `device/xiaomi/sdm845-common/parts/res/values/strings.xml`.

### Orphaned LiveDisplay file context

Status: RESOLVED

Removed the stale file-context entry for the absent SDM845 LiveDisplay service from `device/xiaomi/sdm845-common/sepolicy/vendor/file_contexts`.

### Unexpected `/file_contexts.bin` in target-files

Status: RESOLVED

Running the module-only `m file_contexts.bin -j4` target left `out/target/product/polaris/root/file_contexts.bin`, which target-files then copied into `system.img` even though Android 12 intentionally has no label for `/file_contexts.bin`. Deleting only that generated artifact fixed packaging; no obsolete SELinux label was added.

Future note: Module-only SELinux verification can pollute `TARGET_ROOT_OUT`; check generated root artifacts before changing policy.

## Last successful full build

Artifact: `/root/android/exthmui12/out/target/product/polaris/exthm-12.0-20260828-UNOFFICIAL-SNAPSHOT-polaris-Vanilla.zip`

SHA-256: `b512de6329d2d3cf94a08ba8a75dad9a480aab89dc6a929f3278c1092afd8fb5`

Size: `1,032,381,653` bytes

Build log: `out/build-polaris-13.log` — completed successfully in `05:52`.
