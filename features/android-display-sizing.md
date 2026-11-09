# Android 平台 - 呈现方式与触屏逻辑

《粉刷匠大作战》默认使用的 Theme 不会隐藏虚拟导航键，这对于游戏而言真的是好主意吗？

此外，粉刷匠在设备处于反向横屏状态时触控指向会与实际点击位置具有偏差，在高版本 Android 下尤甚，需要调查该问题

## 结论(实测)

不是镜像,而是**刘海/挖孔安全区 inset**;是否出现取决于该方向下 inset 落在哪一侧,
与"反向"本身没有必然关系:

- 非全面屏设备基本正常;全面屏设备在摄像头一侧缺一条引擎画面,
  透出 Theme 的 `windowBackground`,同时触控有一个固定量的偏移。
- targetSdk 22 且清单/主题都没有 `layoutInDisplayCutoutMode`,Android 9+ 会按
  "未适配刘海"把 content 区域缩进(刘海安全区 / 系统栏 inset);窗口仍是全屏,
  所以 inset 处露出主题背景。
- `GammaActivity` 用自建 `SurfaceView` 当 content,没有 `LAYOUT_*` /
  `setDecorFitsSystemWindows(false)`;native 的 client rect 直接取
  `ANativeWindow_getWidth/Height`,输入坐标却是窗口坐标,两者差一个 inset。

修复与验证见
[modernize-gamma/android-manifest.md](../modernize-gamma/android-manifest.md) §5。