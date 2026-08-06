# plot-telemetry-column 参考文档

本文件为 SKILL.md 的按需加载详情，仅在需要完整示例或工作原理时读取，不随 skill 触发
自动加载（节省 token）。

## 工作原理

- 读取 `--data-dir` 下所有 `*.txt`（按文件名排序），除非用 `--files` 指定子集。
- 每个文件：读字节一次、解码一次（utf-8 → utf-8-sig → gbk → latin-1），解析表头，
  按行分类——非空 cell 全是 1..8 位 hex token 的判为原码行，否则为解析行；跳过重复表头行。
- **按表头分组**：以表头元组为 key，同 key 的文件合并；不同 key 各成一个 schema group。
  - 单个 group → 单张图（与旧行为一致）。
  - 多个 group → 纵向堆叠子图，共享 x 轴；每个子图标题标注 schema 列数、文件数、行数。
  - `--no-schema-group` 关闭分组：全部文件用第一个文件表头拼接（旧行为，可能混入无关同名列）。
- 列解析：`--column` 可传索引或列名。按列名时，**每个 group 各自在自己的表头里找**，
  找不到的 group 跳过并告警；按索引时，各 group 用该索引（若在范围内）。
- `--mode parsed`：数值 cell → float；非数值 cell（如 `加电`/`断电`/`健康`）→ 按首次
  出现顺序编码为整数画阶梯曲线，映射打印并标注；`NaN`/空 → 曲线缺口。
- `--mode hex`：`int(v, 16)` 转整数；非 hex cell → 缺口。
- `--x-mode time`：用时间戳作 x 轴（datetime 对象，自动旋转日期标签），便于按秒观察
  计数类字段的异常（如本应每秒 +1 却某秒未加）。
- 输出：PNG 写入 `--output`（请先建父目录）。

## 效率优化点（相对旧版）

- 文件只读一次字节、解码一次（旧版按编码尝试多次重读）。
- hex 行判别用 frozenset 集合比较，短路退出（旧版逐 cell 正则）。
- CLI 只构建所需 mode 的池（`parse_file(path, mode=...)`），另一半不分配（webapp 缓存
  仍传 `mode=None` 构建双池，保持兼容）。
- `extract_column` 支持 `with_labels=False`，CLI 画图路径不构建无用 labels 列表。

## 注意事项

- 未传 `--column` 时脚本打印列名列表后退出，不画图——可作为发现步骤。
- 运行前请先创建输出目录，例如 `mkdir plots`。
- 列名含前导空格或特殊字符时按字面匹配，建议改用 0 基索引更稳妥。
- 多表头目录下用列名画图时，不同表头里同名列会被分别画到各自子图，互不干扰。

## 完整用法示例（核心 SKILL.md 仅保留 3 个，其余在此）

在**项目根目录**（即包含 `data_raw/` 的目录）下运行。

1. **列出所有可用列**（推荐先做这一步，无需 matplotlib）：

```bash
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --list-columns
```

2. **按索引画某列的解析值曲线**（时间轴，便于看逐秒异常）：

```bash
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --column 126 --mode parsed \
  --x-mode time --output plots/req.png
```

3. **画同一列的 16 进制原码曲线**：

```bash
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --column 126 --mode hex --output plots/req_hex.png
```

4. **按列名画**（多表头时各 schema 各自找该列名）：

```bash
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --column "电压" --mode parsed --x-mode time \
  --output plots/voltage.png
```

5. **叠加多列**（重复 `--column`，在同一 group 内叠加）：

```bash
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --column 126 --column 127 --mode parsed \
  --x-mode time --output plots/two.png
```

6. **画状态列**（categorical，自动编码为整数阶梯曲线，映射打印并标注）：

```bash
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --column 1 --mode parsed --output plots/state.png
```

7. **只处理指定文件**（相对 `--data-dir` 的文件名）：

```bash
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --files 20250318112637遥测数据.txt \
  --column 126 --mode parsed --output plots/a.png
```

8. **关闭按表头分组**（旧行为：全用第一个文件表头拼接）：

```bash
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --column 126 --mode parsed \
  --no-schema-group --output plots/legacy.png
```
