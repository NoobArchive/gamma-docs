# 推送/通知服务:崩溃原因与修复方案

> 对象:`com.joyegame.shellclient.NotificationBootReceiver` /
> `NotificationService`(`ShellClient` 体系,1.1.x–1.3.3 通用)。
>
> 结论:**可以修,不必删**。崩溃只是"隐式 Service Intent";但只改这一行会在
> Android 8+ 撞上后台限制,所以要么补前台服务,要么改成 Alarm/Job 定时通知。

## 0. 先分清两套通知系统

1.3.3 里有两套互不相干的东西,常被混为一谈:

| | `com.joyegame.shellclient.NotificationService` | `com.sdg.sdgpushnotificationservice.GPushService` |
|---|---|---|
| 归属 | Gamma ShellClient | 盛大 GamePlus 推送 SDK(2.0.2.0) |
| 用途 | 本地定时/离线提醒,内容来自服务端 cmd 7 | MQTT + 小米/魅族/华为通道 |
| 出现版本 | 1.1.x 起 | **1.3.3**(1.3.0/1.3.2 清单里没有该 service) |
| 是否常驻通知 | 否 | **是**,空 Notification 导致 |
| 谁启动 | 服务端 cmd 7 / 开机广播 | `GameClientActivity.onCreate` → `GHome.initialize` → `GamePlus.my_initGame` → `ThirdPartyApi.init` |

### 1.3.3 的"xxx 正在运行"从哪来

`GPushService.onCreate()`:

```java
this.m_nForegroundID = GPushBaseUtility.generateUniqueNotificationID(this);
startForeground(this.m_nForegroundID, new Notification());   // ← 空通知
startService(new Intent(this, InnerService.class));          // InnerService 也 startForeground(空) 后 stopSelf
```

Android 8+ 会给前台服务额外显示一条**系统级**"正在运行 / 在后台运行"通知
(AOSP: "<app> is running in the background", "Tap to learn more or stop the app");
当应用自己的前台通知是空/无效时,用户就只看到这一条。Android 13+ 还会出现在
"Active apps" 里。

所以:

- 1.1.x / 1.2.x / 1.3.0 / 1.3.2 没有这条,因为它们没有声明/启动 GPushService;
- 它不是 `NotificationService` 发的,清 `NotificationCfg.txt` 不会让它消失。

### GPush 怎么处理

- **默认建议移除**:MQTT/厂商推送服务端早已下线,留着只多一个进程 + 常驻通知。
- 若要保留或自建后端:
  - 给 `startForeground` 一个真实通知(渠道 + 小图标 + 文案),不要 `new Notification()`;
  - 或去掉 `startForeground`,改普通服务/JobScheduler;
  - targetSdk ≥ 26 后空通知会直接 `RemoteServiceException`,必须改;
  - targetSdk ≥ 34 还要 `foregroundServiceType` 与对应权限。

## 1. 现场

开机即崩:

```
FATAL EXCEPTION: main
Process: com.mqgamer.fenshuajiang.gw9e
java.lang.RuntimeException: Unable to start receiver
  com.joyegame.shellclient.NotificationBootReceiver:
  java.lang.IllegalArgumentException: Service Intent must be explicit:
  Intent { act=com.joyegame.shellclient.NotificationService }
```

对应代码(1.3.3,整个类就这一句):

```java
public class NotificationBootReceiver extends BroadcastReceiver {
    public void onReceive(Context context, Intent intent) {
        context.startService(new Intent("com.joyegame.shellclient.NotificationService"));
    }
}
```

而同一个 APK 里,`ShellClient` 其实已经有正确写法:

```java
protected Intent getNotifyServiceIntent() {
    return new Intent(this, NotificationService.class);   // 显式
}
```

也就是说:**官方自己知道要显式,只是开机广播这条路径写漏了**。

## 2. 为什么不能只改一行

| Android | 追加问题 |
|---|---|
| 5.0 (21) | 隐式 Service Intent 直接抛 `IllegalArgumentException` —— 本次崩溃 |
| 8.0 (26) | 后台 `startService()` 抛 `IllegalStateException`;通知必须有渠道,否则 `notify()` 静默失效 |
| 12 (31) | 带 intent-filter 的组件必须写 `android:exported`;`PendingIntent` 必须指定 `IMMUTABLE`/`MUTABLE` |
| 13 (33) | `POST_NOTIFICATIONS` 运行时权限;没授权时通知不显示(不崩) |
| 14 (34) | 前台服务必须声明 `foregroundServiceType` 并申请对应权限 |

原包 `targetSdk = 22` 把这些全绕过了,但 22 在 Android 14+ 已经**装不上**,
所以改包时这些坑会一次性全部暴露。

## 3. 方案 A:保留 Service(最小改动)

### 3.1 清单

```xml
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
<!-- targetSdk >= 34 才需要下面两条 -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_SPECIAL_USE"/>
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>

<receiver
    android:name="com.joyegame.shellclient.NotificationBootReceiver"
    android:exported="false">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED"/>
    </intent-filter>
</receiver>

<service
    android:name="com.joyegame.shellclient.NotificationService"
    android:exported="false"
    android:foregroundServiceType="specialUse">
    <!-- 不需要 intent-filter;显式 Intent 直接按类名启动 -->
    <property
        android:name="android.app.PROPERTY_SPECIAL_USE_FGS_SUBTYPE"
        android:value="game reminder notification"/>
</service>
```

> `BOOT_COMPLETED` 是受保护广播,`exported="false"` 仍能收到系统广播。
> 原来那个 `category.LAUNCHER` 没有意义,可以删。

### 3.2 广播接收器

```java
public void onReceive(Context context, Intent intent) {
    Intent svc = new Intent(context, NotificationService.class);
    if (Build.VERSION.SDK_INT >= 26) {
        context.startForegroundService(svc);
    } else {
        context.startService(svc);
    }
}
```

### 3.3 Service

```java
private static final String CHANNEL_ID = "bop_reminder";
private static final int NOTI_ID = 1;

@Override
public void onCreate() {
    createChannel();
    startForeground(NOTI_ID, buildOngoing());   // 5 秒内必须调用
    // ...原有 NotificationCfg.txt 解析...
}

private void createChannel() {
    if (Build.VERSION.SDK_INT >= 26) {
        NotificationManager nm = getSystemService(NotificationManager.class);
        NotificationChannel ch = new NotificationChannel(
                CHANNEL_ID, "游戏提醒", NotificationManager.IMPORTANCE_LOW);
        nm.createNotificationChannel(ch);
    }
}

private Notification buildOngoing() {
    NotificationCompat.Builder b = (Build.VERSION.SDK_INT >= 26)
            ? new NotificationCompat.Builder(this, CHANNEL_ID)
            : new NotificationCompat.Builder(this);
    return b.setSmallIcon(R.drawable.icon)
            .setContentTitle("")
            .setContentText("")
            .setOngoing(true)
            .setPriority(NotificationCompat.PRIORITY_LOW)
            .build();
}
```

同时把两处 `PendingIntent` 补上 flag(否则 targetSdk 31+ 直接抛异常):

```java
PendingIntent.getActivity(this, 1, new Intent(),
        PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_IMMUTABLE);
```

> 原代码里的 `16`、`134217728` 分别是 `FLAG_ACTIVITY_NO_ANIMATION` 和
> `FLAG_UPDATE_CURRENT`;前者当 PendingIntent flag 用是无效值,顺手改成 0 即可。

### 3.4 `ShellClient.updateNotificationFile`

```java
protected void updateNotificationFile(String fileContext) {
    Intent notify = getNotifyServiceIntent();
    stopService(notify);
    // ...写 NotificationCfg.txt...
    if (Build.VERSION.SDK_INT >= 26) startForegroundService(notify);
    else startService(notify);
}
```

### 3.5 代价

- 通知栏会常驻一条"游戏提醒"前台通知;
- OEM(尤其国产 ROM)仍可能杀后台;Doze 下 `Handler.postDelayed` 会被延迟;
- Android 15 对 `BOOT_COMPLETED` 启动前台服务有限制,但只点名
  `dataSync/camera/mediaPlayback/phoneCall/mediaProjection/microphone`,
  `specialUse` 不在列表内(仍建议实机确认)。

## 4. 方案 B:去掉常驻 Service(推荐)

通知内容本来就来自 `NotificationCfg.txt`(由 native 侧 `sendCommand(7, json)`
经 `updateNotificationFile` 写入),不需要"一直活着的服务"。

做法:

1. 新增 `NotificationAlarmReceiver`,收到 alarm 后 `notify()`,并计算下一次时间;
2. `updateNotificationFile` 解析 JSON 后,用 `AlarmManager` 为每个条目排一个 alarm;
3. `NotificationBootReceiver` 只负责开机后重新读文件、重排 alarm;
4. 不再声明 `NotificationService`,也不需要前台服务权限。

```java
AlarmManager am = getSystemService(AlarmManager.class);
PendingIntent pi = PendingIntent.getBroadcast(this, id,
        new Intent(this, NotificationAlarmReceiver.class).putExtra("id", id),
        PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_IMMUTABLE);
if (Build.VERSION.SDK_INT >= 31 && !am.canScheduleExactAlarms()) {
    am.setAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, triggerAt, pi);       // 非精确
} else {
    am.setExactAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, triggerAt, pi);  // 精确
}
```

优点:没有常驻通知、没有 FGS 类型/权限、进程被杀也能响。
缺点:精确闹钟在 Android 12+ 需要 `SCHEDULE_EXACT_ALARM` 用户授权;
不授权就用非精确,可能晚几分钟 —— 对"回流提醒"完全够用。

> 若以后要联网下发提醒,JobScheduler + `setPersisted(true)` 也可以,
> 但 `JobInfo` 只保证"大致时间",且同样依赖 `RECEIVE_BOOT_COMPLETED`。

## 5. 方案 C:整块删除(仅离线包可考虑)

只有确认"不要任何回流提醒"时才建议删:

- receiver + service 两个类;
- `RECEIVE_BOOT_COMPLETED`、`FOREGROUND_SERVICE*`、`POST_NOTIFICATIONS`;
- 各推送 SDK(MIUI/Meizu/个推)的组件与权限。

删除能减少 ~31 条权限里的一大块,也能消掉开机崩溃;
但这是**功能取舍**,不是唯一修法。按当前结论:能修,不建议默认删。

## 6. 验证

```powershell
# 1) 装包后先看开机广播是否还炸
adb logcat -b crash -d | Select-String "NotificationBootReceiver|FATAL"

# 2) 手动触发 receiver(需要 root / run-as 调试包)
adb shell am broadcast -a android.intent.action.BOOT_COMPLETED -p <package>

# 3) 直接起服务
adb shell am start-foreground-service -n <package>/com.joyegame.shellclient.NotificationService

# 4) 看通知渠道与前台服务
adb shell dumpsys notification --noredact | Select-String "<package>"
adb shell dumpsys activity services <package>
```

验收:Android 8/12/14 各一次,开机不崩、服务能起、通知能显示(授权后)。

## 7. 附:`NotificationCfg.txt` 格式(以后想利用时用)

链路(1.3.3 实测):

```
服务端 SDK 命令 7 (UPDATENOTIFICATIONFILE)
  → native CAndroidSDKInterface::SendLoginAndPay(cmd=7, json)
  → JNI ShellClient.sendCommand(7, json)
  → Handler → updateNotificationFile(json)
  → files/NotificationCfg.txt + 重启 NotificationService
```

JSON 结构(`NotificationService.onCreate` 解析):

```json
{
  "notification": [
    { "id": 1, "title": "该回来看看了", "context": "体力已恢复",
      "iconName": "icon", "type": 1, "time": "09-02-01-20-30" },
    { "id": 2, "title": "离线奖励", "context": "离线超过 30 分钟",
      "iconName": "icon", "type": 2, "time": 30 }
  ]
}
```

- `type=1` 定时:`time` 是 `MM-FF-WW-HH-mm`(年由代码补),对应
  `SimpleDateFormat("yyyy-MM-FF-WW-HH-mm")`;
  - `time.charAt(0) == '9'` → 按月重复;
  - `time.charAt(6) == '9'` → 按周重复;
  - 两者都是 `9` → 每月固定;
  - 0:00–8:00 会被顺延到 8:00。
  - `FF` 最终用于 `Calendar.set(DAY_OF_WEEK, ...)`,语义以原解析代码为准。
- `type=2` 离线:`time` 是分钟数;应用退到后台 N 分钟后提醒。
- `iconName` 会解析但**从未使用**,通知图标固定 `R.drawable.icon`。
- 服务每 60 秒检查一次;`isBackground()` 用
  `ActivityManager.getRunningAppProcesses()` 判自身 importance == 400。

### 为什么"从来没见它工作过"

- Lua(`misc/ShellClientLua` 全量)和 native 里都没有 `notification` 字样;
- 唯一写入点是服务端命令 7,而现有服务端实现/抓包都没有下发过它;
- 没有命令 7 时,`NotificationCfg.txt` 不存在,服务 `FileNotFoundException`
  后静默返回,什么也不做。

所以它是"预留给运营后台的本地提醒通道",客户端本身不会产生内容。

