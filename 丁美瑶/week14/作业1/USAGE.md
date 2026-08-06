# plot-telemetry-column 使用说明

把 `data_raw/` 里的遥测 txt 按列拉成曲线，用来人工看曲线发现异常（如计数类字段本应
每秒 +1、某秒没加即异常）。支持解析工程值与 16 进制原码两层，自动按表头分组，横坐标
默认为时间。

## 1. 它解决什么问题

- 目录下的 txt 相当于**多张不同表头的表**，不同表头可能有相同列名（"电压""电流"
  "遥测请求计数"…）。脚本按表头分组，不会把不同表里同名列的无关数据混到一条曲线。
- 每个采样在文件里占两行（共享时间戳）：**解析行**（工程浮点数 / 中文状态字 / `NaN`）
  与**原码行**（16 进制）。脚本自动分层，按 `--mode` 选一层画。
- 横坐标默认时间戳，便于逐秒观察计数类字段。

## 2. 依赖

```bash
pip install matplotlib
```
仅 `--list-columns` 不需要 matplotlib。脚本用 Agg 后端写 PNG，Windows 自动检测中文字体。

## 3. 快速上手（在项目根目录运行）

```bash
# 第一步：列出所有列名（0 基索引，col 0 = 时间戳），无需 matplotlib
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --list-columns
```

输出形如：
```
# columns in data_raw (from 20250318112637遥测数据.txt):
  [  0] 0
  [  1] 帧头
  ...
  [126] 遥测请求指令计数
# files parsed (2):
  20250318112637遥测数据.txt: 4187 parsed rows
  20250318170100遥测数据.txt: 4178 parsed rows
```

```bash
# 第二步：画某列曲线（横坐标默认为时间）
# 先建输出目录
mkdir plots
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --column 126 --mode parsed --output plots/req.png
```

终端会打印该列的统计，例如：
```
column '遥测请求指令计数' (idx 126): 8365 rows, 8365 numeric, 0 empty
Saved plot -> plots/req.png
files: 2, schema groups: 1, total parsed rows: 8365
```

## 4. 常用参数

| 参数 | 说明 |
|------|------|
| `--data-dir DIR` | txt 所在目录，默认 `data_raw` |
| `--files a.txt b.txt` | 只处理目录内指定文件子集 |
| `--column N` 或 `--column 名` | 要画的列，按索引或列名；**可重复**叠加多列 |
| `--mode parsed` / `--mode hex` | 解析工程值（默认）/ 16 进制原码 |
| `--x-mode time` / `--x-mode index` | 横坐标用时间戳（默认）/ 采样序号 |
| `--output PATH` | 输出 PNG 路径，如 `plots/req.png` |
| `--title TXT` | 自定义图标题 |
| `--list-columns` | 只列出列名后退出，不画图 |
| `--no-schema-group` | 关闭按表头分组，退回旧行为（全用第一个文件表头拼接） |

## 5. 典型场景

### 5.1 看计数类字段是否每秒 +1（找异常）

```bash
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --column 126 --mode parsed --output plots/req.png
```
横坐标是时间。整段概览看不出逐秒细节，**用 webapp（见第 7 节）拖拽缩放到某一小段**
即可逐秒观察某秒是否未加 1。

### 5.2 按列名画（多表头时各 schema 各自找该列名）

```bash
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --column "电压" --mode parsed --output plots/v.png
```
若目录里有多种表头，每个表头里叫"电压"的列会画到各自子图，互不混入。

### 5.3 画 16 进制原码

```bash
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --column 126 --mode hex --output plots/req_hex.png
```

### 5.4 叠加多列

```bash
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --column 126 --column 127 --mode parsed \
  --output plots/two.png
```

### 5.5 只看某个文件

```bash
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --files 20250318112637遥测数据.txt \
  --column 126 --mode parsed --output plots/a.png
```

### 5.6 画状态列（categorical）

状态列取值如 `加电`/`断电`/`健康`，脚本自动编码为整数画阶梯曲线，映射关系会打印
并在图上标注：
```bash
python .cursor/skills/plot-telemetry-column/scripts/plot_column.py \
  --data-dir data_raw --column 1 --mode parsed --output plots/state.png
```

## 6. 按表头分组说明

- 默认开启：以表头元组为 key 分组。
  - 同表头的文件合并成一条曲线。
  - 不同表头各自一个子图（纵向堆叠、共享 x 轴），子图标题标注 schema 列数、文件数、
    行数。
  - 单个 group → 单张图（与旧行为一致）。
- 按列名画时，每个 group 各自在自己的表头里找该列；找不到的 group 跳过并告警。
- `--no-schema-group` 关闭分组：全部文件用第一个文件表头拼接（旧行为，可能混入无关
  同名列）。

## 7. 交互式查看（推荐用于逐秒找异常）

skill 生成的是整段 PNG 概览。要拖拽缩放、逐秒看，用配套 webapp：

```bash
F:\D\miniconda3\envs\py313\python.exe webapp/server.py
```
浏览器打开 http://127.0.0.1:8000/ ，选文件 → 选列 → 绘制，可在曲线上拖拽缩放、双击
重置、悬停看具体值。webapp 复用 skill 的解析器，行为一致。

## 8. 数据格式（脚本自动识别，无需手动）

- Tab 分隔，第一行非空行是表头。
- 之后每个采样两行：解析行 + 原码行，共享同一时间戳（col 0）。
- 脚本按"该行所有非空 cell 是否都像 1..8 位 hex token"判别两层，跳过重复表头行。
- 编码尝试顺序：utf-8 → utf-8-sig → gbk → latin-1。

## 9. 常见问题

- **图是空的 / "No plottable data"**：该列在该 mode 下没有可转数值的 cell。先用
  `--list-columns` 确认列索引，或换 `--mode`。
- **列名找不到**：列名含前导空格或特殊字符时按字面匹配，建议改用 0 基索引。
- **没生成 PNG**：输出目录不存在会失败，先 `mkdir plots`。
- **中文乱码**：脚本会自动找中文字体（Microsoft YaHei / SimHei / Noto 等），无需配置。
- **端口被占用（webapp）**：`Get-NetTCPConnection -LocalPort 8000` 查到 PID 后
  `Stop-Process -Id <PID> -Force`，再重启 server。

## 10. 文件清单

```
.cursor/skills/plot-telemetry-column/
├── SKILL.md          # 常驻加载的精简说明（agent 触发用）
├── reference.md      # 按需加载的完整说明与示例
├── requirements.txt  # matplotlib 依赖
├── USAGE.md          # 本文件：人工使用说明
└── scripts/plot_column.py   # 解析 + 画图脚本
```
