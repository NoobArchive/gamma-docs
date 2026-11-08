# 清单 / SDK / 权限 / 显示

> 覆盖:安装门槛、targetSdk 选择、权限裁剪、`exported`、主题与导航栏、
> 反向横屏、ABI、存储、明文 HTTP。字体问题单独见 [font-module.md](font-module.md),
> 推送崩溃见 [push-notification.md](push-notification.md)。

## 1. minSdk / targetSdk

现代 Android 的安装门槛:

| 系统 | 拒装条件 |
|---|---|
| Android 14 (34) | `targetSdk < 23` |
| Android 15 (35) | `targetSdk < 24` |

原包:1.1.x `targetSdk=25`,1.3.0+ `targetSdk=22` → 1.3.x 官方包在 14+ 上装不上。

选择建议:

| 方案 | targetSdk | 代价 |
|---|---:|---|
| **保守(推荐)** | **30** | 仍要处理分区存储;但不需要 `exported`、`FLAG_IMMUTABLE`、通知运行时权限、FGS 类型 |
| 折中 | 33 | 追加 `POST_NOTIFICATIONS` 运行时申请、`exported`、PendingIntent flag |
| 激进 | 35 | 再追加 FGS 类型/权限、Android 15 的 BOOT_COMPLETED 限制 |

单机改包参考:`Rhythm Painters.apk` 用 **min 9 / target 30**,已验证可安装。

`minSdk` 抬到 23/24 即可,不必跟着 target 走;原包大量 `Build.VERSION.SDK_INT`
分支在 14 以下才有意义,抬高反而少一堆兼容代码。

## 2. 权限裁剪

1.3.3 官方包 **31 条**,其中真正需要的不到三分之一。参考单机改包只剩 8 条。

| 组 | 权限 | 处理 |
|---|---|---|
| 网络 | `INTERNET` `ACCESS_NETWORK_STATE` `ACCESS_WIFI_STATE` | 保留(私服/统计要用) |
| 存储 | `READ/WRITE_EXTERNAL_STORAGE` | 保留;target 30 仍可申请,target 33+ 基本无效 |
| 设备 | `WAKE_LOCK` `VIBRATE` `MODIFY_AUDIO_SETTINGS` | 保留(挂机/音效) |
| 悬浮窗 | `SYSTEM_ALERT_WINDOW` `GET_TASKS` | 只有渠道 SDK 悬浮图标需要;单机可删 |
| 短信/电话 | `SEND_SMS` `RECEIVE_SMS` `WRITE_SMS` `CALL_PHONE` `READ_PHONE_STATE` | **删**;支付 SDK 的短信代扣,早已失效 |
| 定位 | `ACCESS_COARSE_LOCATION` `ACCESS_FINE_LOCATION` | **删**(除非有 LBS 玩法) |
| 相机 | `CAMERA` `FLASHLIGHT` + `uses-feature camera*` | **删**;扫码支付/分享才用 |
| 系统设置 | `WRITE_SETTINGS` `READ_SETTINGS` `CHANGE_CONFIGURATION` | **删** |
| 调试/越权 | `READ_LOGS` `MOUNT_UNMOUNT_FILESYSTEMS` `INTERACT_ACROSS_USERS_FULL` | **删**(后两条普通应用根本拿不到) |
| 推送 | `RECEIVE_BOOT_COMPLETED` | 看是否保留通知功能 |
| 推送 SDK | `MIPUSH_RECEIVE` `com.meizu.*` `C2D_MESSAGE` | **删**(服务端已死) |

> 注意:`uses-feature android:name="android.hardware.camera"` 不带
> `required="false"` 会被应用商店当成硬性要求,一并删掉。

## 3. `android:exported`(targetSdk ≥ 31)

带 `intent-filter` 的四大组件必须显式声明,否则**安装期**就失败:

```xml
<activity android:name="..." android:exported="true">   <!-- 有 LAUNCHER 的 -->
<receiver android:name="...NotificationBootReceiver" android:exported="false">
<service  android:name="...NotificationService"      android:exported="false">
```

系统广播(`BOOT_COMPLETED` 等)发给 `exported="false"` 的 receiver 仍然能收到,
不需要为了收广播而导出。

## 4. 主题与导航栏

实测 1.3.3:

```
android:theme             = @0x01030007   # Theme.NoTitleBar.Fullscreen
android:screenOrientation = 6             # sensorLandscape
android:configChanges     = 0x4a0          # orientation|keyboardHidden|screenSize
```

`0x01030007` 的继承链是 `Theme.NoTitleBar` + `windowFullscreen=true`。
**`windowFullscreen` 只隐藏状态栏,不隐藏导航栏**;
所以"默认主题不隐藏虚拟导航键"不是选错主题,而是平台主题本来就没有 immersive 语义。

两种改法:

```java
// API < 30
getWindow().getDecorView().setSystemUiVisibility(
    View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY
  | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
  | View.SYSTEM_UI_FLAG_FULLSCREEN
  | View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
```

```java
// API 30+
getWindow().setDecorFitsSystemWindows(false);
getWindow().getInsetsController().hide(WindowInsets.Type.systemBars());
getWindow().getInsetsController().setSystemBarsBehavior(
    WindowInsetsController.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE);
```

> 这是 `NativeActivity`,没有 `setContentView`;在 `onCreate`/`onWindowFocusChanged`
> 里做即可。注意 `configChanges` 已包含 `orientation`,旋转不会重建 Activity,
> 沉浸式标志要在每次焦点变化后重新贴。

## 5. 反向横屏触控偏移

`sensorLandscape` 允许 0°/180° 两个方向。输入由 `NativeActivity` 的 input queue
直接交给 native,Surface 可能带 180° transform;若 native 只用
`ANativeWindow_getWidth/Height`、没对 `ANativeWindow_getTransform` 做反向映射,
就会出现 X/Y 镜像。

验证(MuMu 或真机):

```powershell
adb shell settings put system accelerometer_rotation 0
adb shell settings put system user_rotation 0   # 正向横屏
adb shell settings put system user_rotation 2   # 反向横屏
# 同一个屏幕坐标连点两次,对比 native 收到的 motion event
```

定位顺序:

1. 先看 Java 层 `GammaActivity` 有没有 `onConfigurationChanged` / 手动改坐标;
2. 再看 native 的 `ANativeActivity` 回调里是否读 `AConfiguration_getOrientation`
   或 `ANativeWindow_getTransform`;
3. 若都没有,基本就是没处理 180° transform,需要在把坐标送进 UI 系统前乘一次
   镜像矩阵。

## 6. 权限提示框(NativeActivity 的坑)

1.1.19 / 1.1.20 的 `MPermissionsActivity` 继承链:

```
GameClientActivity → MPermissionsActivity → ShellClient → GammaActivity → NativeActivity
```

拒绝权限时它弹的是 `android.support.v7.app.AlertDialog`,而 Activity 主题是
平台 `Theme.NoTitleBar.Fullscreen` → 必抛
"You need to use a Theme.AppCompat theme (or descendant) with this activity"。

修复:**换平台 `android.app.AlertDialog`**,或自己画一个 View/用 Toast;
不要给 `NativeActivity` 套 AppCompat 控件。

## 7. ABI

1.3.3 的 `lib/`:

```
armeabi       8 个 .so
armeabi-v7a   8 个 .so
x86           5 个 .so   ← 缺 libShellClient.so / libAndroidBoot.so
```

- 没有 **arm64-v8a**,64 位-only 设备装得上也跑不了 native;
- x86 目录不完整,x86 模拟器起不来;
- 本项目用 **MuMu(ARM 转译)**;Android SDK 的 x86_64 AVD 不适用
  (见根目录 `AGENTS.md` 与 `CONTRIBUTING.md`)。

改包时不要把 `lib/x86` 当垃圾删掉——某些渠道包靠它跑模拟器;
但也不要指望它完整。

## 8. 存储与明文 HTTP

- 推送 SDK 会往 `/sdcard/GPushDispatcherInfo.dat` 等路径写日志;
  target 29+ 分区存储后这些写入会失败(不崩,但刷 log)。删除推送 SDK 即解决。
- 若私服/SDK 走 `http://`:target 28+ 默认禁止明文,
  需要 `android:usesCleartextTraffic="true"` 或更细的 `networkSecurityConfig`。
  先抓包确认协议,不要无脑开全局明文。

## 9. 签名

```powershell
zipalign -f -p 4 in.apk aligned.apk
apksigner sign --ks my.jks --v1-signing-enabled true --v2-signing-enabled true `
  --v3-signing-enabled true --out out.apk aligned.apk
apksigner verify --print-certs out.apk
```

target 30+ 的设备只认 v2+,只打 v1 会 `INSTALL_PARSE_FAILED_NO_CERTIFICATES`。
