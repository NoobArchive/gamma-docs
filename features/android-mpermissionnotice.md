# Android 平台 - 权限申请提示
显然 Android 棉花糖改进了权限系统，需要应用主动申请权限

《粉刷匠大作战》早期使用 targetSdk 版本对应 Android 7.1，当时的最新版本

如果权限申请被拒绝，这个 Activity 就会给出提示。

但似乎这个 Activity 本身就会崩溃掉...

官方的修复方式是...**在准备公测时把 targetSdk 改回 Android 5.1？？？**，好样的，妙趣游戏

很显然我们需要修复这个问题来让现代 Android 开心。（高版本 Android 已拒绝 targetSdk 低于 Android 6.0 的应用安装）

（另外，官方版本粉刷匠包体似乎尝试在 Manifest 里申请大量不必要权限，我们也需要精简，v1.3.3 单机改包 Rhythm Painters.apk 就已经做了精简）