# `gamma_formats.py` 代码整洁度评估

> 评估对象:`gammaray-py/src/deltaxutils/gamma_formats.py`
> 日期:2026-11-20
> 工具:`ast` 统计 + 当前环境 `ruff check`

## 1. 体量

```text
3,349 行 / 135,465 字节
130 个函数 / 4 个类
```

最大的函数:

| 行数 | 函数 | 说明 |
|---:|---|---|
| 381 | `render_gui` | UI 合成(含 `draw`/`draw_text` 闭包) |
| 214 | `draw`(内部) | 窗口树遍历/状态选择 |
| 151 | `main` | 全部子命令注册 |
| 140 | `_eft_unit` | 13 种 effect unit 分支 |
| 132 | `draw_text` | 文本布局/描边 |
| 108 | `parse_mtr_extra` | 新旧 mtr 分支 |
| 106 | `parse_eqp` | 完整装备表 |
| 105 | `cmd_eft_preview` | 结构预览卡 |

## 2. Ruff 结果

当前配置下 **96 条**,分布:

| 规则 | 数量 | 含义 |
|---|---:|---|
| `BLE001` | 27 | `except Exception` |
| `SIM115` | 25 | `open()` 未用 `with` |
| `RUF046` | 22 | 多余的 `int()` |
| `B023` | 5 | 闭包捕获循环变量 |
| `RUF059` | 5 | 未使用的解包变量 |
| `S110` | 4 | `try/except/pass` |
| `RUF012` | 2 | 类属性可变默认值 |
| `SIM113` / `RUF010` / `FURB188` / `S112` / `I001` / `UP009` | 各 1 | 风格 |

其中 `B023` 是 `point(lambda v: v * a)` 这类**同步调用**,
当前不会真的串值,但建议改成默认参数绑定,避免以后改成异步/延迟执行时踩坑。
`RUF012` 是 `TexFile.LEGACY_BPP` / `LEGACY_BLOCK` 两个只读字典,风险低。

## 3. 优点

- 每个格式都有“恰好读完”的硬校验(`r.p == len(extra)` / `pos == len(main)`);
- 关键字段注释直接标二进制地址与 `sub_xxx`,可回溯;
- 旧版本分支(1.1.x、古域2、0x2711..0x2715)处理得比较完整;
- 单文件即可运行,不依赖复杂框架,适合逆向脚本;
- 错误路径大多会继续处理下一个文件,批量任务不会因单个坏文件中断。

## 4. 主要问题

1. **职责过多**:公共头、tex、r/filelist、eqp、gui、eft、mtr、皮肤、CLI
   全在一个文件;改一个格式要滚 3000 行。
2. **重复实现**:
   - `_GuiReader` / `_EftReader` 各自实现 `u8/u16/u32/f32/string`;
   - `read_text_any` 与 `decode_text` 两套文本解码;
   - `.eqp` 同时保留启发式 `parse_eqp_names`/`parse_eqp_skins` 和完整
     `parse_eqp`;`cmd_deps` 仍走启发式版本。
3. **读写不对称**:`tex-pack` 只包裸 payload,不写 mip 长度前缀;
   `TexFile.build` 不支持旧版/ATF;`rpack`/`texbuild` 已经拆出去,但
   `gamma_formats` 里仍有 `r-pack`/`tex-pack` 的旧入口。
4. **异常粒度粗**:27 处 `except Exception`。对批量工具合理,但会掩盖
   真正的解析回归;建议至少区分 `ValueError` / `struct.error` / `OSError`。
5. **CLI 集成过重**:`main()` 151 行,所有参数和子命令都在这里;
   GammaRay 又维护一份参数组织逻辑,边界容易漂移。
6. **无类型标注/测试**:130 个函数没有类型提示;没有 pytest/金样本,
   回归依赖手工跑 `fl-verify`/`gui-batch`。
7. **导入时副作用**:`import texture2ddecoder` 失败会直接 `print(WARN)`,
   作为库导入时污染 stdout;应改成惰性初始化或 `logging`。

## 5. 建议拆分(与 `todo.md` 一致)

```text
deltaxutils/
  common.py       CCommonFileHead / decode_text / 4CC
  package.py      .r / filelist / 分片
  tex.py          TexFile / ATF / 旧版 / RLE
  gui.py          _GuiReader / parse_gui_tree / render_gui
  eft.py          _EftReader / parse_eft / preview
  mtr.py          parse_mtr_extra
  eqp.py          parse_eqp / skins
  cli_formats.py  原 gamma_formats 的 argparse 子命令
```

`gamma_formats.py` 保留为**兼容 shim**,重新导出旧名字;
`gammaray` 继续从 `deltaxutils` 导入,不直接依赖大文件内部实现。

### 优先级

| 优先级 | 动作 | 风险 |
|---|---|---|
| P0 | `ruff --fix` 的 4 条安全修复;`open` 改 `with`;`eqp` 启发式解析标记 deprecated | 低 |
| P1 | 拆 `common.py` / `package.py` / `tex.py` / `eqp.py`,保留 re-export | 中 |
| P2 | 拆 `gui.py` / `eft.py` / `mtr.py` / `cli_formats.py` | 中高 |
| P3 | 加金样本测试:213 gui、754 eft、67 mtr、5 eqp、pack 往返 | 低 |

## 6. 结论

**功能密度高、逆向注释扎实,但结构整洁度一般(约 6/10)。**
它现在更像“一个长期演进的逆向工作台”,而不是可长期维护的库。
在不改变行为的前提下,先做 P0/P1 的机械拆分,收益最大;
`render_gui` / `_eft_unit` 这类大函数可以在拆分后逐步消减。
