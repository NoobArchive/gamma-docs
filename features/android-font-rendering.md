# Android 平台 - 字体渲染
Android Java (classes.dex)侧 com.joyegame.gammakernel.GammaFontModule

已知《粉刷匠大作战》一测 1.1.6.x 版本的相应类和后续版本不同，导致使用 1.1.6.x 包体运行 1.3.3 ShellClient 时游戏内字体渲染完全失效

通常而言这样的版本跨度理应配置为更新时要求更新 APK，但粉刷匠正式运营（1.3.0+）后把内测服的版本配置写错了，导致 一测包体未收到 APK 更新直接使用新资源和新Native程序导致游戏无法渲染任何文本

需要调研旧版本粉刷匠和新版本粉刷匠，或许还需要回顾早期 Gamma 手游项目
