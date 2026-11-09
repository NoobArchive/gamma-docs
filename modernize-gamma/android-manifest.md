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

> 这里是 `NativeActivity` + 自建 `SurfaceView`;在 `onCreate`(`setContentView` 之前)
> 或 `onWindowFocusChanged` 里做即可。注意 `configChanges` 已包含 `orientation`,
> 旋转不会重建 Activity,沉浸式标志要在每次焦点变化后重新贴。

## 5. 全面屏 / 刘海:缺一条渲染 + 触控偏移

现象(实测修正):

- **非全面屏设备**:反向横屏触控基本正常;
- **全面屏(刘海/挖孔)设备**:刘海区（摄像头那一侧）**完全没有引擎画面**,
  透出 Theme 的窗口背景(例如 Holo 深色渐变);即使用户在系统里把游戏设成"全屏"仍然如此;
- 触控**不是镜像**,而是整体平移一个 inset 量(肉眼像"一点偏移")。

### 原因

1. 原包 `targetSdk=22`,清单和主题里都**没有 `layoutInDisplayCutoutMode`**。
   Android 9+ 对这类应用按"未适配刘海"处理:横屏时 content 区域被
   `DisplayCutout` 的安全区(以及系统栏)inset 缩进;窗口本身仍是全屏,
   所以 inset 那条露出 Decor 的 `windowBackground`(就是看到的主题背景)。
   系统里的"全屏显示"开关只是**允许**应用使用刘海区,并不会替应用消掉它自己的
   content inset —— 应用侧仍要显式声明。
2. `GammaActivity` 用自建 `SurfaceView` 当 content(`setContentView(m_mainView)`),
   只靠 `windowFullscreen` 隐藏状态栏;没有 `SYSTEM_UI_FLAG_LAYOUT_*`、
   没有 `setDecorFitsSystemWindows(false)`、也没有任何 cutout 处理。
3. Native 侧(`libShellClient.so`):
   - `Gamma::SWindowContext::ProcessInput`(@`0x26c23c`)在窗口创建/尺寸变化时把
     client rect 设成 `(0, 0, ANativeWindow_getWidth(), ANativeWindow_getHeight())`;
   - `glViewport` 用的也是这个尺寸;
   - `CAndroidApp::Run`(@`0x262b1c`)注册的 `ANativeActivityCallbacks` 里
     **没有 `onContentRectChanged`**,也没有用 `ANativeWindow_getTransform`;
   - `AMotionEvent_getX/Y` 拿到的是**窗口坐标**,原样打包 (`x | y<<16`) 后透传。

   于是 Surface 被 inset 后,输入坐标和引擎的 surface 坐标系差了一个 inset 量:
   画面缺一条、触控固定偏移,两个现象同源。

> 这也解释了为什么只有刘海设备中招:普通设备 cutout 为 0,inset 也是 0。

### 修复

清单(API 28+):

```xml
<activity
    android:name="com.mqgamer.fenshuajiang.client.GameClientActivity"
    android:layoutInDisplayCutoutMode="shortEdges"
    ... />
```

`GammaActivity.onCreate`(`setContentView` 之前):

```java
if (Build.VERSION.SDK_INT >= 28) {
    WindowManager.LayoutParams lp = getWindow().getAttributes();
    lp.layoutInDisplayCutoutMode =
        WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES;
    getWindow().setAttributes(lp);
}
if (Build.VERSION.SDK_INT >= 30) {
    getWindow().setDecorFitsSystemWindows(false);
} else {
    getWindow().getDecorView().setSystemUiVisibility(
        View.SYSTEM_UI_FLAG_LAYOUT_STABLE
      | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
      | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
      | View.SYSTEM_UI_FLAG_FULLSCREEN
      | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
      | View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY);
}
```

- `shortEdges` 允许横屏占用左右短边的 cutout;`always`(API 30+)更强,但会连
  竖屏顶部也铺进去,游戏用 `shortEdges` 就够。
- 只加 `shortEdges` 可能画面铺满、触控仍差一点;要同时修触控,必须让 content
  真正铺满(inset 归零),所以 `setDecorFitsSystemWindows(false)` /
  `LAYOUT_*` 不能少。
- 清单里 `configChanges` 已包含 `orientation`,旋转不会重建 Activity;
  沉浸式标志建议在 `onWindowFocusChanged` 里重新贴一次。

### 验证

```powershell
# 看 content frame 与 window frame 是否差一个 cutout 宽度
adb shell dumpsys window windows | Select-String "<package>|frame=|content="
adb shell dumpsys window displays | Select-String "cutout|DisplayFrames"
```

代码级快速判定:在 `GammaActivity.onCreate` 里延迟一帧打印

```java
m_mainView.post(() -> {
    View decor = getWindow().getDecorView();
    Log.i("Gamma", "surface=" + m_mainView.getWidth() + "x" + m_mainView.getHeight()
          + " window=" + decor.getWidth() + "x" + decor.getHeight());
    WindowInsets ins = decor.getRootWindowInsets();
    if (ins != null) {
        if (Build.VERSION.SDK_INT >= 30) {
            Insets i = ins.getInsets(WindowInsets.Type.systemBars()
                                   | WindowInsets.Type.displayCutout());
            Log.i("Gamma", "insets L/T/R/B=" + i.left + "/" + i.top + "/" + i.right + "/" + i.bottom);
        } else {
            Log.i("Gamma", "insets L/T/R/B=" + ins.getSystemWindowInsetLeft() + "/"
                  + ins.getSystemWindowInsetTop() + "/" + ins.getSystemWindowInsetRight()
                  + "/" + ins.getSystemWindowInsetBottom());
        }
    }
});
```

- `surface != window` → content 被 inset(直接证据);
- 哪一侧的 inset 非 0,就应该和"缺一条"的那一侧对得上;
- 如果非 0 的是系统栏宽度而不是刘海宽度,说明 band 来自系统栏 inset,
  修法相同(`setDecorFitsSystemWindows(false)` + 隐藏系统栏);
- 修复后 `surface == window`,四个 inset 全 0。

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
