# Known Issues

Keep this file short and evidence-based. Record only information useful across future debugging sessions.

## Current

### Stage 6 non-A/B Updater

Status: IMPLEMENTED IN SOURCE, VERIFIED IN THE SYSTEM IMAGE, AND INCLUDED IN A COMPLETE FLASHABLE PACKAGE; controlled runtime feed, download, and recovery-install verification pending

Evidence: The former ArrowOS updater mixed obsolete mirror scraping, changelog handling, A/B installation paths, and a hard-coded branded source with the Polaris recovery-OTA flow. The Stage 6 implementation keeps the platform-signed `org.lineageos.updater` package and existing `updater_app` domain, but now uses an empty default source and accepts only a configured HTTPS feed. URL templates, redirects, feed entries, download operations, persisted source identity, package size, OTA signature, device metadata, non-A/B package shape, and compatibility archives are validated fail-closed. Polaris installation remains `RecoverySystem.installPackage()` only; there is no UpdateEngine fallback or direct reboot path.

Fix: Added stable source-identity and per-operation download handling, DB v2 metadata, stale-source cleanup, cancellation-safe HTTP and verification paths, final pre-install revalidation, synchronous pending-install persistence, wake-lock reference counting, and orphan-file cleanup. The download directory is `/data/exthm_updates`, created by init and labeled `ota_package_file`; obsolete Updater-to-UpdateEngine SELinux access was removed. Arrow/Mirror/SourceForge/changelog/jsoup/A-B UI and implementation remnants were removed. The obsolete database-injection helper `push-update.sh` was deleted because it bypassed the current verification and source-identity model.

Verification: All CPU-intensive builds used affinity `0-11` and reported `nproc=12`, with a private build `/tmp`. `m UpdaterCoreTests` succeeded and the complete five-class host JUnit suite passed all 29 tests. `m Updater`, `m selinux_policy`, and `m systemimage` completed successfully. The incremental `system.img` is `1,696,211,536` bytes with SHA-256 `6362265ea6dff79fb63980134d6a820503b4a82bcbbe3815acdf74dd48ec0d95`; read-only ext4 checking passed. The image and final target-files contain Updater APK SHA-256 `ac393abcddcd55d4420552f1cb6aae958238248db8ec1446b9b0b2396b8d78a4`, signed by the platform certificate SHA-256 `c8a2e9bccf597c2fb6dc66bee293fc13f2fc47ec77bc6b2b0d52c11f51192ab8`, the empty `updater_server_url`, the privapp allowlist, init creation/restorecon rules, the `updater_app` seapp mapping, and compiled `/data/exthm_updates -> ota_package_file` lookup. The complete block OTA is `1,104,672,244` bytes with SHA-256 `fe8fa8f96e901f476eab7ed4266edbda20df7337b717234ae705623b8d80ae96`; ZIP integrity and Android whole-file signature verification passed, metadata reports `pre-device=polaris` and `ota-type=BLOCK`, `payload.bin` is absent, and explicit target-files VINTF verification returned `COMPATIBLE`. This package contains zero `compatibility.zip` entries, which is valid because build-time target-files VINTF checking passed.

Runtime limitation: The ROM default intentionally has no OTA source, so it must not contact a server or schedule checks until a source is configured. No device, hotspot, Recovery, partition, or installation action was performed in Stage 6. Runtime testing still needs a controlled HTTPS feed and a matching signed Polaris package to cover source save/clear, invalid URLs, feed changes, pause/resume/cancel/replacement, verification failures, reboot persistence, and the user-authorized recovery handoff. The current unofficial userdebug OTA is signed with the AOSP test key and is not a release-key distribution package.

### Stage 5 Seedvault backup transport

Status: IMPLEMENTED IN SOURCE AND INCREMENTAL SYSTEM IMAGE; device backup, migration, and restore verification pending

Evidence: Vanilla previously had no Seedvault source or package while its common SettingsProvider overlay selected the unavailable GMS backup transport. The Stage 5 product now pins LineageOS Android 12 Seedvault at `c47aaf20183e4d5a14ba32c5048a8f9e342e15bd`, packages it only when `EXTHM_GAPPS != true`, and selects the Seedvault transport for Vanilla. The GApps branch retains the GMS transport and does not package Seedvault.

Fix: Added mutually exclusive Vanilla/GApps backup overlays, enabled Seedvault's post-Provision manual restore entry, and added an unexported Settings `BOOT_COMPLETED` receiver. Migration only runs when the current transport is empty or is the stale GMS transport while GMS is absent. It preserves LocalTransport, Seedvault, installed-GMS, and all other non-empty user choices. The receiver uses Android 12's component-based asynchronous BackupManager API so system_server binds and registers the Seedvault transport before selecting it, avoiding the delayed-registration boot race.

Verification: All CPU-intensive builds used affinity `0-11` and reported `nproc=12`. `m Seedvault`, `m Settings`, the focused `RunExthmSettingsRoboTests`, and `m systemimage` completed successfully. The focused suite passes 10 tests covering the decision matrix, GMS package presence, component selection, and repeated-run idempotency. The final `system.img` is `1,692,508,752` bytes with SHA-256 `cbe8256044718f0a81ddf7c7b826063efc2837d87e1c9c2bbb5e6412d5d471f4`. Files extracted from the image match staging: Seedvault `220b4c71d2b67cbdeeb86b46127084d1f0d46bb60a6e5ee4755c8e4978594c76`, Settings `4d3c5f870081e0d4384635a149ce203d29e2e466671a57f4730ab783ca438c1c`, SettingsProvider `d8f24efbfab438c832f09fbf8909278e4267cc6477d30d9120db33e54eb118b6`, and LocalContactsBackup `3a01d7f7ad0f93305e9bc393b98d1441ad9b56ca9e0e42a28b82539056b6ee94`. The image contains the transport whitelist, privapp permissions, `show_restore_in_settings=true`, and the exact Seedvault default transport.

Runtime limitation: GApps behavior is statically verified but was not built in this Vanilla pass. Device testing still must confirm fresh-install defaults, upgrade migration, preservation of user-selected transports, actual backup, and manual restore after Provision. Seedvault's post-setup restore can overwrite active app data, so use disposable test data.

### Stage 4 OmniJaws weather integration

Status: IMPLEMENTED IN SOURCE AND INCREMENTAL SYSTEM IMAGE; device runtime and visual verification pending

Evidence: The product previously retained `WeatherIcons` but did not have a complete Android 12 weather provider, Settings entry, or lockscreen consumer. The Stage 4 implementation now packages a hardened Corvus Android 12 OmniJaws baseline, exposes explicit Settings entries, and provides mutually exclusive compact-row and KeyguardSlice lockscreen presentations. When Smartspace is active, the legacy slice is filtered to the exact OmniJaws weather URI so date, alarm, media, DND, and action rows are not duplicated.

Fix: Added `packages/services/OmniJaws` with service and lockscreen weather disabled by default, MET Norway as the default provider, user-supplied OpenWeatherMap API keys only, HTTPS-only network policy, Android 12 location handling, signature-protected provider/control surfaces, and a manually addable resizable widget. Settings now links to the weather service and lockscreen weather pages. SystemUI reads weather through `org.omnirom.omnijaws.READ_WEATHER`, keeps compact and slice modes mutually exclusive, and retains Smartspace. Removed two unreferenced Lawnchair permission/hidden-API XML remnants from `vendor/exthm`.

Verification: All CPU-intensive builds used affinity `0-11` and reported `nproc=12`. `m OmniJaws`, `m Settings`, `m SystemUI`, `m WeatherIcons`, and `m systemimage` completed successfully. The final incremental `system.img` is `1,676,751,440` bytes with SHA-256 `10b5306d5b10e4a27a9ceb6dc4e525a08cbf68cbde7d38c949ac1bb0bb52aa8f`. APKs extracted from that image match the module outputs: OmniJaws `dcd092d361e3ba2db6389004393cb2c79854d831d1f887260429b3629ce96676`, Settings `022c527b58b534f7fb3cc2416b012290c9fae8f571a5d611ad15d9d0dc6f9be8`, SystemUI `0838e353da2201e3304554c7094e586097221579b644300f5a4704182983e6b0`, and WeatherIcons `67ab99da4ca94180da6b65a2c3e6195149859a7ffb91bbf7653daf8ac49c1968`. The image contains no Lawnchair permission or hidden-API XML.

Test limitation: The new SystemUI tests reached Java compilation after correcting the weather test's Mockito `aryEq` import. The complete `SystemUI-tests` module still fails on 20 unrelated pre-existing test/source API mismatches; those baseline tests were not modified as part of Stage 4. Runtime verification is still required for permission flows, automatic/manual location, provider updates, widget behavior, Smartspace on/off, compact/slice switching, no-data states, AOD, large clock, notification positioning, reboot, and user switching.

### WFD service package required a nonexistent native shared library

Status: FIXED IN SOURCE AND INCREMENTAL SYSTEM IMAGE; end-to-end Miracast verification deferred for the current pass

Evidence: PackageManager logged `com.qualcomm.wfd.service requires unavailable shared library .WfdService`. The original APK manifest declared a required native library named `com.qualcomm.wfd.service.WfdService`, while the implementation actually loads `wfdnative` and the Qualcomm WFD native stack is present. A platform-signed candidate with only that compiled manifest element removed is accepted as an updated system app on the test device.

Fix: Replaced only `vendor/xiaomi/sdm845-common/proprietary/system_ext/priv-app/WfdService/WfdService.apk` and updated its SHA-1 in `device/xiaomi/sdm845-common/proprietary-files.txt`. The input blob SHA-1 is `6c72bec6f59f1ef7588764173e13edafd1a42cf1`; Soong's platform-resigned installed APK SHA-1 is `933b3878381107573daeafd5036682ef7889460a`.

Verification: CPU affinity `0-11` reported 12 processors. `m WfdService` and the effective image target `m systemimage` completed successfully. Polaris nests `system_ext` inside `system.img`, so `systemextimage` alone is not an effective packaging check. The active platform-signed candidate was accepted after reboot, the framework started and stopped Wi-Fi Display scans, and the Qualcomm WFD services were running. Real-sink discovery/connection, audio/video, HDCP, rotation, disconnect, reconnect, and sink persistence were explicitly skipped for this pass and are not considered verified.

### Polaris software Face Unlock

Status: IMPLEMENTED IN SOURCE AND INCREMENTAL SYSTEM IMAGE; runtime enrollment and unlock verification pending

Evidence: The previous image declared `android.hardware.biometrics.face` and `ro.face_unlock_service.enabled=true` but contained neither a face provider nor `FaceUnlockService`. PixelExperience Android 12 supports Polaris with a software provider rather than a Face HAL.

Fix: Integrated pinned PixelExperience Android 12 `FaceUnlockService` and `external/faceunlock` sources, added the framework `CustomFaceProvider`, and adapted the complete Settings enrollment, deletion, redo, and app-auth flows. The synthetic sensor uses ID `1008`, maximum enrollment count `1`, and `STRENGTH_WEAK`. The temporary Polaris `TARGET_FACE_UNLOCK_SUPPORTED := false` override was removed, so the existing exTHm product block again packages the service, property, and feature XML. No Face HAL or VINTF declaration was added.

Verification: All builds ran with affinity `0-11` and reported `nproc=12`. `m FaceUnlockService`, `m services.core`, `m Settings`, `m SystemUI`, and `m systemimage` completed successfully. `get_build_var TARGET_FACE_UNLOCK_SUPPORTED` returned `true`. The unpacked sparse `system.img` contains the platform-signed FaceUnlockService APK, face feature XML, privapp and hidden-api files, all required arm64 ArcSoft/Megvii libraries, updated `services.jar`, Settings and SystemUI, and `ro.face_unlock_service.enabled=true`. The incremental `system.img` SHA-256 is `5cea286dcafac7108929d4884d6a709488ba45e79f638058888d45d391376175`.

Limitations: This is RGB-camera software 2D recognition, not TEE-backed or payment-grade biometrics. It must remain `STRENGTH_WEAK` and may be susceptible to photo/video spoofing. The bundled ArcSoft/Megvii binaries and models have no clear public redistribution grant, so they were pinned to an upstream archive rather than republished to the project GitHub organization. Device-side enrollment, lockscreen, `BIOMETRIC_WEAK` app authentication, camera contention, reboot, and user-switch tests remain required.

### Five-item runtime bring-up status (2026-08-29)

#### NFC vendor data directory

Status: FIXED IN SOURCE; runtime verification pending the rebuilt image

Cause: The `android.hardware.nfc@1.2-service` module is defined by `hardware/nxp/nfc/pn8x/Android.bp`; its init rc did not create `/data/vendor/nfc`, so NXP could not persist `libnfc-nxpConfigState.bin`.

Fix: `hardware/nxp/nfc/pn8x/1.2/android.hardware.nfc@1.2-service.rc` now creates `/data/vendor/nfc` idempotently as `0770 nfc:nfc` during `post-fs-data` and runs `restorecon_recursive`. The similarly named vendor/pn5xx rc is not used by this module and was left unchanged.

Verification: The rebuilt vendor staging tree contains the new init rules. Device-side verification remains: directory ownership/mode/label, config file read/write, and persistence across NFC toggles and reboot.

#### VR high-performance feature declaration

Status: FIXED IN SOURCE; runtime verification pending the rebuilt image

Cause: `device/xiaomi/sdm845-common/sdm845.mk` copied `android.hardware.vr.high_performance.xml` even though the Polaris VR HAL and `vr.sdm845` module had already been removed.

Fix: Removed only that `PRODUCT_COPY_FILES` entry. The generic framework XML remains intact, and `android.software.vr.mode` was not changed.

Verification: The rebuilt vendor staging tree no longer contains `vendor/etc/permissions/android.hardware.vr.high_performance.xml`; source searches show no Polaris VR HAL/module reference.

#### BootReceiver tracefs instance

Status: FIXED IN SOURCE; runtime verification pending the rebuilt boot image

Cause: Polaris reports kernel version 4.9, for which the generic init block sets `bootreceiver.enable=0`; BootReceiver still opens the trace pipe unconditionally. The legacy kernel also lacks the `error_report/error_report_end` tracepoint.

Fix: `system/core/rootdir/init.rc` now creates `/sys/kernel/tracing/instances/bootreceiver` for `ro.kernel.version=4.9`, sets the existing AOSP buffer/options, and deliberately leaves the nonexistent event disabled.

Verification: The rebuilt generated init rc contains the 4.9 block. Runtime checks must confirm the instance exists before BootReceiver runs and that the system-server WTF is gone.

#### `svc bluetooth` command path

Status: NOT REPRODUCED; debug-only impact remains possible

Evidence: With the screen awake and SELinux Enforcing, Settings' Bluetooth page toggled off/on successfully; two `svc bluetooth disable/enable` cycles exited 0, returned the adapter to ON, and reported `Bluetooth crashed 0 times`. No `SyncNotedAppOp`, fatal exception, or AppOps verifier error appeared in the fresh log buffer. The earlier unattended report's CLI NPE therefore remains intermittent or image-state dependent.

Decision: No framework/AppOps change was made without a reproducible public-path failure. Repeat the matrix on the rebuilt image if the CLI crash returns.

#### GPU memory eBPF statistics

Status: KERNEL CAPABILITY LIMITATION (statistics only)

Evidence: `dumpsys gpu` still reports `Failed to initialize GPU memory eBPF`; the pinned map exists but `prog_gpu_mem_tracepoint_gpu_mem_gpu_mem_total` and the `gpu_mem/gpu_mem_total` tracepoint are absent. The running kernel has `CONFIG_BPF=y`, `CONFIG_BPF_SYSCALL=y`, `CONFIG_TRACEPOINTS=y`, and `CONFIG_QCOM_KGSL=y`, but `kernel/xiaomi/polaris/drivers/gpu/msm/kgsl_trace.h` wraps its KGSL tracepoint definitions in `#if 0` and does not provide the Android 12 `gpu_mem_total` ABI.

Decision: No speculative kernel/framework/SELinux change was made and graphics functionality remains enabled. Treat GPU memory accounting as unavailable unless a compatible tracepoint implementation is deliberately ported and validated.

## Resolved runtime issues

### Hotspot client management and Internet forwarding

Status: VERIFIED WITH A ROOTED ANDROID CLIENT; DO NOT REPEAT WITHOUT NEW REGRESSION EVIDENCE

Evidence: On 2026-08-31, rooted Android 14 client `chopin` associated to the Polaris WPA2 hotspot, completed DHCP and private-DNS validation, and Android marked the network `VALIDATED`. Direct traffic over the hotspot passed sustained Baidu HTTP 200 and MIUI connectivity HTTP 204 probes; public DNS and upstream-router ICMP had zero loss. Polaris used `wlan0` upstream and `wlan1` downstream with forwarding, NAT, BPF maps, and hardware offload active. No SoftAP, DHCP, client-blocking, upstream-selection, forwarding, DNS, private-DNS, or VPN-path failure was reproduced.

Cleanup: The hotspot was disabled after testing, Polaris returned to `1203_5G`, the temporary client network was forgotten, Wi-Fi power save and tethering offload defaults were restored, and SELinux remained Enforcing. Windows WLAN was deliberately excluded. Full evidence is in `artifacts/stage2/hotspot-recheck-android-client-20260831.md`.

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

Artifact: `/root/android/exthmui12/out/target/product/polaris/exthm-12.0-20260831-UNOFFICIAL-SNAPSHOT-polaris-Vanilla.zip`

SHA-256: `fe8fa8f96e901f476eab7ed4266edbda20df7337b717234ae705623b8d80ae96`

Size: `1,104,672,244` bytes

Build log: `out/stage6-bacon-build.log` — completed successfully in `07:17` with CPU affinity `0-11` and `nproc=12`.
