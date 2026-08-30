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

## 基线与设置持久化

- 确认 Settings 首页只有一个“exTHmUI 定制”入口，旧的重复首页入口已移除。
- 进入八个分类，确认页面无崩溃、空白项或无后端的游戏性能档位。
- 未修改任何选项时，确认快速下拉、时钟、电池、亮度、截图、熄屏、Adaptive Playback 和电池灯保持各自默认行为。
- 分别修改至少一个布尔、枚举和范围设置，重启 SystemUI、切换用户并重启设备，确认合法值保持不变。
- 如通过 shell 检查设置，只读取、不批量覆盖：

  ```bash
  adb shell settings list system | grep -E 'qs_|status_bar_|network_traffic|screenshot|screen_off|adaptive_playback|battery_light|lockscreen_battery'
  ```

## 主题与壁纸

- 打开“主题与壁纸”，确认能进入现有 ThemePicker/WallpaperPicker2，而不是缺失的旧主题 APK。
- 更换 Monet 颜色、深浅主题和壁纸，确认 Settings、SystemUI、原生电池图标及圆形电池图标着色同步。
- 确认系统可解析 `org.arrowos.customisation` 主题目标包，无 package/overlay 错误。

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
