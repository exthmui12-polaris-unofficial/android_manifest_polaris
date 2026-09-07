# NFC persistence and Android 12 GPU memory accounting

## Result

Polaris NFC cache persistence and native GPU memory accounting passed device
validation with SELinux Enforcing. A complete Vanilla block OTA was built with
CPU affinity 0-11, verified, and prepared for the exthmui-12 repositories.

## Changes

- Legacy Qualcomm NFC policy adds only hal_nfc_default directory
  search/write/add_name and file create for nfc_vendor_data_file. Existing
  pn8x mkdir/restorecon and file access remain.
- Polaris enables KPROBES/KPROBE_EVENT to select BPF_EVENTS, adds the Android 12
  gpu_mem_total tracepoint and KGSL absolute global/process counters. DMA-BUF
  imports are deduplicated globally; each process counts its mappings. Final
  release emits zero for Android's existing BPF cleanup. Sparse VA reservations
  are excluded; no newer driver or BPF subsystem was transplanted.
- A default-off diagnostic kernel parameter preserves real boot flags when
  explicitly requested so the standard Android debug ramdisk can work. The
  production package has no such parameter or debug ramdisk.
- Local manifest removes the overridden duplicate vendor/gapps declaration,
  preserving the existing fixed source and revision.

## Validation

- NFC: 0770 nfc:nfc directory, 0600 nfc:nfc cache, correct labels, 4-byte CRC
  66 2f ba f9. Five toggles, HAL recreation from a missing cache, and another
  boot with a changed boot ID preserve the CRC; no cache AVC/open/read errors.
- GPU: runtime event offsets 8/12/16, loaded BPF program/map and real dumpsys
  totals. Controlled 4/2/1 MiB allocations match sysfs/global increments;
  threads belong to TGID, concurrent processes remain isolated. Duplicate
  DMA-BUF import does not double global usage. Failures, frees and process exit
  leave no stale entries. Statistics return after reboot.
- Rendering: Adreno 630 GLES test runs 600 seconds, 33452 frames/pixel checks
  pass, plus 400 concurrent allocation/free operations. Final global usage
  returns to 177565696 bytes and the renderer PID disappears. No GPU hang/fault
  or kernel BUG was found. Camera preview frames and actual AVC video playback
  were exercised. Temporary media, permissions and test binaries were cleaned;
  ADB returned to shell and SELinux remained Enforcing.
- Memory-focused gpuservice tests pass 9/9. All 22 full-suite assertions pass,
  but its process exits with a libstatspull/Binder LeakSanitizer finding on the
  earlier diagnostic kernel; the full process result is not counted as passing.
- Physical NFC tags and secure GPU allocations remain untested. The rendering
  check is a stability test with 16 ms pacing, not an old/new performance
  benchmark. Existing allocations enter a newly attached consumer's process
  map on the next accounting event; events carry absolute totals.

## Package

- File: `exthm-12.0-20260907-UNOFFICIAL-SNAPSHOT-polaris-Vanilla.zip`
- Size: 1098363841 bytes
- SHA-256: `a36f2de44e279b65a38bee22638892c2c465b35ffa6dbbd36b3b813c0fc77548`
- ZIP CRC, Android whole-file test-key signature, Polaris/BLOCK metadata,
  target-files VINTF and effective packaged NFC policy checks passed.
- Packaged kernel is byte-identical to the device-tested candidate kernel;
  embedded config includes BPF_EVENTS. Diagnostic boot flags and ramdisk files
  are absent. This is an AOSP-test-key unofficial package.

No permanent boot flashing was done during validation. The device remains on
the temporary GPU boot and returns to its previously installed boot on ordinary
reboot until the new package is installed. Detailed local evidence is kept in
artifacts/nfc-gpu-20260907, and server logs in /root/android/nfc-gpu-20260907.
