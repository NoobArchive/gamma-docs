# Android 平台 - 字体渲染
Android Java (classes.dex)侧 com.joyegame.gammakernel.GammaFontModule

已知《粉刷匠大作战》一测 1.1.6.x 版本的相应类和后续版本不同，导致使用 1.1.6.x 包体运行 1.3.3 ShellClient 时游戏内字体渲染完全失效

通常而言这样的版本跨度理应配置为更新时要求更新 APK，但粉刷匠正式运营（1.3.0+）后把内测服的版本配置写错了，导致 一测包体未收到 APK 更新直接使用新资源和新Native程序导致游戏无法渲染任何文本

需要调研旧版本粉刷匠和新版本粉刷匠，或许还需要回顾早期 Gamma 手游项目

## 结论(实测)

分界不是笼统的 1.1.6.x,而是 **1.1.6.10 → 1.1.7.1**:
1.1.6.6 / 6.8 / 6.10 是旧实现,1.1.7.1 起 Java 与 native 同时换代。

根因是 JNI 方法 `SetTextSize(int)` 的**返回值语义变了**:

- 旧:`return m_yOffset`(纵向偏移,通常 0~4),缓冲区固定 48×48;
- 新:`return m_nRealFontSize`(字体位图边长),缓冲区随字体度量重建。

新 native 把这个返回值当作位图的行步长与行数。旧 APK 返回 0 时,
`FillModule` 一行都不拷贝 → 文字全空白,同时顶部偏移变成约 22px。

反向(新 APK + 旧 native)没事,因为新 `GetBuffer()` 留了
`m_buffer == null → m_bNewVersion = false` 的兼容分支。

细节、反编译地址与兼容矩阵见
[modernize-gamma/font-module.md](../modernize-gamma/font-module.md)。
