# Known Issues

本文件只记录跨后续调试仍有价值的问题、关键证据、当前结论和下一步；不记录中间制品哈希、重复构建参数或完整执行过程。

## 当前

### 代码收敛优化

Problem: 本轮正在收敛 Updater 的下载边界、Settings 的旧 Secure 兼容层、SystemUI/Settings 的重复防御逻辑，以及重复测试和流程性文档。

Evidence: UpdaterCoreTests 通过 28 项，`m Updater` 通过；SettingsProviderTest 构建通过。SettingsProvider 已升至 version 205，17 个旧键使用 Android 12 的 SystemSettingsValidators 做一次性迁移。

Conclusion: HTTPS、签名、metadata、设备校验、非 A/B 安装、OmniJaws 主机/API key/重定向保护均保留；尚未执行最终完整 ROM 构建和设备运行验证。

Next step: 完成 Settings、SystemUI、Updater 模块验证，再以 CPU affinity `0-11` 构建并检查最终包。

### Updater 受控运行验证未完成

Problem: 代码已支持无 `Content-Length` 的普通 chunked 下载，但尚未在受控 HTTPS 服务上验证源切换、暂停恢复、取消和替换竞态。

Evidence: 普通下载仍按 feed 上限和 OTA 预期大小流式限制，最终文件必须精确匹配；断点续传仍强制 206、精确 Content-Range 和响应长度。

Conclusion: 静态检查和单元测试覆盖了边界，设备侧网络行为仍未确认。

Next step: 使用匹配签名的 Polaris 非 A/B OTA 做 HTTPS feed、超限、短文件、暂停恢复、取消和 A→B→A 源切换测试。

### 设备运行验证仍未完成

Problem: 近期 bring-up 修复尚有 NFC 数据目录、BootReceiver tracefs、WFD、Face Unlock、OmniJaws、Seedvault 和锁屏/状态栏等设备侧验证未完成。

Evidence: 相关模块或增量 system image 已能构建；GPU memory eBPF 统计仍因 Polaris 4.9 内核缺少 Android 12 `gpu_mem_total` tracepoint 而不可用。

Conclusion: 未发现需要通过关闭 SELinux、加密、RIL、蓝牙、相机或 HAL 来绕过问题的证据；GPU 统计不影响图形功能。

Next step: 只在准确制品、设备和测试授权均确认后，按 TESTING.md 执行运行验证；不得直接刷写或清除设备唯一分区。

## 已解决或暂缓

### Seedvault transport

Problem: Vanilla 版本原先选择不可用的 GMS transport。

Evidence: Seedvault 仅在 Vanilla 打包，GApps 保留 GMS；迁移逻辑保留 LocalTransport、Seedvault、已安装 GMS 和其他非空选择。

Conclusion: 源码和增量镜像已验证，真实备份、升级迁移和恢复仍待设备验证。

Next step: 使用可丢弃测试数据完成 Provision 后的备份/恢复和多用户升级矩阵。

### OmniJaws weather

Problem: 原有天气入口、provider 和锁屏消费者不完整。

Evidence: Android 12 provider、显式 Settings 入口、HTTPS-only 网络策略、权限保护和 compact/slice 互斥逻辑已构建。

Conclusion: 源码边界已收敛，定位、provider、Smartspace、widget 和视觉布局尚未真机确认。

Next step: 覆盖默认关闭、权限拒绝、手动/自动城市、网络失败、Smartspace、AOD、重启和用户切换。

### WFD service

Problem: 原 APK 声明了实际不存在的必需 native shared library。

Evidence: 删除错误 manifest 声明后 PackageManager 可加载服务，WFD 服务可启动并停止扫描。

Conclusion: 包加载问题已解决；真实 Sink 发现、连接、HDCP、重连仍未验证。

Next step: 使用已知 Miracast Sink 做连接、断开、重启和受保护内容测试。

### 软件 Face Unlock

Problem: 产品声明了 face feature，但缺少 provider 和服务实现。

Evidence: PixelExperience Android 12 软件 provider 已集成，sensor ID 为 1008，强度为 `STRENGTH_WEAK`，未添加 Face HAL/VINTF。

Conclusion: 构建和静态内容已验证；录入、解锁、应用认证和多用户隔离仍待设备验证。

Next step: 仅按弱 2D 生物识别验证，不宣称支付级或抗照片/视频攻击能力。

### 其他 bring-up 结论

Problem: 旧配置还包含不适用于 Polaris 的 VR HAL、缺失 NFC 目录、4.9 内核 BootReceiver tracefs 假设、重复 Bluetooth/libplatformconfig 模块和位置 SELinux 冲突。

Evidence: 对应源码修复已构建；`svc bluetooth` 当前未能稳定复现早期 CLI 异常；GPU eBPF 缺陷已定位到内核 tracepoint 能力。

Conclusion: VR HAL、NFC 目录、BootReceiver、Bluetooth 构建冲突、libplatformconfig、位置 neverallow、工具链和资源问题已在源码层处理；未复现的问题不继续增加 speculative fix。

Next step: 在重建镜像上只复测有新证据的路径，保留 GPU 统计为内核能力限制。

## 最近一次完整包

Artifact: `/root/android/exthmui12/out/target/product/polaris/exthm-12.0-20260903-UNOFFICIAL-SNAPSHOT-polaris-Vanilla.zip`

Size: `1,104,437,513` bytes

SHA-256: `fa75714cfb8c4b77cec66381e8131950bbeb5f71a14be295cfc94e623b10cd31`
