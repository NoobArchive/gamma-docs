# `GammaFontModule`:Java ↔ Native 字体契约

> 结论:字体不渲染不是"资源/字体文件"问题,而是 **`SetTextSize(int)` 的返回值语义
> 在两代之间变了**。旧 APK + 新 native 会把返回值当位图边长,而旧实现返回的是
> 纵向偏移(通常 0),于是 native 侧取到 0×0 的图 → 整屏文字空白。
>
> 实测版本:1.1.6.6(旧)、1.1.7.1(新)、1.3.3(新)。
> 工具:`_agent-sandbox-android261108\toolchain`,IDA 库:
> 1.1.6.6 `libShellClient.so` / 1.1.7.1 / 1.3.3 `native_Gamma\libShellClient.so.i64`。

## 1. 两代分界

| 版本 | Java 实现 | Native 实现 |
|---|---|---|
| 粉刷匠 ≤ 1.1.6.10,爆彩童话 1.1.0.2 | 旧(静态 48×48) | 旧(硬编码 48/45) |
| 粉刷匠 ≥ 1.1.7.1 | 新(实例位图) | 新(用返回值做边长) |
| 贪吃蛇乐园、全民大冒险、雷霆 | 无 `gammakernel` | — |

判定方法(裸 dex 字符串,`scripts/font_generation.py`):

```
m_yOffset=1 m_nRealFontSize=0  → 旧
m_yOffset=0 m_nRealFontSize=1  → 新
```

## 2. Java 侧

`com.joyegame.gammakernel.GammaFontModule` 只暴露 4 个方法:

```
static GammaFontModule CreateFont()
int   SetTextSize(int size)
int[] GetBuffer()
int   LoadChar(int c)
```

### 旧(1.1.6.6)

```java
public static int MAX_FONT_SIZE = 48;
public static Bitmap g_bitmap = Bitmap.createBitmap(48, 48, ARGB_8888);
public static Canvas g_canvas = new Canvas(g_bitmap);
private int m_yOffset;
private int[] m_buffer = new int[48 * 48];

public int SetTextSize(int size) {
    m_paint.setTextSize(size);
    Paint.FontMetrics fm = m_paint.getFontMetrics();
    m_yOffset = (int) Math.max(((fm.descent / 2.0f) - fm.ascent) - size, 0.0d);
    m_yDrawPos = -fm.ascent;
    return m_yOffset;                  // ← 纵向偏移
}

public int LoadChar(int c) {
    g_canvas.drawColor(0xFF000000);
    g_canvas.drawText(..., 0.0f, m_yDrawPos, m_paint);
    g_bitmap.getPixels(m_buffer, 0, 48, 0, 0, 48, 48);
    return (int) m_paint.measureText(...);
}
```

### 新(1.1.7.1 / 1.3.3)

```java
private int m_nRealFontSize;
private boolean m_bNewVersion = true;

public int[] GetBuffer() {
    if (m_buffer == null) {            // ← 老 native 的兼容开关
        m_bNewVersion = false;
        m_nRealFontSize = MAX_FONT_SIZE;   // 48
        m_buffer = new int[48 * 48];
    }
    return m_buffer;
}

public int SetTextSize(int nFontSize) {
    m_paint.setTextSize(nFontSize);
    Paint.FontMetrics fm = m_paint.getFontMetrics();
    float fAscent  = Math.abs(fm.ascent);
    float fDescent = Math.abs(fm.descent);
    float fLeading = Math.abs(fm.leading);
    m_yDrawPos = fAscent;
    if (m_bNewVersion) {
        m_nRealFontSize = (int) (fLeading + fAscent + fDescent + 0.5f);
        m_buffer = new int[m_nRealFontSize * m_nRealFontSize];
    }
    m_bitmap = Bitmap.createBitmap(m_nRealFontSize, m_nRealFontSize, ARGB_8888);
    m_canvas = new Canvas(m_bitmap);
    return m_bNewVersion ? m_nRealFontSize : 0;   // ← 位图边长
}
```

注意 `GetBuffer()` 里那段 `m_buffer == null`:**只有老 native 会先调 `GetBuffer` 再调
`SetTextSize`**,于是它自动退回 48×48 老模式。这是官方给自己留的版本兼容开关,
也说明"新 Java 配老 native"是被支持的方向。

## 3. Native 侧

`Gamma::CFontModule` 在 `OnLoadedEnd` 里用 classloader 找类、缓存 methodID,
并把状态放在 `this+4` 指向的结构体里。

### 旧 native(1.1.6.6,`OnLoadedEnd` @ `0x23f438`)

调用顺序:

```
CreateFont()  →  GetBuffer()  →  NewGlobalRef  →  SetTextSize(45)
                                                  └─ 返回值存 struct+8,之后没人用
```

`GetCharModule` @ `0x23f6e4` 的 Java 分支:

```c
width = LoadChar(ch);            // measureText,最大 255
buf   = GetIntArrayElements(...);
for (row = 0; row != 45; ++row) {        // 固定 45 行
    for (i = 0; i < width; ++i)
        dst[3 + row*48 + i] = BYTE1(buf[row*48 + i]);   // 固定 48 步长,取 G 通道
}
dst[0] = 0;                      // 顶部偏移永远是 0
dst[2] = width;
```

→ **`SetTextSize` 的返回值完全不参与几何计算**,常量 48/45 写死。

### 新 native(1.1.7.1 @ `0x29550c`,1.3.3 @ `0x2cb9d4`)

调用顺序:

```
CreateFont()  →  SetTextSize(45)  →  GetBuffer()  →  NewGlobalRef
                  └─ 返回值存 struct+12
struct+8 = (45 - struct+12) * 0.5f      // 基线补偿
```

`GetCharModule`(1.3.3 @ `0x2cbe24`)的 Java 分支:

```c
w = min(LoadChar(ch), 48);
dim = struct[+12];                     // ← SetTextSize(45) 的返回值
params = { stride: 4, row_bytes: dim*4, width: w, height: dim };
buf = GetIntArrayElements(struct[+20]);
FillModule(buf + 1, params, dst);      // 取每个 int 的 G 通道
dst[0] = firstNonEmptyRow + struct[+8];   // 顶部偏移
dst[2] = w;
```

`FillModule` @ `0x2cbd7c` 的目标格仍是 48×48(`dst[48*row+3+i]`,`row > 47` 截止),
但**源图的行步长与行数都由 `SetTextSize` 的返回值决定**。

## 4. 兼容矩阵(实测)

| APK 的 Java \ native | 旧 native(≤1.1.6.10) | 新 native(≥1.1.7.1) |
|---|---|---|
| 旧 Java | ✅ 正常 | ❌ **文字空白** |
| 新 Java | ✅ 正常(`GetBuffer` 回退 48×48) | ✅ 正常 |

### 为什么旧 Java + 新 native = 空白

`SetTextSize(45)` 在旧实现里返回
`max((descent/2 - ascent) - 45, 0)`,典型值是 **0~4**。新 native 拿到 0 后:

- `dim = 0` → `FillModule` 的行循环 `while (row < 0)` 一次都不执行 → 一个像素都不拷;
- `dst[0] = round(-1 + (45 - 0)/2 + 0.5) ≈ 22` → 即使有像素也会整体下移 ~22px。

结果就是"布局还在(advance 由 `measureText` 提供)、但一个字都画不出来",
和"一测 1.1.6.x 包体 + 1.3.3 ShellClient 字体完全失效"的现象一致。

### 反过来为什么没事

新 Java 的 `GetBuffer()` 在 `m_buffer == null` 时把 `m_bNewVersion` 置 false,
分配 48×48、返回 0。旧 native 本来就不用返回值,继续按 48/45 取图 —— 正常。

## 5. 处理建议

1. **首选:配对规则**。native 与 APK 必须同代;服务器版本配置应强制
   `1.1.6.x` 包体更新 APK,而不是只换资源/`libShellClient.so`。
2. **要让旧包体吃新 native:把新 `GammaFontModule` 反向移植进旧 APK**。
   两个版本的类名、方法签名完全一致,直接替换 smali 即可;
   替换后旧 native 仍走 `GetBuffer` 回退分支,新 native 走新分支,双兼容。
3. **不要改 native 去猜返回值**。`0..4` 与 `48..60` 区间虽然可区分,
   但 `dim` 同时被当作 stride/height,改错会引入越界读;类移植风险低得多。
4. 验证:同一份 1.3.3 资源分别配 1.1.6.6 APK(预期空白)与移植后的 APK
   (预期正常),再配 1.3.3 原生 APK 对照。
