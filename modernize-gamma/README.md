# Gamma 手游现代化改包指南

> 目标:把 2017–2018 年的 Gamma Android 包体(以《粉刷匠大作战》1.1.x–1.3.3 为代表)
> 改到现代 Android 上**可安装、可启动、字体/触控/通知正常**。
>
> 本文只写"改什么、为什么、怎么验证";格式层面的东西见 `../formats/`。

## 0. 三个前置事实

1. **只有 32 位库**。1.3.3 的 `lib/` 只有 `armeabi`、`armeabi-v7a`、`x86`;
   `x86` 目录里还缺 `libShellClient.so` / `libAndroidBoot.so`,等于跑不起来。
   → 现代 arm64-only 设备与 Android SDK 的 x86_64 AVD 都不适用,
   本项目暂用 MuMu(ARM 转译),见 `../../../V3-1.3.3/Re0_dedicated/Re0_dedicated_server/CONTRIBUTING.md`。
2. **安装门槛**。Android 14 起拒装 `targetSdk < 23`,Android 15 起拒装 `< 24`。
   原包 1.1.x 是 25、1.3.0+ 是 22,后者在 14+ 上直接装不上。
3. **改完必须重签**。`apksigner` 打 v1 + v2 +(v3);只改 zip 不签名会
   `INSTALL_PARSE_FAILED_NO_CERTIFICATES`。

## 1. 改动清单

| 优先级 | 项 | 说明 | 详见 |
|---|---|---|---|
| P0 | targetSdk / minSdk | 22 → 30+(或 33/35);minSdk 抬到 23/24 | [android-manifest.md](android-manifest.md) |
| P0 | 32 位 ABI | 保持 armeabi-v7a;不要盲目删 `lib/x86` | [android-manifest.md](android-manifest.md) |
| P0 | 推送 receiver 崩溃 | 隐式 Service Intent,开机即崩 | [push-notification.md](push-notification.md) |
| P0 | `android:exported` | targetSdk 31+ 的硬性要求 | [android-manifest.md](android-manifest.md) |
| P1 | 权限裁剪 | 31 条 → 9 条级别 | [android-manifest.md](android-manifest.md) |
| P1 | 通知渠道 / 前台服务 | Android 8+ 的后台限制 | [push-notification.md](push-notification.md) |
| P1 | 1.3.3 常驻"正在运行"通知 | 来自 Shanda `GPushService` 的空 Notification,不是提醒服务 | [push-notification.md](push-notification.md) |
| P1 | 字体模块配对 | 旧 APK + 新 native **必然无字** | [font-module.md](font-module.md) |
| P2 | 导航栏 / 刘海适配 | Theme 不隐藏导航栏;刘海侧缺一条渲染 + 触控偏移 | [android-manifest.md](android-manifest.md) |
| P2 | 权限提示框 | NativeActivity 里用了 AppCompat `AlertDialog` | [android-manifest.md](android-manifest.md) |
| P3 | 明文 HTTP | 若私服/SDK 走 `http://`,targetSdk 28+ 默认禁止 | [android-manifest.md](android-manifest.md) |

## 2. 最小工作流

```powershell
. "D:\粉刷匠大作战\2608新的逆向计划！\_agent-sandbox-android261108\toolchain\env.ps1"

apktool d -f -o bop133 app.apk          # 反编译
# ...改 smali / AndroidManifest.xml / res...
apktool b bop133 -o bop133-unsigned.apk
zipalign -f -p 4 bop133-unsigned.apk bop133-aligned.apk
apksigner sign --ks my.jks --v1-signing-enabled true --v2-signing-enabled true `
  --v3-signing-enabled true --out bop133-signed.apk bop133-aligned.apk
adb install -r bop133-signed.apk
```

## 3. 版本基线与实测

| 包体 | package | min | target | 权限数 | 备注 |
|---|---|---:|---:|---:|---|
| 粉刷匠 1.1.6.6–1.1.20 | `com.mqgamer.fenshuajiang.gw9e` | 14 | 25 | 14 | 旧字体模块 |
| 粉刷匠 1.1.24 | 同上 | 15 | 25 | 26 | |
| 粉刷匠 1.2.3 | 同上 | 14 | 25 | 24 | |
| 粉刷匠 1.3.0 / 1.3.3 | `com.shandagames.bop` | 14 | **22** | 26 / **31** | 官方回退 targetSdk |
| 粉刷匠 1.3.2 Lite | `com.mqgamer.fenshuajiang.gw9e` | 14 | 25 | 26 | 1.3.x 仍有旧包名渠道 |
| 单机改包 Rhythm Painters | `moe.zhuwin.bopoffline3` | 9 | 30 | 9 | 已验证的裁剪参考 |

> 结论:不能把"1.3.x = target 22"当通例,先看包名与 `aapt2 dump badging`。

## 4. 验证清单

- [ ] `adb install` 通过(Android 12/14/15 各一台或 MuMu 对应镜像)
- [ ] 冷启动到主界面,字体不是空白/方块/错位
- [ ] 开机不弹 `FATAL EXCEPTION`(推送 receiver)
- [ ] 拒绝权限时提示框不崩
- [ ] 导航栏不遮挡画面;刘海侧画面完整;正/反横屏触控落点一致
- [ ] 后台 5 分钟后回来,游戏没有被系统杀死到白屏
