# 飞书多维表格写入细节

## lark-cli 身份与路径

### 身份检测
运行以下命令确认当前 lark-cli 登录的用户：
```bash
lark-cli api GET /open-apis/authen/v1/user_info --as user
```
- 返回 `name` 字段即为当前登录用户。
- 沙箱环境的 lark-cli 可能登录的是测试账号（如"来点羊蝎子"），无用户文档权限。

### 路径优先级
1. **使用当前可用且已授权的 lark-cli**：先用 `command -v lark-cli`（Mac）或 `where lark-cli`（Windows）定位；Mac 常见安装位置包括 `/opt/homebrew/bin/lark-cli`、`~/.local/bin/lark-cli`。
2. **沙箱 lark-cli 作为备选**：系统 lark-cli 不存在时使用沙箱版本。
- 先确认身份属于用户本人；不同路径对应不同用户/机器人时不自动切换。权限受阻先交付本地缓存，说明写表尚未完成；只有用户明确要求时才更换身份或修改权限。

### 版本要求
- 使用支持本任务所需命令的 lark-cli；仅在首次使用某命令或参数不匹配时检查 `--help`，不要每条记录重复检查。本文主要示例已对照本机 CLI 帮助核验。
- 查看版本：`lark-cli --version`

## 表格链接解析

### 支持的链接格式
- Wiki 链接：`https://my.feishu.cn/wiki/xxxxx?table=tblxxxxx&view=vewxxxxx`
- Base 直链：`https://my.feishu.cn/base/xxxxx?table=tblxxxxx`
- 两种都可直接用 `+url-resolve` 解析。

### 解析命令
```bash
lark-cli base +url-resolve --url "<表格URL>" --as user
```
返回字段：
- `base_token`：Base 的唯一标识，后续所有操作都需要
- `table_id`：数据表 ID（`tbl` 开头）
- `view_id`：视图 ID（`vew` 开头）
- `wiki_node_token`：如果是 wiki 链接，返回 wiki 节点 token

### 常见错误
| 错误码 | 原因 | 处理 |
|--------|------|------|
| 131006 | 当前用户无权限访问该文档 | 切换 lark-cli 身份，或在飞书中添加协作者 |
| invalid url | URL 格式不正确 | 检查是否为有效的飞书 wiki/base 链接 |

## 字段管理

### 新建数据表或 Base（用户需要时）

已有目标表直接复用。用户指定在已有 Base 中建新表时，用其真实 base_token：

```bash
lark-cli base +table-create --base-token <token> --name "视频记录" \
  --fields @./fields.json --as user
```

`fields.json` 是 11 字段模板的 JSON 数组，首项“标题”为主字段；首次使用时按当前命令帮助确认文件引用方式，不支持 @file 时传入实际 JSON 数组。需要全新 Base 时使用当前 CLI 的 `+base-create --name ... --table-name ... --fields ...`；不误用 BaseApp 的 app-create。已给定 Base 却缺具体表时，只在多个候选无法判定且用户未授权新建时询问。

只保存本地不执行本文件的飞书命令。权限受阻保存缓存并报告，不自动换成另一个用户/机器人身份。

### 查看现有字段
```bash
lark-cli base +field-list --base-token <token> --table-id <id> --as user
```
返回每个字段的 `id`、`name`、`type`。

### 字段定义（基础9字段 + AI扩展2字段）

**基础采集字段（9个，从抖音页面直接抓取）：**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 账号名称 | text | 博主昵称，如"孟湄"；用文本类型避免 select 选项管理，支持任意博主名直接写入 |
| 标题 | text | 视频标题 |
| 介绍 | text | 视频文案/描述 |
| 逐字稿 | text | 完整口播逐字稿 |
| 点赞 | number | 点赞数（整数） |
| 评论 | number | 评论数（整数） |
| 转发 | number | 转发数（整数） |
| 发表日期 | datetime | 发布时间 |
| 视频链接 | text | 视频 URL |

**AI扩展字段（2个，基于逐字稿由AI生成，目标表格存在时才写入）：**

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 选题方向 | select（多选） | 从目标表格已有选项中选择1-3个标签，禁止自行新增选项 |
| 主题总结 | text | 一句话（30-80字）概括视频核心观点，提炼本质不重复标题 |

### 创建字段

#### 单个创建
```bash
lark-cli base +field-create --base-token <token> --table-id <id> \
  --json '{"name":"标题","type":"text"}' --as user
```

#### 批量创建（推荐）
```bash
lark-cli base +field-create --base-token <token> --table-id <id> \
  --json '[{"name":"账号名称","type":"text"},{"name":"标题","type":"text"},...]' \
  --as user
```

#### 字段类型 JSON 格式
- **text**：`{"name":"xxx","type":"text"}`
- **number**：`{"name":"xxx","type":"number"}`
- **datetime**：`{"name":"xxx","type":"datetime"}`
- **select（单选）**：`{"name":"xxx","type":"select","multiple":false,"options":[{"name":"选项A"},{"name":"选项B"}]}`
- **multiselect（多选）**：`{"name":"xxx","type":"select","multiple":true,"options":[...]}`

注意：飞书 API 中多选字段的 type 也是 `select`，通过 `multiple: true` 区分，不是 `multiselect`。

### 重命名字段
```bash
lark-cli base +field-update --base-token <token> --table-id <id> \
  --field-id <field_id> --json '{"name":"新名称","type":"text"}' --yes --as user
```
- `+field-update` 是完整 PUT 语义，必须传入完整的字段定义（name + type），不能只传 name。
- 主字段（第一个字段）不能删除，只能重命名。如果目标表格的主字段是默认的"文本"且为空，应将其重命名为"标题"。

### 转换字段类型
- text → select：直接用 `+field-update` 传入新类型，原有文本值会自动映射为选项（如果匹配）。
- select → text：同样可直接转换；本项目「账号名称」字段已统一为 text，新采集写入无需选项管理。
- 转换后建议用 `+record-get` 抽样验证数据是否正确保留。

### 删除字段
```bash
lark-cli base +field-delete --base-token <token> --table-id <id> \
  --field-id <field_id> --yes --as user
```
- 主字段不能删除，会返回 `unsafe_operation_blocked` 错误。
- 删除前确认字段中没有需要保留的数据。

## 记录写入

### CellValue 格式

| 字段类型 | 写入格式 | 示例 |
|----------|----------|------|
| text | 字符串 | `"你越强势，AI可能越蠢"` |
| number | 数字 | `3091` |
| datetime | 字符串或毫秒时间戳 | `"2026-07-27 20:02"` 或 `1785153720000` |
| select（单选） | 数组（含1个元素） | `["孟湄"]` |
| multiselect（多选） | 数组（含多个元素） | `["AI协作方法","认知思维"]` |
| url/phone | 字符串 | `"https://www.douyin.com/video/xxx"` |

### 批量创建记录
```bash
lark-cli base +record-batch-create --base-token <token> --table-id <id> \
  --json '{"create_records":[{"标题":"xxx","点赞":3091,"账号名称":"孟湄",...},...]}' \
  --as user
```
- 单批最多 200 条记录。
- 超过 200 条时分批写入，批次间短暂等待（避免 1254291 并发冲突）。
- 返回 `record_id_list`，可用于后续验证。

### 逐条更新记录（upsert）
```bash
lark-cli base +record-upsert --base-token <token> --table-id <id> \
  --record-id <record_id> \
  --json '{"标题":"新标题","点赞":9999}' --as user
```
- 带 `--record-id` 时为更新，不带时为创建。
- 适合增量更新单条记录的部分字段。

### 从 JSON 文件读取
当记录数据较大时（如含长逐字稿），建议写入文件后用 `@file` 引用：
```bash
# 写入 records.json
lark-cli base +record-batch-create ... --json @./records.json --as user
```
- `@file` 路径必须是相对于当前工作目录的相对路径，不能用绝对路径。
- JSON 文件需 UTF-8 编码。

## 增量去重逻辑

### 获取已有记录
```bash
lark-cli base +record-list --base-token <token> --table-id <id> \
  --limit 200 --offset 0 --format ndjson --output ./existing-000.ndjson --as user
```
- NDJSON 每行含 `record_id` 和按字段名读取的值，stdout/manifest 提供 `records_count` 与 `has_more`。按当前作者筛选，读取视频链接、账号和需要修复的字段；复用本轮已读数据，不反复导出全表。
- `has_more=true` 时按实际已读条数推进 offset，用不同文件名继续，直到无下一页；返回空页但仍声称有更多时停止并报告映射不完整，不冒充全表已核对。JSON 模式的 limit 为 1–200，不能使用 500。
- 提取规范化视频 ID → record_id 映射。首次写入前建立映射；每批成功后更新本地映射。创建响应不明时按本批视频 ID 复查，避免重复新增。

### 去重判断
```python
existing_urls = set(已有视频链接列表)
new_records = [v for v in 采集结果 if v['url'] not in existing_urls]
```
- 按完整视频 URL 去重（`https://www.douyin.com/video/<video_id>`）。
- 也可按 video_id 去重（从 URL 中提取数字部分）。

### 全量覆盖与补齐

用户要求“全量覆盖 / 重新导入”时，按视频 ID 更新已有记录并新增未存在的视频，保留本轮未采到的旧记录。不要自动先删后建或直接重复追加；已有完整记录默认跳过，逐字稿失败记录在修复后更新同一 record_id。

批量更新示例（每次不超过 200 条）：

```bash
lark-cli base +record-batch-update --base-token <token> --table-id <id> \
  --json '{"update_records":{"rec_existing":{"逐字稿":"真实完整正文"}}}' --as user
```

`+record-upsert` 不会自动按视频 URL 寻找已有记录，带真实 record_id 才是更新。更新响应可能不验证 ID 存在，需要确认时本批统一读回一次；不要每条重新读全表。

## 常见错误与恢复

| 错误码/现象 | 原因 | 处理 |
|-------------|------|------|
| 1254015 字段值类型不匹配 | CellValue 格式错误 | 检查字段类型，单选/多选必须用数组 |
| 1254045 字段名不存在 | 字段名拼写错误或字段未创建 | 先 `+field-list` 确认真实字段名 |
| 1254104 批量超过200 | 单批记录数超限 | 分批写入，每批≤200 |
| 1254291 并发写冲突 | 短时间内多次写入 | 串行写入，批次间 sleep 1-2秒 |
| 91403 无权限 | 当前用户无表格编辑权限 | 切换 lark-cli 身份或添加协作者 |
| 800080207 主字段不能删除 | 尝试删除第一个字段 | 改为重命名主字段 |
| 日期字段格式错误 | 日期字符串格式不对 | 用 `"YYYY-MM-DD HH:MM"` 或毫秒时间戳 |
| `--json @file` 路径错误 | 用了绝对路径 | 改为相对于当前目录的相对路径 |

## 验证写入结果

写前对缓存统一校验一次：真实视频链接、字段类型、逐字稿来源与状态。以主文件 Step 3 的完整性规则为准，正常正文中的关键词和短正文不单独判失败。可确定的无口播、主动跳过、原文缺介绍或获取失败允许留空并记录原因；未知互动数不填 0。

创建成功响应和返回的 record_id 可确认新增，无需每条读表。更新响应不足以确认、字段被忽略或请求超时结果不明时，完成本批后按本批记录统一读回一次；短暂等待后最多复查一次。正文仍失败只修复一次并记录原因，不要求“所有字段全通过”才交付。

- 更新失败保留完整缓存，下次按 record_id 恢复。
- 创建结果不明时先查询本批视频 ID，存在则更新/复用，不盲目重建。
- 写入受阻交付本地基础数据，报告飞书输出尚未完成；分析可以使用本地缓存继续。

---

## 本地保存分支（用户明确不用表格时）

用户在前置检查中明确表示不用飞书表格时，将采集结果保存为本地文件，采用**每批单独目录 + 每个视频单独 Markdown 文件**的结构。

### 目录结构

```
douyin_data/
└── {安全昵称或作者ID}_{日期}_{批次标识}/
    ├── 01_{video_id}.md
    ├── 02_{video_id}.md
    ├── 03_{video_id}.md
    ├── ...
    └── _all.json          （必需，汇总本批结果与状态）
```

### 批次目录命名

- **根目录**：当前工作目录下的 `douyin_data/` 文件夹。
- **批次子目录**：`{博主昵称}_{采集日期YYYYMMDD}_{视频数量}条/`
  - 示例：`瑶瑶_20261016_batch01/`；同一任务续跑复用，不同任务使用不同批次标识。
- 博主昵称含特殊字符时，替换为下划线或去掉特殊字符。
- 每批采集单独建目录，不同批次互不干扰。

### 每个视频的 Markdown 文件

- **文件命名**：`{序号}_{video_id}.md`
  - 序号按本轮目标顺序从 01 开始，至少两位，超过 99 条用三位；Top 模式按点赞排名，最新模式按真实发布日期。
  - 示例：`01_7655189977860862854.md`、`02_7654993145986808761.md`

- **文件内容格式**：
  ```markdown
  # {视频标题}

  ## 基本信息
  - **账号名称**：{博主昵称}
  - **发表日期**：{YYYY-MM-DD HH:MM}
  - **视频链接**：{url}
  - **点赞**：{点赞数}
  - **评论**：{评论数}
  - **转发**：{转发数}

  ## 介绍/文案
  {视频介绍或文案全文}

  ## 选题方向
  {标签1}、{标签2}、{标签3}

  ## 主题总结
  {一句话主题总结，30-80字}

  ## 逐字稿
  {完整口播逐字稿全文，禁止写占位符}
  ```

- 章节顺序固定：标题 → 基本信息 → 介绍/文案 → 选题方向 → 主题总结 → 逐字稿。
- 逐字稿为空时按状态分别写“无口播内容”“用户跳过”或“获取失败：原因”；不能把获取失败标成无口播。跳过 AI 字段时可省略对应章节。

### 汇总 JSON 文件（必需）

在批次目录下同时保存 `_all.json`，包含所有视频的结构化数据，便于后续程序处理：

```json
[
  {
    "video_id": "7655189977860862854",
    "url": "https://www.douyin.com/video/7655189977860862854",
    "序号": "01",
    "账号名称": "瑶瑶",
    "标题": "你越强势，AI可能越蠢",
    "介绍": "很多人以为把要求说清楚AI就能做好...",
    "逐字稿": "你越强势，AI效果就越蠢。你一定听过一个观点...",
    "transcript_status": "success",
    "transcript_source": "web_fetch",
    "点赞": 3091,
    "评论": 78,
    "转发": 543,
    "发表日期": "2026-07-27 20:02",
    "选题方向": ["AI协作方法", "认知思维"],
    "主题总结": "探讨强势指令对AI效果的负面影响，建议用引导式提问替代命令式要求。"
  }
]
```

### Python 保存示例

```python
import json
import os

# 数据列表（每条包含所有字段）
data = [...]

# 批次目录
author = "瑶瑶"  # 实际使用时将昵称中的路径分隔符替换为下划线
date = "20261016"
batch_id = "batch01"  # 新任务生成新标识，续跑从进度中读取
count = len(data)
batch_dir = f"douyin_data/{author}_{date}_{batch_id}"
os.makedirs(batch_dir, exist_ok=True)

# 序号位数
pad = len(str(count)) if count > 9 else 2

# 为每个视频生成 Markdown 文件
for i, item in enumerate(data, 1):
    seq = str(i).zfill(pad)
    video_id = item["video_id"]
    md_path = os.path.join(batch_dir, f"{seq}_{video_id}.md")

    # 选题方向转为顿号分隔
    topics = item.get("选题方向", [])
    topics_str = "、".join(topics) if topics else "未生成"
    status = item.get("transcript_status", "failed")
    empty_text = {"skipped": "用户跳过", "not_applicable": "无口播内容"}.get(
        status, "获取失败：" + item.get("transcript_error", "未取得完整逐字稿"))
    transcript = item.get("逐字稿") or empty_text

    md_content = f"""# {item['标题']}

## 基本信息
- **账号名称**：{item['账号名称']}
- **发表日期**：{item['发表日期']}
- **视频链接**：{item['url']}
- **点赞**：{item['点赞']}
- **评论**：{item['评论']}
- **转发**：{item['转发']}

## 介绍/文案
{item.get('介绍', '无')}

## 选题方向
{topics_str}

## 主题总结
{item.get('主题总结', '无')}

## 逐字稿
{transcript}
"""
    with open(md_path + ".tmp", "w", encoding="utf-8") as f:
        f.write(md_content)
    os.replace(md_path + ".tmp", md_path)
    with open(md_path, encoding="utf-8") as f:
        assert f.read() == md_content
    # 实际 Agent 此时更新本地文件输出信息；失败逐字稿仍保留 failed 状态。

# 保存汇总 JSON（必需）；实际逐条采集时每条缓存后刷新，而非只在末尾保存
json_path = os.path.join(batch_dir, "_all.json")
with open(json_path + ".tmp", "w", encoding="utf-8") as f:
    json.dump(data, f, ensure_ascii=False, indent=2)
os.replace(json_path + ".tmp", json_path)
```

### 本地保存的数据质量验证

批次完成后统一核对一次：Markdown 数量是否等于实际已输出视频数、序号与视频 ID 对应、JSON 可读且包含完整结果和状态。逐字稿校验复用采集阶段结论，不因正常词语/短正文反复重采。

本地写入错误最多重写缓存一次；仍失败报告具体文件和原因。无口播、主动跳过或获取失败均允许带状态交付，缺介绍/日期等原始信息时如实留空，不无限修复。

### 交付文件

验证通过后，用 `present_files` 工具将批次目录交付给用户：
- 说明批次目录路径、视频数量、文件结构（每个视频一个 Markdown 文件，含基本信息、介绍、选题方向、主题总结、逐字稿）。
- 如有逐字稿获取失败的记录，在交付说明中标注对应文件名。
- 一并交付必需的 `_all.json`；`present_files` 不支持目录时打包或给出可用文件路径，不重新采集。
