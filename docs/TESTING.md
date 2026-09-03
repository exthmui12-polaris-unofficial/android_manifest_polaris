# exTHmUI 12 Polaris 功能验证

适用设备：Xiaomi Mi MIX 2S（polaris）

本文只保留可复用的构建、模块和设备验证步骤。未核对设备、分区、制品和授权前，不执行刷机、Recovery 安装或任何分区写入。

## 构建与静态检查

```bash
cd /root/android/exthmui12
source build/envsetup.sh
lunch exthm_polaris-userdebug
taskset -c 0-11 m SettingsProviderTest UpdaterCoreTests Settings SystemUI Updater
```

完整构建使用 `taskset -c 0-11 mka bacon`。保留 `out/`，失败时记录最早有意义的错误，只修复一个可验证原因后再重建。

检查项：

- `git status`、各仓库 `git diff --check` 和提交范围；
- Updater 单元测试、APK 构建、HTTPS/签名/metadata/设备/非 A/B 校验；
- SettingsProvider version 205 的一次性 Secure→System 迁移；
- Settings/SystemUI 只读 System namespace，callback 只反注册成功注册的对象；
- SELinux、VINTF、ZIP 完整性、Android whole-file 签名和最终制品 SHA-256。

## Updater 受控 HTTPS 测试

使用只绑定测试网段的 HTTPS 服务和匹配签名的 Polaris 非 A/B OTA，逐项验证：

1. 默认无源、保存/清除源、非法 URL、HTTPS 重定向和 API key/主机限制。
2. 正常 Content-Length、无 Content-Length 的 chunked feed/OTA、短文件和超过 feed/预期尺寸的文件。
3. 暂停、断点恢复、错误 206/Content-Range/响应长度、取消、服务重启和网络恢复。
4. A→B→A 源切换；旧响应、旧重试和旧 callback 不得覆盖新源数据。
5. 最终验证 OTA 签名、`pre-device=polaris`、`ota-type=BLOCK`、无 `payload.bin`，安装前再次验证。

不执行未经确认的 Recovery 安装；不擦除 persist、modem、校准数据或 bootloader 关键分区。

## Settings/SystemUI 测试

- `SettingsProviderTest`：有效 17 键迁移、System 值优先、非法值清理、缺失值和多用户隔离；确认重启后不再从 Secure 回读。
- `ExthmSettingsRoboTests`：布尔、枚举、范围设置的默认值、合法值持久化、非法数字的默认回退，以及只写 System。
- 受影响的 SystemUI focused tests：TunerService、advanced reboot 和设置 observer；确认只监听 System。
- Hotspot callback：分别让 SoftAP/tethering 注册成功或抛出 Binder/runtime 异常；失败路径不得反注册未成功注册的 callback，成功注册后仍正常清理。

## 设备基线

首次开机后保存：

```bash
adb wait-for-device
adb shell getprop ro.build.fingerprint
adb shell getenforce
adb logcat -b all -d > logcat-initial.txt
adb shell dumpsys > dumpsys-initial.txt
```

必须保持 SELinux Enforcing、加密、RIL、蓝牙、相机和其他 HAL 功能；只读检查设置和服务状态，不批量覆盖数据库。

## 复用型硬件/功能矩阵

- NFC：检查 `/data/vendor/nfc` 的创建、owner/mode/label、配置持久化和重启。
- WFD：检查实际 APK/native 库、服务启动、Sink 发现、连接、音视频、HDCP、断开重连。
- Face Unlock：检查 feature/provider、弱强度、最多一个模板、录入/删除/重录、`BIOMETRIC_WEAK`、凭据回退和用户隔离。
- OmniJaws：检查默认关闭、权限、MET Norway/手动城市、HTTPS、失败保留、锁屏 compact/slice/Smartspace、widget 和用户切换。
- Seedvault：检查 Vanilla/GApps transport、升级保留用户选择、Provision 后备份/恢复；只用可丢弃数据。
- 状态栏/QS/电源/截图/Adaptive Playback：每项验证默认值、合法范围、重启、SystemUI 重启、旋转和用户切换。

每个失败只记录首个有意义的日志、复现条件、影响范围和下一步；未复现的问题不添加猜测性修复。

## 最近一次完整包

路径：`/root/android/exthmui12/out/target/product/polaris/exthm-12.0-20260903-UNOFFICIAL-SNAPSHOT-polaris-Vanilla.zip`

大小：`1,104,437,513` bytes

SHA-256：`fa75714cfb8c4b77cec66381e8131950bbeb5f71a14be295cfc94e623b10cd31`
