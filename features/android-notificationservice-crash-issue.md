# Android 平台 - 通知推送服务崩溃

在手机启动后立刻带着整个应用崩溃，在一些不隐藏崩溃提示的 Android 系统中尤为烦人。

自《爆彩童话》到《粉刷匠大作战》的整个生命周期都没有修掉

```
FATAL EXCEPTION: main
Process: com.mqgamer.fenshuajiang.gw9e, PID: 13592
java.lang.RuntimeException: Unable to start receiver com.joyegame.shellclient.NotificationBootReceiver: java.lang.IllegalArgumentException: Service Intent must be explicit: Intent { act=com.joyegame.shellclient.NotificationService }
	at android.app.ActivityThread.handleReceiver(ActivityThread.java:4939)
	at android.app.ActivityThread.-$$Nest$mhandleReceiver(Unknown Source:0)
	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2527)
	at android.os.Handler.dispatchMessage(Handler.java:109)
	at android.os.Looper.loopOnce(Looper.java:232)
	at android.os.Looper.loop(Looper.java:317)
	at android.app.ActivityThread.main(ActivityThread.java:8934)
	at java.lang.reflect.Method.invoke(Native Method)
	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:591)
	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:911)
Caused by: java.lang.IllegalArgumentException: Service Intent must be explicit: Intent { act=com.joyegame.shellclient.NotificationService }
	at android.app.ContextImpl.validateServiceIntent(ContextImpl.java:1991)
	at android.app.ContextImpl.startServiceCommon(ContextImpl.java:2033)
	at android.app.ContextImpl.startService(ContextImpl.java:2004)
	at android.content.ContextWrapper.startService(ContextWrapper.java:855)
	at android.content.ContextWrapper.startService(ContextWrapper.java:855)
	at com.joyegame.shellclient.NotificationBootReceiver.onReceive(NotificationBootReceiver.java:12)
	at android.app.ActivityThread.handleReceiver(ActivityThread.java:4930)
	... 9 more

```

## 问题原因及修复方式

`NotificationBootReceiver` 只有一句:

```java
context.startService(new Intent("com.joyegame.shellclient.NotificationService"));
```

Android 5.0 起 Service Intent 必须是显式的,这里只给了 action,所以开机广播一进来就抛
`IllegalArgumentException`。讽刺的是,同一个 APK 的 `ShellClient.getNotifyServiceIntent()`
本来就是显式写法,只有开机路径漏了。

但只改成显式 Intent 还不够:Android 8+ 会撞后台服务限制,12+ 还要 `exported` /
`PendingIntent` flag,13+ 要通知权限,14+ 要前台服务类型。完整分析与两种修法见
[modernize-gamma/push-notification.md](../modernize-gamma/push-notification.md)。

结论:可以修,不必删。

## 附:1.3.3 的"正在运行"常驻通知不是它

那条通知来自盛大的 `com.sdg.sdgpushnotificationservice.GPushService`:
它在 `onCreate` 里 `startForeground(id, new Notification())`(空通知),
Android 8+ 便显示系统级的"正在运行 / 点按即可了解详情或停止应用"。
1.3.0 / 1.3.2 没有声明这个 service,所以没有;与本篇的提醒服务无关。
详见 [modernize-gamma/push-notification.md](../modernize-gamma/push-notification.md) §0。