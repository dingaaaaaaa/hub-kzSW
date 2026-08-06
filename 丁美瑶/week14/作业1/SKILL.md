---
name: plot-telemetry-column
description: >-
  解析目录中 Tab 分隔的遥测 txt，按表头分组、按选定列拉曲线，可选解析工程值或
  16 进制原码。触发词：遥测 txt、遥测数据、拉曲线、筛选列、原码/解析、看曲线
  发现异常、data_raw。
---

# 遥测列曲线绘制

解析 `--data-dir` 下所有 `*.txt`，按选定列画曲线。每个采样占两行（共享时间戳）：**解析行**
（工程浮点数 / 中文状态字 / `NaN`）与**原码行**（16 进制）；脚本按"非空 cell 是否全像 hex
token"自动分层，跳过重复表头行。

**多表头**：目录可能有不同表头的文件（同名列在不同表里含义或不同）。默认按表头分组——
同表头合并成一条曲线，不同表头各自子图，不混入无关同名列。`--no-schema-group` 退回旧行为。

依赖：`pip install matplotlib`（仅画图需要；`--list-columns` 不需要）。Agg 后端写 PNG。

```bash
# 列出列名（0 基索引，col 0 = 时间戳），无需 matplotlib
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --list-columns

# 画某列：--mode parsed（默认）/hex；--column 接列名或索引可重复；
# 默认 x 轴为时间戳，便于逐秒看计数类异常；--x-mode index 换回采样序号
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --column 126 --mode parsed --output plots/req.png
```

输出到 `--output`（先建目录，如 `mkdir plots`）。列名含特殊字符建议用索引。
完整说明与示例见 [reference.md](reference.md)。
