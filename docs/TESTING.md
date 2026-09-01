# exTHmUI 12 Polaris 功能验证

适用设备：Xiaomi Mi MIX 2S（polaris）
适用版本：本项目生成的 exTHmUI 12 userdebug 包

本文只描述刷入成功后的验证步骤。不要在未核对设备、分区和产物时执行刷机命令；不要擦除 persist、modem、校准数据或引导加载器关键分区。

## 测试前准备

1. 记录所刷 ZIP 的文件名和 SHA-256。
2. 首次开机后确认 SELinux 仍为 Enforcing、加密和主要硬件功能未被关闭。
3. 在开始测试前保存一份完整日志：

   ```bash
   adb wait-for-device
   adb shell getprop ro.build.fingerprint
   adb shell getenforce
   adb logcat -b all -d > logcat-initial.txt
   adb shell dumpsys > dumpsys-initial.txt
   ```

4. 除明确要求外，优先通过“设置 > exTHmUI 定制”修改选项，不直接写数据库。

## 已完成且本轮不重复的测试

### 热点客户端管理与网络转发（阶段 2）

2026-08-31 已使用已 root 的 Android 14 设备 `chopin` 完成真机复测。WPA2 关联、DHCP、Android `VALIDATED`、私有 DNS、Polaris 的 `wlan0` 上游与 `wlan1` 下游、转发/NAT、硬件 offload，以及持续 HTTP/ICMP 流量均通过；没有复现热点断网。测试后热点、临时网络和诊断覆盖均已清理并恢复默认状态。

除非后续改动涉及 SoftAP、Tethering、Wi-Fi、网络栈或出现新的可复现回归，否则不要重复该测试，也不要使用 Windows 无线网卡作为健康判据。完整记录见 `artifacts/stage2/hotspot-recheck-android-client-20260831.md`。

## Updater 与非 A/B Recovery OTA

### 默认未配置状态

- 首次启动或清除 Updater 数据后，打开“设置 > 系统 > 系统更新”，应显示“OTA 更新源未配置”，列表为空且刷新不可用。
- `adb shell getprop ro.exthm.updater.uri` 默认应为空；APK 内 `updater_server_url` 也应为空。默认状态不得发起 HTTP(S) 请求、读取旧 feed cache 或保留周期检查/重试 alarm。
- `android.settings.SYSTEM_UPDATE_SETTINGS` 必须只解析到 `org.lineageos.updater/.UpdatesActivity`，不得出现第二个系统更新处理器。
- 检查 `adb shell ls -ldZ /data/exthm_updates`：目录应由 init 创建为 `system:cache`、模式 `0770`，SELinux 类型为 `ota_package_file`。SELinux 必须保持 Enforcing。

### 更新源输入与切换

- 通过 Updater Preferences 的 Save/Clear 操作配置或清除 override，不直接改 SharedPreferences 或数据库。
- 逐项拒绝 `http://`、带用户名/密码、fragment、空白/控制字符、反斜杠、未知模板、非 HTTPS 重定向和畸形 URL；原始非空无效 override 必须保持 INVALID，不得静默回退到产品属性或资源。
- 只允许 `{device}`、`{type}`、`{version}`、`{ziptype}`、`{incr}`。使用受控 HTTPS 服务器确认展开值与当前 Polaris Vanilla 构建一致。
- 保存相同源不得无条件清 cache；更改源时应立即取消旧请求、旧重试和旧 alarm。旧源较晚返回的响应不得覆盖新源 cache 或列表；Clear 后应回到未配置状态且不联网。

### Feed 与列表状态

- 使用受控 feed 覆盖空/超大响应、非 JSON、缺字段、重复 ID、非法 filename/URL、错误 device、非 Android `12.0`、错误发布类型、未来/回退 timestamp、非法 size，以及 HTTP 错误和重定向链。
- 同 ID 更新必须比较 filename、timestamp、romtype、version 和 size。跨源切换时清理所有非 VERIFIED 项；已验证包可以保留为 offline，但与新 feed 冲突时不得被静默覆盖。
- 切换源、重启应用、重启设备和清除 source 后检查列表、通知与数据库一致，不出现旧源更新、空指针崩溃或 RecyclerView 越界。

### 下载、验证与文件生命周期

- 覆盖开始、暂停、恢复、取消、删除、网络断开/恢复、服务重启和同一更新重新下载。每次新下载必须使用新的 operation 文件，旧 callback 不得删除或覆盖 replacement。
- 验证实际下载字节数、feed size、OTA whole-file 签名、`pre-device=polaris`、非 A/B block OTA 形态和最终安装前复验。包含 `payload.bin` 的 A/B 包必须拒绝。
- `compatibility.zip` 数量为 0 或 1 均可接受：0 依赖制包阶段 target-files VINTF 检查，1 由 `RecoverySystem.verifyPackage()` 追加验证；数量大于 1 或目录形式必须拒绝。
- 验证失败、取消和删除后不得留下无数据库归属的文件；应用重启应清理 orphan。VERIFIED 文件必须保持可读，pending-install 偏好写入失败时必须禁止安装。

### Recovery 安装安全边界

- 未核对 ZIP 文件名、SHA-256、OTA 签名、metadata、目标设备和明确刷机授权前，不要点击安装/重启，也不要调用任何 Recovery 或分区写入命令。
- Polaris 只能走 `RecoverySystem.installPackage()` 的非 A/B 路径；不得出现 UpdateEngine、payload apply、直接 `PowerManager.reboot()` 或 A/B UI fallback。
- 获得明确授权后，先用可恢复测试环境验证确认页、pending-install 持久化、Recovery 命令生成、失败回退和开机后的状态清理。不得擦除 persist、modem、校准数据或重锁 bootloader。

## 基线与设置持久化

- 确认 Settings 首页只有一个“exTHmUI 定制”入口，旧的重复首页入口已移除。
- 进入八个分类，确认页面无崩溃、空白项或无后端的游戏性能档位。
- 未修改任何选项时，确认快速下拉、时钟、电池、亮度、截图、熄屏、Adaptive Playback 和电池灯保持各自默认行为。
- 分别修改至少一个布尔、枚举和范围设置，重启 SystemUI、切换用户并重启设备，确认合法值保持不变。
- 如通过 shell 检查设置，只读取、不批量覆盖：

  ```bash
  adb shell settings list system | grep -E 'qs_|status_bar_|network_traffic|screenshot|screen_off|adaptive_playback|battery_light|lockscreen_battery'
  ```

## WFD / Miracast

### 包加载与基础服务

- 确认当前生效的 `com.qualcomm.wfd.service` 来自本次待验证产物，并记录其 APK SHA-1。
- 确认 PackageManager 不再报告 `requires unavailable shared library .WfdService`，包不得因缺失必需 native shared library 而被忽略。
- 确认实际 WFD JNI 库 `libwfdnative.so` 和 Qualcomm WFD HAL/守护进程仍存在；不要通过伪造 public native library 来使包扫描通过。
- 重启后再次检查 PackageManager、`wfdhdcphalservice` 和 `wfdvndservice`，确认修复不依赖一次性手工启动。

### 投屏行为

- 在“设置 > 已连接的设备 > 连接偏好设置 > 投屏”中开启无线显示，确认能持续扫描并发现已知可用的 Miracast Sink。
- 完成首次连接，验证画面、音频、触摸/旋转后的显示方向，以及普通和受保护内容的 HDCP 行为。
- 主动断开后立即重连；随后分别重启手机和 Sink 后重连，确认无卡死、无残留会话，不需清除应用数据。
- 每次测试同时收集 PackageManager、ActivityManager、Wi-Fi Display、SurfaceFlinger、AudioFlinger 和 WFD vendor 日志，关联首个有意义的错误，不只记录最终断连现象。

## 软件 Face Unlock

### 组件与注册

- 确认 `adb shell pm list features` 同时包含 `android.hardware.biometrics.face` 和 `android.hardware.fingerprint`。
- 确认 `ro.face_unlock_service.enabled=true`，并确认 platform 签名的 `org.pixelexperience.faceunlock` 作为 system priv-app 安装。
- `dumpsys biometric` 应列出 sensor ID `1008` 的 Face provider，强度必须为 `STRENGTH_WEAK`；不得添加伪造的 Face HAL 或 VINTF 声明。
- 确认 privapp 权限、hidden-api allowlist、ArcSoft/Megvii arm64 库和模型随同一系统镜像安装，无缺库、类加载或 PackageManager 拒绝信息。

### 录入、删除与重录

- 从“设置 > 安全 > 人脸与指纹”进入人脸录入；验证凭据确认、challenge/HAT 获取、前摄预览和完成返回均正常。
- 最多只允许一个模板。取消录入、锁屏凭据错误、相机权限/隐私开关禁用和中途切后台时，应安全退出并允许再次进入。
- 完成录入后验证普通删除；删除后入口应回到“未录入”，锁屏不得继续认证旧模板。
- 验证“重新录入”：确认提示后先删除旧模板，再携有效 HAT 启动新录入；失败时不得留下无法删除或无法重录的半状态。

### 解锁与应用认证

- 分别在亮屏、抬手/按键唤醒、通知到达、失败后重试和临时锁定恢复后验证锁屏人脸；PIN/图案/密码和指纹必须始终可用。
- 验证“直接进入桌面/停留锁屏”行为、用户关闭“使用人脸解锁设备”后的状态，以及重启后的持久化。
- 使用请求 `BIOMETRIC_WEAK` 的测试应用验证 `BiometricPrompt`，软件人脸必须要求用户确认；请求 `BIOMETRIC_STRONG` 的应用不得把该 2D 人脸当作强生物识别或支付级凭据。
- 前摄被相机应用占用、相机隐私开关打开、横竖屏切换、暗光、连续失败和服务进程重启时，系统不得卡死或阻断指纹/凭据回退。
- 切换用户并重启设备后确认模板、开关和锁定状态按用户隔离；不得跨用户复用模板。

该实现只使用 RGB 前摄，属于便利性的弱 2D 生物识别，不应宣称具备活体、TEE 安全相机、抗照片/视频攻击或支付级安全性。

## 主题与壁纸

- 打开“主题与壁纸”，确认能进入现有 ThemePicker/WallpaperPicker2，而不是缺失的旧主题 APK。
- 更换 Monet 颜色、深浅主题和壁纸，确认 Settings、SystemUI、原生电池图标及圆形电池图标着色同步。
- 确认系统可解析 `org.arrowos.customisation` 主题目标包，无 package/overlay 错误。

## 天气服务、锁屏天气与桌面小组件

### 默认关闭与权限

- 新装或清除天气服务数据后，确认天气服务和锁屏天气均默认关闭；未启用时不得请求定位、启动周期更新或发起网络请求。
- 从“设置 > exTHmUI 定制 > 便利功能 > 天气服务”进入 OmniJaws 设置；从“锁屏 > 锁屏天气”进入锁屏天气页面，两个入口均不得依赖隐式 Intent 解析。
- 开启自动定位时验证 Android 12 前台定位权限流程；拒绝、仅本次允许、使用期间允许和永久拒绝后重新进入设置都应有明确状态，不能循环弹窗或静默联网。
- SystemUI 只能通过签名权限 `org.omnirom.omnijaws.READ_WEATHER` 读取 Provider；普通第三方应用不得读取或写入天气、设置 URI，也不得调用受保护的控制服务或伪造更新广播。

### 服务、城市与网络

- 默认提供商应为 MET Norway、单位为公制、更新间隔为两小时、图标包为 outline。
- 自动定位和手动城市分别完成一次有效更新；切换模式、修改城市、关闭服务和重新开启后，旧城市或旧天气不得错误残留。
- OpenWeatherMap 未填写用户 API key 时不得发送请求，并应显示可理解的配置提示；填写有效 key 后再验证更新。不得在源码、日志或备份中保存项目公共 key。
- 抓取一次网络日志，确认所有提供商请求均为 HTTPS；重定向到 HTTP、证书错误、超时、空响应、畸形 JSON 和限流响应应安全失败并保留上次有效数据或显示无数据状态。
- 两小时周期更新、手动刷新、重启、网络断开/恢复、飞行模式和用户切换后检查调度，避免重复 Job/Alarm、后台定位或无界重试。

### 锁屏紧凑行与 KeyguardSlice

- 默认不显示天气。分别选择紧凑行和 Slice，两种样式必须严格互斥，切换后即时生效且重启、SystemUI 重启和用户切换后保持。
- Smartspace 开启时，Slice 只显示 OmniJaws 天气行；日期、闹钟、媒体、DND 和 action row 不得与 Smartspace 重复。Smartspace 关闭时，原有 KeyguardSlice 内容应恢复。
- 覆盖无天气数据、Provider 暂时不可用、服务关闭、锁屏天气关闭和权限被撤销；锁屏不能崩溃、卡住或持续显示陈旧天气。
- 在小/大时钟、AOD、充电、通知图标出现/消失、横竖屏、RTL、字体缩放和解锁共享元素动画下检查位置。重点确认紧凑行不会被通知区域挤压，Smartspace 与 weather-only Slice 的 Y 动画一致且无跳动、重叠或空隙。
- 次屏/锁屏演示布局必须保持 only-clock 行为，不能因为 Dagger 绑定所需的隐藏天气 View 而额外显示天气。

### 桌面小组件

- 从小组件选择器手动添加天气小组件；确认 ROM 没有修改 Launcher3 默认工作区，也没有自动迁移或重写桌面数据库。
- 验证首次添加、缩放到最小/最大尺寸、横竖屏、字体缩放、主题/Monet 切换、手动城市和自动定位更新。
- 服务关闭、无数据、定位拒绝、网络断开、重启、Launcher 重启和删除/重新添加小组件后不得崩溃或留下失效更新任务。

## Seedvault 备份与手动恢复

### 包、权限与默认 transport

- Vanilla 镜像应包含 system_ext privileged app `com.stevesoltys.seedvault`、LocalContactsBackup、privapp 权限 XML 和 transport sysconfig whitelist；GApps 镜像不得打包 Seedvault。
- Vanilla 新用户的默认 transport 必须是 `com.stevesoltys.seedvault.transport.ConfigurableBackupTransport`；GApps 版本必须继续使用 GMS transport。
- 只读检查 BackupManager、Secure Setting 和已注册 transport 是否一致：

  ```bash
  adb shell bmgr list transports
  adb shell dumpsys backup
  adb shell settings get secure backup_transport
  ```

- Settings 中的接收器必须保持 `exported=false`。日志中不得出现权限拒绝、transport whitelist 拒绝或重复迁移循环。

### 升级迁移与幂等性

- 从仍保存旧 GMS 默认值、但未安装 GMS 的 Vanilla 版本升级；开机后确认系统选择 Seedvault，且 BackupManager 与 Secure Setting 状态一致。
- 分别以 LocalTransport、Seedvault、可用的 GMS transport 和另一个有效 transport 作为当前用户选择完成升级，确认均不会被覆盖。
- 重启两次并切换用户，确认迁移按用户工作且不会在已正确选择后重复改写。
- 若 Seedvault transport 绑定失败，确认当前选择不被伪造为成功；下一次启动仍可安全重试。

### 备份与恢复

- 完成 AOSP Provision 后，从“设置 > 系统 > 备份 > Seedvault”配置存储位置并完成一次可验证的应用/数据备份。
- 在 Seedvault 设置页确认手动恢复入口可见；本项目不宣称支持首次开机向导恢复，也不得引入 Lineage SetupWizard、Profiles、Lineage SDK 或完整 LineageParts。
- 只使用可丢弃测试数据验证恢复。确认恢复前提示清晰、失败可重试、重启后状态一致，并检查应用数据、联系人和所选存储后端。
- post-setup 恢复可能覆盖正在使用的应用数据；不要在唯一数据副本或生产用户上首次验证。

## 状态栏与快捷设置

### 快速下拉

- 关闭时，单指从任意边缘下拉仍进入普通通知栏。
- 右侧模式仅从屏幕右侧四分之一区域触发直接展开 QS；左侧模式同理。
- 在锁屏、Doze/AOD、验证界面、通知栏已展开或面板被禁用时，不应触发快速下拉。

### 时钟、电池和移动网络图标

- 逐项验证时钟跟随系统、右、中、左位置；旋转、切换用户和主题后位置正确且只有一个时钟实例。
- 秒钟开关即时生效；24 小时制下不显示 AM/PM。
- 验证电池跟随系统、竖向、圆形、纯文字，以及百分比跟随、隐藏、图标内、图标旁。
- 纯文字模式始终只显示一份百分比；充电、省电和深色主题状态表达正确。
- 开启旧移动网络图标后，只改变 data type 的视觉布局；通话、数据注册、信号强度和网络切换行为不得改变。

### 网速与 QS 网格

- 分别验证关闭、状态栏、快速设置标题位置；同一时刻只出现一处网速。
- 验证上下行、仅上行、仅下行，bits/bytes、箭头、刷新周期和自动隐藏阈值。
- 断开所有有效网络后网速应隐藏，恢复网络后自动恢复；禁用后不应继续周期刷新或耗电。
- 竖屏验证行 `1–5`、列 `2–5`；横屏验证行 `1–3`、列 `3–6`。
- 值 `0`、非法值、旋转和用户切换都应即时回到资源默认或对应用户布局，QQS tile 布局保持正常。

### 亮度

- 验证跟随系统、隐藏、仅展开 QS、始终显示四种滑块模式。
- “始终显示”时 QQS 与展开 QS 的滑块过渡平滑，数值同步，不出现两套互相覆盖的镜像。
- 自动亮度按钮默认隐藏；开启后仅在设备支持自动亮度时显示，并能切换当前用户的自动/手动模式。
- 状态栏滑动亮度默认关闭。开启后仅从收起状态栏开始、长按达到阈值后横向滑动生效。
- 锁屏、Doze、验证界面、通知栏展开、多指、纵向下拉或窗口隐藏时必须取消手势，不得误调亮度。

## 锁屏与截图

### 充电信息

- 默认关闭时锁屏充电文案保持原样。
- 开启后连接充电器，按广播实际可用字段显示 mA、W、V、°C；无效字段应单独省略。
- 更换充电器、温度变化、切换用户后即时更新；断开充电器后不残留信息。

### 三指与截图格式

- 三指截图只验证现有手势后端，不应出现第二套手势冲突。
- 质量 `100`：文件扩展名、MIME、MediaStore 和分享/编辑均为 PNG。
- 质量 `10–99`：最终文件为对应质量 JPEG，扩展名和 MIME 均为 JPEG。
- 普通截图、长截图裁剪、分享、编辑、快速分享和删除流程均需验证。
- 长截图中间拼接不应出现重复 JPEG 压缩伪影；只在最终导出应用格式和质量。

## 显示、电源与媒体

### 熄屏动画

- 分别验证 Fade、CRT、Scale，每种至少连续熄屏/唤醒 30 次。
- 在竖屏、横屏、主题切换、用户切换和充电状态下重复测试。
- 验证 AOD/Doze 和亮屏路径保持原行为，无黑色覆盖层、冻结、Surface 泄漏或 system_server 崩溃。
- 若截图、EGL 或显示初始化失败，应自动回退 Fade；记录完整 system_server 和 SurfaceFlinger 日志。

### Adaptive Playback

- 默认关闭时，将媒体音量降到 0 不应由系统自动暂停/恢复。
- 开启后，在音乐活跃时把媒体音量从非零降到 0，应只暂停当前本地媒体会话一次。
- 在所选超时内重新调高音量，应只恢复同一用户、同一会话和同一路由。
- 超时、关闭功能、更改超时、用户手动媒体控制、切换会话、切换扬声器/耳机/蓝牙或切换用户后，不得自动恢复。
- 分别验证 30 秒、1/2/5/10 分钟选项，并检查快速静音后立即恢复音量的顺序。

## 通知与白色电池灯

- 三个开关默认均开启，确认行为与原 AOSP 基线一致。
- 关闭总开关后，低电量、充电和满电均不亮。
- 禁止 DND 时点亮后，开启勿扰应立即熄灭；允许时保持原充电指示。
- 关闭低电量闪烁后，低电量放电状态改为白色常亮；不要寻找无效的颜色或亮度选项。
- 验证充电、接近满电、满电、低电量和用户切换，确认没有修改灯光 HAL 的副作用。

## 日志收集

出现问题后不要只截取最终异常。先复现一次，再立即收集：

```bash
adb logcat -b all -v threadtime -d > logcat-full.txt
adb shell dumpsys activity service SystemUI > dumpsys-systemui.txt
adb shell dumpsys display > dumpsys-display.txt
adb shell dumpsys media_session > dumpsys-media-session.txt
adb shell dumpsys audio > dumpsys-audio.txt
adb shell dumpsys battery > dumpsys-battery.txt
adb shell dumpsys notification > dumpsys-notification.txt
```

报告时注明：ZIP 与 SHA-256、是否干净刷入、测试用户、设置值、最短复现步骤、期望行为、实际行为、首次有意义的错误时间点，以及对应完整日志。
