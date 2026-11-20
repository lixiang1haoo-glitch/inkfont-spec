# INKFONT 协议

v1 实施规范 · 2026 年 11 月 20 日

统一 `.inkfont` 用 ZIP 交换中心线字形。JSON 保存同一个字符的多份完整写法及笔序。应用根据轨迹和自身设置计算字距、行高，并处理候选选择、抖动与倾斜。压力、时间、公式度量和历史可按需携带。

本文定义 v1 的字包成员、坐标、扩展和读写规则。附录提供可运行的测试字包生成脚本。

## 字包为什么保存这些数据

| 数据 | 保存位置与理由 |
| --- | --- |
| 字符和候选 ID | 存于索引。轨迹的形状无法可靠确定字符或属于哪份写法。显式索引让应用只读取当前需要的字形。 |
| 有序笔段和点坐标 | 存于 JSON 主轨迹。这是绘制所需的几何。每段单独保存，才能保留抬笔边界、连笔和单点。 |
| 基线和坐标单位 | 由 v1 统一定义。基线无法从最下方的墨迹可靠推断。使用统一坐标后，每字无需重复保存基线和字面高。 |
| 墨迹宽高与字距 | 应用从最左、最右、最高和最低的坐标计算，并按笔宽和排版策略确定留白。新包省略 `width`、`height`、`advance` 等排版度量。 |
| 压力、时间、数学轴和来源 | 这些值通常无法从 x、y 还原。有支持场景时写入可选扩展，普通绘制只读主轨迹。 |
| SVG 与编辑器状态 | SVG 可以从 JSON 生成，按需缓存或附带。编辑画布、缩放和撤销状态由应用存储。 |
| 成员字节数与摘要 | 从成员字节计算并记录在清单里，读取前可检查资源预算，读取后可校验字节以及扩展绑定的轨迹。 |

### 绘制与排版的责任

图形一致性的验收使用固定候选、字号和原点。字距算法、整行换行、分页和随机扰动由应用决定。校正后的新字体按当前墨迹测量，文档内的局部校正沿用原版面。

## ZIP 成员与清单

| 成员路径 | 是否必需 | 内容 |
| --- | --- | --- |
| `manifest.json` | 必需 | 格式识别、字体身份和成员校验信息。 |
| `index.json` | 必需 | 字符到候选 ID 数组的映射。 |
| `glyphs/<candidate_id>.json` | 必需 | 每份候选的主轨迹。路径由候选 ID 生成。 |
| `previews/<candidate_id>.svg` | 可选 | 由对应 JSON 派生的预览。绘制和迁移统一从 JSON 读取。 |
| `extensions/` | 可选 | 按用途和版本命名的扩展，以及它们引用的快照。 |

### 清单字段

| 字段 | v1 类型和规则 |
| --- | --- |
| `format` | 字符串，固定为 inkfont。 |
| `format_version` | 整数，固定为 1。应用版本单独维护。 |
| `font_id` | 标准连字符形式的小写 UUID。编辑或另存沿用，派生独立字体时新建。 |
| `name` | 非空字符串，显示名称，最长 128 个 Unicode 标量。 |
| `members` | 对象，以成员路径为键。每项包含 `bytes`（解压后的非负整数字节数）和 `sha256`（64 位小写十六进制）。清单自身不列入。 |

### 字符索引和候选标识

`index.json` 是非空对象，键须为单个 Unicode 标量，空白和控制字符由应用处理。值是非空候选 ID 数组。ID 匹配 `[a-z0-9][a-z0-9_-]{0,63}`，在整包唯一。

```json
{"A":["a-01","a-02"],"。":["stop-01"]}
```

v1 字包往返时应保留候选列表顺序，应用自行选择候选。每个 ID 对应一个 `glyphs/<candidate_id>.json`。修订同一候选沿用 ID，新增候选分配新 ID。

## 主轨迹 坐标与字距

主轨迹对象只包含必需字段 `strokes`。它是按书写顺序排列的非空笔段数组。每段为非空的点数组，每个点为两个有限 JSON 数值组成的 `[x, y]`，绝对值各不超过 10 em。单点笔段有效，相邻笔段分别绘制。

```json
{"strokes":[[[0.1,-0.02],[0.35,-0.7],[0.6,-0.02]],
            [[0.22,-0.3],[0.48,-0.3]]]}
```

### 坐标和对齐

统一采用 em 坐标，X 向右、Y 向下。`Y=0` 为基线，`X=0` 为字形排版原点。字号按同一比例缩放整份候选。保留原有留边、负坐标和字面外笔迹。

```text
page_x = origin_x + x * size_mm
page_y = baseline_y + y * size_mm
```

迁移校验例。InkStruct 原基线 0.88 em，原坐标 (0.2, 0.68) 转为 (0.2, −0.2)。OpenInk 字面高 20 mm，原坐标 (4 mm, 11 mm) 也转为 (0.2, −0.2)。

### 如何从笔迹算字距

程序遍历各点得到 `x_min`、`x_max`、`y_min`、`y_max`。墨迹宽度是 `x_max` 与 `x_min` 的差，乘以字号。应用可以把笔宽和额外字间距加进去，再决定推进量。行高使用纵向墨迹范围和应用的最小行距。

例如 x 范围为 −0.10 到 0.50 em，字号为 10 mm，中心线宽 6 mm。按笔迹紧排时可补偿左侧越界并另加间距。单点和完全竖直的笔迹仍可由应用分配最小步进。

### 坐标精度

默认导出在换算后按 0.001 em 网格量化，正负中点均向远离零的方向舍入。量化保留笔段、点序和点数，允许相邻坐标相同。20 mm 字号下，量化导致的单点平面位移最多约 0.014 mm。

全精度导出可保留更细坐标。v1 读取器接受范围内的有限数值。默认量化只作用于主轨迹坐标。需要研究时可附全精度采样快照。

## 可选扩展

所有应用共享 `extensions/` 下按用途和版本命名的成员。v1 扩展文件均为 JSON 对象，含整数 `schema_version: 1` 和 `entries` 对象。`entries` 的键为当前候选 ID，每项必须包含该记录所对应轨迹的 `trajectory_digest`。

| 文件名 | 记录字段和约束 |
| --- | --- |
| `pressure-v1.json` | `values` 为逐笔逐点压力数组。值为 0 到 1 的有限数值，`null` 表示缺测。`origin` 为逐笔来源数组。 |
| `timing-v1.json` | `t_ms` 为逐笔逐点相对毫秒数组，首个有效点为 0，沿全字笔序非递减。可附同形状的 `pen_state` 数组，值为 `down`、`up`、`end` 或 `null`。分别用 `time_origin` 与 `pen_state_origin` 标注来源。 |
| `math-metrics-v1.json` | `math_axis_em` 为相对 `Y=0` 基线的有限 Y 坐标，允许负值。无标注时省略该记录，支持公式的应用按本地规则定位。 |
| `provenance-v1.json` | `source_kind` 为 `recorded`、`generated`、`reused`、`composed` 或 `unknown`。可附 `generator_version`、`source_candidate_id`、`letter_style`（`upright`、`italic`、`unspecified`）和历史 `review_status`。 |

### 绑定 变更与往返

逐点数组和被绑定的轨迹逐段等长，缺失点用 `null`。来源数组每段一项，取 `measured`、`inherited`、`inherited_retimed`、`synthesized`、`absent` 或 `unknown`。时间及笔状态来源数组随各自通道记录。

`trajectory_digest` 使用被绑定轨迹成员的解压字节 SHA-256。默认绑定当前 `glyphs/` 成员。压力或时间如需绑定历史采样点，可写 `source_trajectory_ref`，指向 `extensions/snapshots/<snapshot_id>.json`。快照沿用 `strokes` 结构，摘要绑定快照。

基础读取器可跳过未知扩展。声明支持无损往返的写入器保留其原字节。校正或重采样后应校验绑定。无法对应当前点序的数据可保存在历史快照中，导出时给出字段损失说明。

统一格式省略旧优先级。迁移时保留可用候选，已有的正斜体标签和历史审查记录可随来源保存。按「旧包转换」排除原有不可用候选，并将排除项写入转换报告。

## 读取 写入与校验

### 基本读取顺序

先检查 ZIP 成员名、类型、声明大小和成员数。按 UTF-8 读取 `manifest.json`，根据格式和版本分派。对照 ZIP 成员和 `members` 校验路径集合，再读取并校验 `index.json` 与候选引用。

浏览时可按需解压当前候选，并校验该成员的字节数和摘要，核对坐标及笔段后交给绘制。完整验证时逐项校验字节数和摘要，检查所有候选和已支持的扩展。基础读取器可跳过未知扩展的内容解析。

### 写入契约

写入器按 UTF-8 输出紧凑 JSON。对每个成员解压后的原始字节记录长度与 SHA-256，再写清单。完整组包后验证临时文件，最后发布新包。文件已存在或验证失败时返回错误，保留源文件和已导入字库。

摘要与 JSON 原始字节绑定。两端可用不同 JSON 序列化器。复制一份未改变轨迹的成员时保留原始字节，可以保留它与扩展的绑定。

### 资源上限

| 检查项 | v1 上限 |
| --- | --- |
| 压缩包及全部解压成员 | 256 MiB 压缩包，1 GiB 全部解压字节。按实际读取字节计数。 |
| 单个成员 | 384 MiB 解压字节，适用于所有成员。 |
| 成员与候选数量 | 最多 60,000 个 ZIP 文件成员，30,000 个候选。 |
| 轨迹点数 | 每候选 100,000 点，主轨迹合计 30,000,000 点。 |

只接受存储或 `Deflate` 压缩的常规文件成员。路径用正斜杠，拒绝绝对路径、空段、`.`、`..`、反斜杠、冒号和控制字符。重复成员（含仅大小写不同）、加密、链接、重复 JSON 键、非法坐标或校验失败均报错。

核心身份按需计算，写入器只保存成员摘要。SHA-256 输入由固定前缀、`font_id` 和排序后的核心成员记录拼接而成。换行用 LF，路径按 ASCII 升序。

```text
payload = "inkfont/1\n" + font_id + "\n"
for path in sorted(["index.json", *main_glyph_paths]):
    payload += path + "\t" + members[path]["sha256"] + "\n"
core_sha256 = SHA256(payload.encode("ascii"))
```

核心成员包括索引和主轨迹，预览与扩展有各自的成员摘要。增加基础读取器必需理解的语义时递增 `format_version`。读取器应拒绝未知版本。

## 读取成本 旧包迁移与验收

### 存储选择

用同一批 4,000 个合成字形测试，每字 40 点，候选与索引相同。ZIP 使用 `Deflate`，内存中打开包，校验已读成员并解析。九次取中位数，时间不含写盘、界面和排版。

| 轨迹组织方式 | 压缩体积 | 读取 20 字 | 读取全包 |
| --- | --- | --- | --- |
| 现行联调方案 每候选一文件 | 1.57 MB | 9.9 ms | 73.9 ms |
| 对照 所有候选共用一文件 | 0.70 MB | 58.9 ms | 50.4 ms |

v1 采用每候选一个成员，便于按需读取、定位和替换。代价是 ZIP 成员开销与全读时间。发布前用两边的实际字库补测峰值内存和导入耗时。

### 旧包转换

| 旧包标识 | 换算到新坐标 |
| --- | --- |
| `inkstruct_font`<br>`format_version` 为字符串 1.0 | X 沿用 em 坐标，Y 减去候选的原基线。数学轴原本从基线向上量，换算到 Y 向下的坐标时取相反数。已有压力、时间和来源可迁入对应扩展。 |
| `openink-font`<br>`version` 为整数 1 | 以候选的旧字面高度 H 归一。X 除以 H，Y 除以 H 后减 0.75，保留沿横线排版的相对位置。旧 `advance_mm` 留在源包。 |

新版读取器只接收 v1。首版用独立转换器识别两种旧清单，并另存 v1 包。旧工程继续由各自应用打开。InkStruct 沿用原字体和候选 UUID，按原规则排除停用、拒绝或待复查的候选，并报告排除项。

OpenInk 候选全部保留。用旧包 ID 和候选 ID 得到稳定的 UUID，重复迁移沿用结果。转换说明列出未能保留的字段及行为差异。

```text
font_id = uuid5(NAMESPACE_URL, "urn:inkfont:openink-v1:" + old_package_id)
candidate_id = uuid5(font_id, old_variant_id)
```

### 首次双向验收

旧包验收使用 InkStruct 双写法示例，以及从 OpenInk 公开回归生成的旧包。用[下方的统一样例](#完整联调样例)做双向导入、另存和重开。核对候选、单点、基线、下伸部、负坐标及未知扩展。

[InkStruct 原字体包说明](https://lixiang1haoo-glitch.github.io/InkStruct-Public/developers/inkfont.html) · [OpenInk 原字包协议](https://github.com/EmberCollective/openink-studio/blob/85f70791e2529d90f1110815dd683d24e99bcea1/docs/contracts/OpenInk%E5%AD%97%E5%8C%85%E5%8D%8F%E8%AE%AE.md) · [双写法样例](https://lixiang1haoo-glitch.github.io/InkStruct-Public/developer-examples/two-shapes.inkfont) · [OpenInk 旧包回归](https://github.com/EmberCollective/openink-studio/blob/85f70791e2529d90f1110815dd683d24e99bcea1/desktop/src/inkfont.rs#L576-L610)

## 完整联调样例

以下 Python 脚本仅使用标准库。运行后在当前工作目录生成 `exchange-demo.inkfont`。试验包包含两个 A 候选、一份单点句号和一个压力扩展。

```python
import json
from hashlib import sha256
from zipfile import ZIP_DEFLATED, ZipFile

def encode(value):
    return json.dumps(value, ensure_ascii=False, allow_nan=False,
                      separators=(",", ":")).encode("utf-8")

files = {
    "index.json": encode({"A": ["a-01", "a-02"], "。": ["stop-01"]}),
    "glyphs/a-01.json": encode({"strokes": [
        [[.1, -.02], [.35, -.7], [.6, -.02]],
        [[.22, -.3], [.48, -.3]]]}),
    "glyphs/a-02.json": encode({"strokes": [
        [[.08, 0], [.3, -.64], [.62, 0]],
        [[.2, -.27], [.46, -.27]]]}),
    "glyphs/stop-01.json": encode({"strokes": [[[.1, -.04]]]}),
}
files["extensions/pressure-v1.json"] = encode({
    "schema_version": 1, "entries": {"a-01": {
        "trajectory_digest": sha256(files["glyphs/a-01.json"]).hexdigest(),
        "values": [[.4, .6, .2], [None, .5]],
        "origin": ["measured", "inherited"],
    }},
})
manifest = {
    "format": "inkfont", "format_version": 1, "name": "交换样例",
    "font_id": "00000000-0000-4000-8000-000000000001",
    "members": {p: {"bytes": len(b), "sha256": sha256(b).hexdigest()}
                for p, b in files.items()},
}
with ZipFile("exchange-demo.inkfont", "x", ZIP_DEFLATED) as z:
    z.writestr("manifest.json", encode(manifest))
    for path, data in files.items():
        z.writestr(path, data)
```

期望读出 2 个字符、3 个候选和 11 个主轨迹点。第一个 A 的压力数组长度为 3 和 2。句号的单点位于基线以上 0.04 em。

该样例的 `core_sha256` 应为以下值。

```text
31b1ecf657a829dc43255839f97364b9551ecb82a44038387853faa4e641eeec
```
