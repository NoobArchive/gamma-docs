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