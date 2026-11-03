# Gamma 资源格式总览

> 新用户从 **[GETTING-STARTED.md](../GETTING-STARTED.md)** 开始;
> 工具布局/迁移计划见 **[ARCHITECTURE.md](../ARCHITECTURE.md)**;
> 各解析器能力/缺陷见 **[STATUS.md](../STATUS.md)**。
> 数据基准一律以 `V3-1.3.3` 同版本全量解包为准(`Rhythm Painters` 是单机魔改版, 不作依据)。

格式细节:

- 公共头/文本编码:[common.md](common.md)
- `.tex`:[tex.md](tex.md)
- `.r`/filelist:[package.md](package.md)
- 模型/骨架/动画:[ams.md](ams.md)、[ans.md](ans.md)、[anf.md](anf.md)
- UI/特效/材质/皮肤:[gui.md](gui.md)、[eft.md](eft.md)、[mtr.md](mtr.md)、[male.eqp.md](male.eqp.md)、[skins.md](../features/skins.md)
- 历史线索:[legacy-notes.md](legacy-notes.md)

Windows 端(《古域2》)独有容器:

- `.pkg`:[win32/pkg.md](win32/pkg.md) — 头部 + 尾部索引,`pckg`,v1 位置 XOR,文件名仅哈希;
- `.gmz`:[win32/gmz.md](win32/gmz.md) — 更新增量包,待解。


统计来源:`_agent-sandbox/gx_dump`(1.3.3 全量解包)。

| 扩展名 | 数量 | 体积 | 用途 | 状态 |
|---|---|---|---|---|
| `.tex` | 792 | 116 MB | 贴图(ATF: raw / DXT / ETC2 / 双 ETC1) | ✅ 788 PNG + 4 JXR |
| `.anf` | 573 | 5.8 MB | 骨骼动画 clip | ✅ 542/542 解析(时间戳 u8/u16 两种、骨表可短于骨架) |
| `.ams` | 137 | 1.4 MB | 模型(78 role + 59 fx) | ✅ role 78/78、fx 59/59 导出 GLB |
| `.ans` | 93 | 190 KB | 骨架 + 绑定 + clip 表 | ✅ 骨骼/绑定/inverseBind 全解 |
| `.eqp` | 1 | 5 KB | 皮肤/装备清单(UTF-16 名字表) | ✅ `eqp` 命令导出名单 |
| `.gui` | 213 | 2.9 MB | UI 窗口树 + 图集引用 | ✅ 结构 213/213 精确解析,可合成预览图 |
| `.mtr` | 12 | 3 KB | 材质 → `gfx/*.inl` 着色器 | ✅ `extra` 26B 全解(alphaMode/doubleSided) |
| `.eft` | 376 | 1.9 MB | 特效(prop + 粒子 unit,引用 ams/ans/tex) | ✅ prop 754/754、unit 逐字段(5036 个) |
| `.lua` | 179 | 1.2 MB | 游戏逻辑/UI 脚本 | ⬜ 未研究(明文 Lua?) |
| `.txt` | 150 | 0.7 MB | 配置表 | ⬜ 未研究 |
| `.xml` | 6 | 20 KB | 配置 | ⬜ 未研究 |
| `.inf` | 3 | — | 元信息 | ⬜ 未研究 |
| `.264` | 2 | 4.8 MB | H.264 视频 | ⬜ 未处理(可直接封装 mp4) |
| `.mp3` | 38 | 7.4 MB | 音频 | ⬜ 无需处理,直接可播 |
| `.luaproj` | 2 | — | Lua 工程文件 | ⬜ 未研究 |

## 已验证的工具链

```powershell
# 贴图
python gamma_formats.py tex-batch <tex目录> <输出目录>
python gamma_formats.py fl-verify <dir> --filelist filelist_1.3.3.4.7.r

# 模型 / 骨架 / 动画
python gamma_models.py ams2glb <x.ams|dir> <x.glb|outdir> ^
    --ani-root <role/ani> --tex-root <png_all> ^
    [--anim <clip.anf> | --clips] [--fps 30] [--skin 0] [--flip-v]
python gamma_models.py anf-info <x.anf> --ans <x.ans>
python gamma_models.py anf-table <x.ans> [--json out.json]

# 解包 / 校验
python gamma_formats.py unpack <assets_dir> <out_dir> --filelist filelist_1.3.3.4.7.r
python gamma_formats.py fl-verify <assets_dir> --filelist filelist_1.3.3.4.7.r

# 装备清单 + 皮肤名
python gamma_formats.py eqp <male.eqp> [--json out.json] [--txt out.txt] [--root <dump>]

# UI
python gamma_formats.py gui-info <x.gui> [--json out.json]
uv run --with pillow python gamma_formats.py gui-render <x.gui> <out.png> --tex-root <png_all> --config <config>
uv run --with pillow python gamma_formats.py gui-batch <gui目录> <out目录> --tex-root <png_all> --config <config>

# 特效
python gamma_formats.py eft-info <x.eft> [--json out.json] [--units]
uv run --with pillow python gamma_formats.py eft-tex <x.eft> <out.png> --tex-root <png_all>
uv run --with pillow python gamma_formats.py eft-preview <x.eft> <out.png> --tex-root <png_all>

# 材质
python gamma_formats.py mtr-info <x.mtr> [--json out.json]

# 皮肤表 -> 3D 缩略图 / 皮肤选择页模拟
uv run --with numpy --with pillow python skin_sheet.py <male.eqp> <skin_all2> <outdir> --page <皮肤页.png>
```

## 下一步建议(按性价比)

1. **`.eft` 时间轴/发射参数语义**:结构/语义已全解,预览卡已能画粗略粒子云;
   下一步把 `props` 映射成速度/生命周期/颜色曲线,做逐帧模拟;
2. **`.lua` + `.txt`**:UI 文本、数值表、玩法参数;想还原界面文字/状态就必须解;
3. **`.gui` wrap 平铺与运行时状态选择**:剩余少量 wrap/段位/wifi 格数;
4. **`.264`**:封装成 mp4 直接看。
