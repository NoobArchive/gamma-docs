# .mtr — Gamma 材质

记录 4CC = `"mtr\0"`,版本 `0x2716` / `0x2717`,使用 `CCommonFileHead`。

## 1. 结构

```
"mtr\0" | version | depCount | deps[] | extraSize | extra[extraSize]
```

- `gfx` 依赖 = 着色器模板(`skeletal.inl`、`shader/skeletal.gfx`、`@UserShader_NNN`);
- `extra` 由 `CMaterialTemplate::_OnReadMainData`(@`0x2dd400`)读取:
  - **0x2717** = 26 字节 + shader 覆盖名长度;
  - **0x2716** = 24 字节 + shader 覆盖名长度;
  - 0x2716 少 `f126/f127` 两个字节。

## 2. extra 字段(版本 0x2717,26 字节)

先读 `shaderOverride`:`u32 n` + n 字节**单字节字符串**(std::string,不是 UTF-16)。
下表的偏移从**字符串结束之后**开始(0x2717 该串为空,所以文件偏移就是 4+表内偏移):

| 偏移 | 类型 | 成员/getter | 含义 |
|---:|---|---|---|
| 0 | u8 | `GetAlphaBlendEnable`(+156) | 1=启用混合 |
| 1 | u8 | `GetSrcBlend`(+116) | D3D9 混合因子:2=One、5=SrcAlpha、6=InvSrcAlpha |
| 2 | u8 | `GetDestBlend`(+117) | 同上 |
| 3 | u8 | +157 | 旧版 zWrite 副本,12/12 样本与 zWrite 同值 |
| 4 | u8 | +158 | 旧版固定字节(样本恒为 5),无 getter |
| 5 | u8 | `GetAlpharef`(+118) | alpha test 阈值(0/76/128/130) |
| 6 | u8 | `GetZTestEnable`(+159) | 深度测试 |
| 7 | u8 | `GetZWR`(+120) | 深度写入 |
| 8 | u8 | `GetZTestFun`(+119) | 0=Never … 4=Less … 7=Always(实测 4) |
| 9 | u8 | `GetCull`(+122) | 1=None(双面)、2=CW、3=CCW |
| 10 | u8 | +160 | 旧版固定字节(样本恒为 3),无 getter |
| 11 | u32 | `GetDiffuse`(+128) | RGBA 字节序:字节 0=R、1=G、2=B、3=A |
| 15 | u8 | `GetSpecularSharp`(+125) | 高光锐度 |
| 16 | u8 | +126 / `specularEnabled` | 255=高光材质(`specular.mtr`) |
| 17 | u8 | +127 / `emissiveEnabled` | 255=自发光材质(`emissive*`/`octopus`) |
| 18 | u8 ×4 | `GetFactor`(+132) | 写入顺序 `(b2,b1,b0,b3)/255` → 存储为 **BGRA**,值域 0..1 |

0x2716 在偏移 15 的 `specularSharp` 之后直接是 4 字节 factor(没有 +126/+127)。

> 读取端还有两处兼容分支:`ver<=0x2713` 会在 diffuse 前多跳 4 字节;
> `ver<=0x2714` 会跳过 `specularSharp` 并在末尾多跳 4 字节。本作的两个版本不走这些分支。

> 需要区分两组:
>
> - `+126/+127` 位于 `SMaterial`(从 `+76` 起 0x48 字节)**内部**
>   (即 `SMaterial+50/+51`),会被 `CreateMaterial` 复制进 `CMaterial`;
>   样本中分别是 `specularEnabled`/`emissiveEnabled` 的 0/255 标记。
> - `+157/+158/+160` 位于 `SMaterial` **之后**,是 `CMaterialTemplate`
>   的旧版/编辑器字段,不进入 GLB。`+157` 在 12/12 个样本里与 `zWrite`
>   同值;`+158` 恒为 5;`+160` 恒为 3。
>
> 因此“5 个模板字节”不需要再猜成 5 个未知渲染参数:
> 2 个是材质开关,3 个是模板遗留字段;它们都不改变 `alphaMode`/`doubleSided`。

## 3. 枚举与 glTF 映射

- `alphaMode`:
  - `alphaBlendEnable==0` → `OPAQUE`;
  - `src==5 && dest==6` → `BLEND`;
  - `src==5 && dest==2` → 加色(`ADDITIVE`;glTF 无对应模式,导出时写 `BLEND` + `extras.blendMode`);
  - `src==0 && dest==0` → `OPAQUE`(自发光/默认着色器);
  - 其他组合 → `BLEND`;
  - `alpharef/255` 作为 `alphaCutoff` 一并输出(当前角色贴图全不透明,主要供后续 alpha test 用)。
- `doubleSided = (cull == 1)`(D3D `CULL_NONE`)。地形翻转剔除的代码也用 2↔3 互换,
  证明 1 是“不剔除”。

## 4. 实测 12 个材质

```
alphablend_ccw_1/2  0x2717  BLEND     cull=1 双面   alpharef=130/0
alphablend_ccw      0x2716  BLEND     cull=3 单面   alpharef=128
default_ccw         0x2716  OPAQUE    cull=3 单面
default_mozhi       0x2717  OPAQUE    cull=3 单面   @UserShader_130
emissive_ccw(_0)    0x2717  OPAQUE    cull=3/1       diffuse=#999999FF
lighttrans          0x2717  ADDITIVE  cull=3 单面   src=5 dest=2, zWrite=0
lighttrans2         0x2717  BLEND     cull=3 单面   src=9 dest=3
octopus             0x2717  OPAQUE    cull=3 单面   @UserShader_135
shui_cp             0x2716  BLEND     cull=3 单面   diffuse=#FF6106F5
specular            0x2717  OPAQUE    cull=3 单面   skeletal_specular.inl
```

## 5. 工具

```powershell
# 解码 extra + 依赖 + alphaMode/doubleSided(可导出 JSON)
python gamma_formats.py mtr-info <x.mtr> [--json out.json]
```

`ams2glb` 会自动读取 AMS 的 `mtr` 依赖(按 `--mtr-root` 或从 AMS 目录向上找 `mat/`),
把 `alphaMode` / `doubleSided` 写进 GLB 材质:

```powershell
python gamma_models.py ams2glb <role/mod> <outdir> --tex-root <png_all> \
    --ani-root <role/ani> [--mtr-root <gx_dump>]
```

## 6. 与模型的关系

```
.eqp -> *.ams -> { mat/*.mtr -> gfx/*.inl , role/tex/*.tex }
```

完整关系可用:

```powershell
python gamma_formats.py deps <role_dir> role_deps.json
```
