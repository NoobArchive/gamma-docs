# .gui — Gamma UI 资源

记录 4CC = `"gui\0"`,常见版本 `0x271d`;旧包还见 `0x271b/0x271c`,使用
`CCommonFileHead`。

- `<=0x271c` 没有新版的两个附加字符串 `s17/s26`(读取顺序是 `s16,s24,s25`);
- `0x271b/0x271c` 的 `more[]` 数量按版本为 4;`0x2719` 及更早为 3;
- 旧版字段差异已在 `parse_gui_tree` 中分支,1.1.6.6 的 49 个 GUI 全部精确闭合。

实测 `etc/gui/cfg/jieya1.gui`:

```
0x00  "gui\0"
0x04  version 0x271D
0x08  depCount = 1
dep tag "tex\0", nameCount = 2
  ""                      (空名, 占位)
  "./../tex/fs_jm008.tex" (UI 图集)
```

**没有 main 段**,全部内容在 `extra` 里。

## 1. 读取器

```
Gamma::CWindowResource::LoadItem(CGUIMgr*, const CCommonFileHead&, const SS&,
                                 CBufFile&, CWindowRootResource*)   @0x2983d4
Gamma::ReadGUIResString(CBufFile&, u32 version, std::string& out, std::string* dict)  @0x297190
Gamma::CGImageList::AddImage(CRenderer*, int id, const char* name,
                             const TRect<float>* src, TRect<float>& dst,
                             j,j,j,j,j,j, f,f)   @0x2889ac
```

`ReadGUIResString`:

```
version > 0x2718:  u32 len + len 字节 UTF-8
否则:              u32 count + count × u16 UTF-16LE
若传了 dict 且字符串长度 > 2: 用 CDictionary 查表做本地化(替换成词条值)
```

## 2. extra 结构(逐窗口递归)

一个 `.gui` = 一棵窗口树;`LoadItem` 读一个窗口,末尾递归读子窗口。

```
u32 nameLen + UTF-8        窗口类名(CGWnd / CGButton / CGCheckButton / CGRichWnd / …)
u32 nameLen + UTF-8        窗口名(运行时用名字查找/引用)

u32 ×5                     f[0..4]:  f[1..4] = x, y, w, h(窗口矩形, 父坐标系)
u32 ×2                     g[0..1](version > 0x2711)
u32                         h        (常用 0xFFFFFFFF)
str                         skin     (皮肤/主题名, 常见 "default"/"微信号")
u32 ×4                     i[0..3]
str, str                   s16, s17
str                        s24
str, str                   s25, s26           (s25 常见音效路径, 如 sound\btn_xuanze.mp3)
str ×4                     more[0..3]         (实测恒为 4 个, 与反编译的 3/4/6 分支不符, 以实测为准)
u32                        f30
str                        fade               (淡入淡出参数, version > 0x2717)

u32 childCount
重复 childCount,i 从 1 开始:
    i != 1: i32 ×4      sub 矩形 (left, top, right, bottom),相对窗口
    u32 imageListCount
    重复 imageListCount:
        u32 a           (listIndex << 16) | 1
        u32 b
        u32 c
        u32 stateCount
        重复 stateCount:                        // 每个 54 字节
            u32 texIndex                         // 索引 .gui 的 tex 依赖表
            f32 ×4  src                          // **源矩形 (l,t,r,b),单位 texel**
            f32 ×4  dst                          // **目标矩形 (l,t,r,b),相对窗口**
            u32 k
            u8  alpha                            // 0..255
            i8  f                                // ×1/128
            u16 ×4  u[0..3]
            u32 u5
        u32 groupCount                            // version > 0x2715
        groupCount × 20 字节                      // SGroup

按深度递归时(根窗口或 version > 0x271A):
    u32 windowCount
    重复 windowCount:
        str name            // 子窗口名(先读名字)
        LoadItem(...)       // 紧接着读该窗口的类名与字段
```

> 校验方式:解析必须以 **恰好读完 extra** 结束。213 个 `.gui` 全部精确闭合(0 残留),
> 因此上面的字段顺序/尺寸是可信的。`more` 的个数以实测 4 为准。

## 3. 语义要点

- `state.src` / `state.dst` 都是 `(l, t, r, b)` 而不是 `(x, y, w, h)`;
  宽度 = `r-l`。9-slice 的九块就是九个 state(角/边/中心),`dst` 尺寸与 `src` 不同时拉伸。
- `state.texIndex` 指向 `.gui` 头部的 `tex` 依赖(常见 2 张:占位 + 真正的图集)。
  纹理索引越界/名称为空时**不一定可见**:静态预览只把 `u[1] & 0x08` 的 state
  当作**纯色块**(登录页大块米黄底、全屏压暗层);其余 `tex=-1` 是动态控件占位,
  不绘制。这个区分来自 `login` 与列表/遮罩层的对照,不是全局固定规则。
- **UV 会环绕**:很多 state 的 `src` 超出图集尺寸(如 u=1152,v=-767,图集 1024²),
  采样时按**整张纹理**循环(`GL_REPEAT`),不是先把矩形裁到图集内再缩放。
  预览器先把纹理按周期平铺成临时画布,再按有符号源矩形裁切,越界部分仍取自图集另一侧。
- 图片列表 `a = (listIndex << 16) | 1`,按钮类用它区分 常态/悬停/按下/禁用 四组图。
- 文字不在 state 里:窗口的 `name` 常常是**本地化 key**,文本在
  `config/dictionary/*.txt`(UTF-16 `id<TAB>文本`),去掉 key 首字符标记后查表即可。
  实测字典共 4594 条;`#cRRGGBBAA…`/`#n` 是富文本颜色标记,渲染前要剥掉。
- **文本颜色不是全局黑,也不是单一默认色**。每个 `SChildImageList` 的
  `b`(ARGB)就是该状态行的文字颜色,`c` 是文字底色/描边色;
  按钮常态通常查 `a=0x00020001` 这一行:
  - 登录按钮 `denglu`:`b=0xFFFFFBE8`(奶白),所以是白字;
  - 普通说明/正文:`b=0xFFA9793C`(棕),所以默认看起来是棕色;
  - 禁用态常见 `b=0xFFFFFFFF`、`c=0x13000000`。
  预览器现在按 `b` 渲染,并用 `c` 的 ARGB(含 alpha)做 1px 八向描边/阴影近似;
  字典里的 `#cRRGGBB` 优先覆盖主色。缺失时才回退到引擎常见棕色。
- **`state.k` = ARGB 染色/透明度**(实测 `0xFFFFFFFF` 白 23810 次、`0xFFC0C0C0` 灰化禁用
  3045 次、`0x75FFFFFF` 半透明白 187、`0xFF00FF00` 绿 68……)。渲染时按 `k` 做乘算染色,
  alpha 乘 `k` 的高字节 —— 之前漏了这一步,灰化/半透明元素会显示成全彩。
- **`state.u` = 标志 + wrap 参数**(对应 `CImageGroup::AddImage` 的 `a7..a11`):

  | 字段 | 含义 |
  |---|---|
  | `u[0]` | 未标定(实测 0/13/14/15,`DrawTo` 未直接使用) |
  | `u[1]` | 标志位:`0x02` 颜色乘算、`0x08` 纯色块、`0x10`/`0x20` **反向**平铺、`0x40` 未定 |
  | `u[2]` / `u[3]` | **平铺周期(纹理像素/texel)**;65535 表示不平铺,实测有 100/1280 等值 |
  | `u5` 低 16 位 | **图集组 id**(`u5` 高 16 位未标定) |

  `u[2]/u[3]` 的完整语义(已由 `login.gui` 实机截图校验):

  1. 周期比例 = `clamp(|u2| / src_width, 1/16, 1)`;`u3` 同理。65535 会被 clamp 到 1,
     也就是整块源矩形只画一次。
  2. **`u[1] & 0x10` 时从源矩形右边往左取一个周期**,`& 0x20` 时从下边往上;
     否则从左/上开始。`login` 背景的 `u2=100` 就是“从 `src.r=308` 向左取 100px”,
     即图集 `x=208..308`、`y=529..737` 的格子纹理。
  3. 目标端每个周期宽度 = `dst_width * 比例`;正/负方向分别从目标左/右开始铺。
  4. 源坐标越界按**整张图集**循环,所以 `login` 的 `src.x=-1180..308` 是合法的。

  `login.gui` 的顶部/底部背景就是这样由 100px 周期横向重复 15 次得到的;
  `--no-wrap-tile` 只是调试开关,会退回“整块缩放一次”。
- **`src` 矩形的负宽/负高 = 翻转**(不是错误数据!)。引擎把 `(l,t,r,b)` 直接作为
  纹理坐标,`r<l` 即水平镜像、`b<t` 即垂直镜像;UI 用它拼四向箭头、四角、按钮左右边。
  渲染器以 `min/max` 裁剪后再做 `FLIP_LEFT_RIGHT` / `FLIP_TOP_BOTTOM`。
  全量可见 state 中有 **2979 个**带翻转。
- **`state.f`(signed byte)= 旋转角**。`CImageGroup::DrawTo` 用
  `sin/cos(f/128 * π)` 生成旋转矩阵,即角度 = `f/128 * 180°`;全量有 24 个 state
  的 `f` 非 0,实际绘制到 10 个(其余被状态备选/越界跳过)。
- **`SGroup`(20 字节)** = `{u16 id, u16 ?, f32 ×4 rect}`,通过
  `CGImageList::AddGroup(renderer, id, -1, u16, u16, const TRect<int>&)` 加入,
  供 `u5` 的组 id 引用(精灵组/序列帧),目前渲染器未使用。
- **窗口 `f[0]` = style 位域**(不是浮点)。`CGWnd::DrawText` 直接测试
  `GetStyle()` 的这几个位,从而决定文本对齐:

  | 位 | 含义 |
  |---|---|
  | `0x01000000` | 水平居中(优先级高于右对齐) |
  | `0x02000000` | 水平右对齐 |
  | `0x04000000` | 垂直居中(优先级高于底部) |
  | `0x08000000` | 垂直底部对齐 |

  判断顺序与 `CGWnd::DrawText` 一致:先测居中位,再测右/底位;都不置位时左/上。
  实测 `0x65000000`/`0x45000000` 都是「水平居中 + 垂直居中」,`0x44000000` 是
  「仅垂直居中」,`0x40020000`(CGRichWnd)是「左上」。
- **文本字号来自 `item.i[0]`**(单位 px,实测 18/25/27/28/30/33/36/45,与控件高度吻合);
  `i[1]` 是字间距,`i[2]` 是**行间距**(加在字号上,不是行高本身:引擎换行步进 =
  `fontSize + rowDistance`),`i[3]` 是 0..15 的 LockFlag(拖拽锁定等,与文本无关)。
  渲染时用 `i[0]` 作字号,普通控件超宽才等比缩小。
- **`CGRichWnd` 自动换行**:按字符/英文单词贪心断行到窗口宽度,行高用
  `i[0] + i[2]`;支持 `\n` 与富文本 `#r` 换行。213 张预览中 975 段文字
  (含 311 字的金币规则长文)已能正确排版。
- **同矩形 `windows` = 运行时状态备选**:同一父节点下若多个子窗口矩形完全一致
  (好友/世界/自己、段位 0..15、wifi 1/2/3 格、不同模式结算页),引擎只按当前状态显示一个。
  静态预览默认取「内容最多」的那个作代表,并保留无贴图的透明点击层(如全屏“点击继续”);
  `--all-windows` 可关闭此策略。
- **动态模板窗口**:部分 `windows` 由 Lua 创建/显示,静态预览会看到默认占位内容。
  例 `fsj_jishaxinxi` 的 `jishazhe`/`beijishazhe` 是击杀提示模板;
  `CKillInfoWnd:ShowKillInfo` 里才 `ShowWnd` + `SetWndText`。
  它的默认 state 是 `tex=fs_jm002`、`uv=(-1270,1974,-1025,2042)`,
  越界取模后正好落在“恭喜您获得宝箱”图块 —— 这是**模板占位图**,不是贴图索引解析错误。
- `s25` = 音效路径,`more[1]` = 特效 `.eft` 路径,`skin` = 主题名。

## 4. 预览工具

```powershell
# 打印窗口树(类名/名字/矩形/每个 state 的贴图与源/目标矩形)
python gamma_formats.py gui-info  <x.gui> [--json out.json]

# 合成一张 PNG(需要 Pillow;--tex-root 指向 png_all,--config 指向 config 目录)
uv run --with pillow python gamma_formats.py gui-render <x.gui> <out.png> ^
    --tex-root <png_all> --config <config> [--hide-window jishazhe]

# 批量
uv run --with pillow python gamma_formats.py gui-batch <gui目录> <输出目录> ^
    --tex-root <png_all> --config <config> [-q]
```

覆盖情况(213 个 `.gui`,2026-11 最新一轮):

```
state 总数 10658;可见绘制 8467,另 35 个纯色块;1445 个 UV 越界循环、
2979 个翻转、10 个旋转、975 段本地化文字;940 个同矩形备选窗口被隐藏
```

> 截图对齐:`login.gui` 的 SDK 状态可加
> `--hide-window W_text --hide-window B_ok`(开发版会显示 IP 输入框)。
> `--no-wrap-tile` 只用于对照“整块缩放”的旧行为。

已批量输出到 `_agent-sandbox/gui_preview/`,例:

- `cfg/fsj_denglu.png` — 主界面(含"玩家名字六字"、"活动/分享/设置/新手"、每日赛时间)
- `cfg/fsj_jiesuan.png` — 结算页("您被击败了!"、"回主界面"、"再来一局"、超神/五杀)
- `cfg/messageboxdanxuan.png` — 单选框对话框("本局游戏已经结束!"、按钮"确定")
- `cfg/fsj_saishi_xianshi.png` — 赛事排行榜

## 5. 仍未完全解出的部分

1. `src` 负宽/负高翻转、`f` 旋转、`f[0]` style 对齐、`i[0..3]`、`state.k`、
   `u[2]/u[3]` 平铺、`u1 bit3` 纯色块、文本颜色 `b` 已标定;
   仍缺 `u[0]`、`u[1]` 的 `0x40`、`u5` 高 16 位、`i[1]` 的精确语义;
2. **文本描边**:`SChildImageList.c` 已确认是该状态文字底色/描边色
   (例如 `0x13000000`);预览已用 1px 八向描边近似,精确 shader 混合仍待真机截图校准;
3. **子窗口可见性**:同矩形备选已能选代表,但具体选哪个仍依赖运行时状态
   (例如玩家实际段位、wifi 格数);需要 Lua/控件状态才能精确还原;
4. 按钮底图 / 列表项 / 滚动条等由控件代码动态生成的部分;
5. **动态数值文本**(玩家名、金币、倒计时)是 Lua 在运行时塞进去的,静态预览只能显示
   `.gui` 里带的默认文案;对齐/换行现在按 style 与 `i[]` 还原,不再靠“窗口居中”近似。

## 6. 皮肤选择页模拟(列表项补全)

皮肤格子是 Lua 在运行时按 `.eqp` 的皮肤表动态创建的,`.gui` 里没有对应的静态子节点。
所以单独写了 `skin_sheet.py`:把 `.eqp` 的 29 个皮肤与 29 个 `role/mod/*.ams` **按顺序 1:1 配对**,
用导出的 GLB 软件光栅化渲染正面缩略图,再拼成卡片。

```powershell
uv run --with numpy --with pillow python skin_sheet.py ^
    <male.eqp> <skin_all2(GLB目录)> <输出目录> ^
    [--page <已渲染的皮肤页 png>] [--page-rect x,y,w,h]
```

产物:

- `thumbs/*.png` — 每个皮肤的 3D 缩略图(贴图 + z-buffer,正交投影,偏航 16°)
- `skin_sheet.png` — 29 个皮肤的联络表(名字 + 来源 ams)
- `skin_page_1.png` — 叠在真实 `.gui` 渲染图上的第 1 页(4 张/页)

配对已验证:`p_001`=粉刷匠、`p_001_02`=粉刷匠_黄色、`jingyanhezi`=经验盒子、`obj_001`=金币;
`p_018`(孙悟空)带一根 2377 单位长的金箍棒,缩略图取景按关节范围裁剪。



这四项不影响"看清界面长什么样",但要做到像素级还原还需要继续读
`CGWnd::DrawRect` / `CGImageList::DrawTo` / `CImageGroup::AddImage`。
