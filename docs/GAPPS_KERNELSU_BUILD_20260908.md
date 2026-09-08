# GApps ROM and standalone KernelSU build — 2026-09-08

Both builds completed successfully on OrangeVPS with CPU affinity `0-11`
and `nproc=12`. No device flashing or runtime testing was performed.

## GApps complete package

- Artifact: `exthm-12.0-20260907-UNOFFICIAL-SNAPSHOT-polaris-Gapps.zip`
- Bytes: `1183705590`
- SHA-256: `c132c260daa0a9970bfc38c8134e763437a8021d0db86c3fea636d1c3e199b9e`
- Build: `EXTHM_GAPPS=true`, `lunch exthm_polaris-userdebug`, `mka bacon`.
- Build exit: `0`; duration: `11:45`.
- The filename retains the build system's 20260907 version date; completion was on September 8.
- ZIP CRC, Android whole-file OTA signature, `polaris`/`BLOCK` metadata,
  target-files VINTF, GmsCore/GSF/Phonesky/SetupWizard/GoogleRestore presence,
  Seedvault absence, packaged NFC policy, GPU BPF configuration, and absence
  of the diagnostic ramdisk/command line passed verification.
- This is an unofficial userdebug package signed with the AOSP test key.

## Standalone KernelSU kernel

- Boot artifact: `exthm-polaris-kernelsu-20260908-boot.img`
- Bytes: `67092480`, matching `BOARD_BOOTIMAGE_PARTITION_SIZE`.
- SHA-256: `eb7fb6368277f7f227694e70af8c1c46157e315d7b1c3328b548b7078429d625`
- Raw payload: `Image.gz-dtb`
- Payload SHA-256: `e7c7a8f266130af090267f24f339e5a95cf9b4af5610b18b4ff08cb869675317`
- Kernel base: `a7cdfb56fa6b72b1b97e39773230bfbdfac0d8ce`.
- KernelSU: `https://github.com/rsuntk/KernelSU`, pinned at
  `648e5988cf421172769f80ce07f86331b548c053`.
- Existing `/root/android/ksu-kernel` worktree fast-forwarded to the current
  GPU-accounting commit; its pre-existing `drivers/Kconfig`, `drivers/Makefile`,
  KernelSU checkout and symlink were preserved. Those existing local changes
  were not committed as part of this task.
- Reused `/root/android/ksu-out`; enabled `KPROBES` and `KPROBE_EVENT` in its
  existing configuration to match the current GPU fix, then ran `olddefconfig`
  and `Image.gz-dtb` with Prelude Clang/LLD and ARM64/ARM32 GNU binutils.
- Build exit: `0`. Embedded `KSU`, `KSU_MANUAL_HOOK`, `KSU_FEATURE_ADBROOT`,
  `KPROBES`, `KPROBE_EVENT`, `BPF_EVENTS`, and `SECURITY_SELINUX` are enabled.
- The boot image uses the new GApps package's exact ramdisk and header/command
  line, replacing only the kernel payload and regenerating its AVB hash footer.
  AVB uses `Algorithm: NONE`, matching the existing userdebug boot image.
- Unpacked payload/ramdisk/header checks and AVB hash verification passed.
  The verifier requires a temporary `boot.img` symlink because AVB resolves
  its hash descriptor by partition name.

## Scope and evidence

No ROM source fix was needed for this build. Only the two user-approved old
generated directories were removed to recover space:
`out/target/product/polaris/obj/PACKAGING/target_files_intermediates` and
`/tmp/exthm-nfc-gpu-build`. Existing ROM ZIPs and other build caches were retained.

Remote evidence and KernelSU artifacts: `/root/android/gapps-kernelsu-20260908/`.
GApps ZIP: `/root/android/exthmui12/out/target/product/polaris/`.
Local artifacts and logs: `artifacts/gapps-kernelsu-20260908/` in the Windows workspace.

The new production kernel was rebuilt from the committed GPU fix and is not
byte-identical to the earlier runtime-tested diagnostic kernel. Prior runtime
results do not constitute runtime verification of these new artifacts.
The standalone KernelSU boot also changes the boot hash relative to the ROM's
vbmeta descriptor; pairing and device startup/root checks remain required
before treating it as device-validated. No partition or vbmeta was modified.
