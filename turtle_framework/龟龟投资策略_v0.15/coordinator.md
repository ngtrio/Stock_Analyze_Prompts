# 龟龟投资策略 v0.15 — 协调器（Coordinator）

> 本文件为多阶段分析的调度中枢。协调器自身不执行数据获取或分析计算，仅负责：
> (1) 解析用户输入；(2) 判断任务组合；(3) 按依赖关系调度 Sub-agent；(4) 交付最终报告。

---

## 输入解析

用户输入可能包含以下组合：

| 输入项 | 示例 | 必需？ |
|--------|------|--------|
| 股票代码或名称 | `0001.HK` / `长和` | ✅ 必需 |
| 持股渠道 | `港股通` / `直接` / `美股券商` | 可选（未指定则用默认值） |
| PDF 年报文件 | 用户上传的 `.pdf` 文件 | 可选 |

**解析规则**：
1. 从用户消息中提取股票代码/名称和持股渠道
2. 检查是否有 PDF 文件上传（检查 `/sessions/*/mnt/uploads/` 目录中的 `.pdf` 文件）
3. 若用户只给了公司名称没给代码，在 Phase 1 中由 Agent 1 通过 `search_stocks` 确认代码

---

## 阶段调度

```
┌─────────────────────────────────────────────┐
│              用户输入解析                      │
│   股票代码 = {code}                           │
│   持股渠道 = {channel | 默认}                  │
│   PDF年报 = {有 | 无}                         │
└──────────┬──────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│         PDF 年报确保步骤（协调器执行）           │
│                                               │
│  ① 检查用户是否上传了 PDF 年报                  │
│  ② 若有上传：校验是否为最新完整财年年报          │
│     （通过文件名/标题中的年份判断）              │
│  ③ 若无上传 或 上传的并非最新财年年报：           │
│     → 调用 snowball-report-downloader          │
│       自动下载最新完整财年年报                    │
│     → 下载成功：使用下载的 PDF 路径              │
│     → 下载失败：记录失败原因，Phase 2 降级执行    │
│                                               │
│  结果：pdf_path = {已确认的PDF路径 | null}      │
└──────────┬──────────────────────────────────┘
           │
           ▼
┌─────────────────────── 并行启动 ──────────────────────┐
│                                                       │
│  ┌──────────────────────┐   ┌──────────────────────┐  │
│  │  Phase 1: 数据采集     │   │  Phase 2: PDF解析     │  │
│  │  Agent 1              │   │  Agent 2              │  │
│  │                       │   │                       │  │
│  │  输入：股票代码        │   │  输入：PDF文件路径     │  │
│  │        持股渠道        │   │        提取清单       │  │
│  │        采集清单        │   │                       │  │
│  │                       │   │  ⚠️ 仅当 pdf_path     │  │
│  │  输出：               │   │     有效时启动         │  │
│  │  data_pack_market.md  │   │                       │  │
│  │                       │   │  输出：               │  │
│  │                       │   │  data_pack_report.md  │  │
│  └──────────┬───────────┘   └──────────┬───────────┘  │
│             │                          │               │
└─────────────┼──────────────────────────┼───────────────┘
              │     等待全部完成          │
              ▼                          ▼
┌─────────────────────────────────────────────┐
│           Phase 3: 分析与报告                  │
│           Agent 3                             │
│                                               │
│  输入：data_pack_market.md                     │
│        data_pack_report.md（若有）              │
│        策略知识库 + 报告模板                     │
│                                               │
│  输出：{公司名}_{代码}_分析报告.md               │
│                                               │
│  ⚠️ 不调用任何外部数据源                        │
│  ⚠️ 内部设置 checkpoint：                      │
│     每完成一个因子 → 将结论追加写入报告文件       │
│     防止 Phase 3 自身 context compact            │
└──────────┬──────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│           协调器交付                           │
│  1. 确认报告文件已生成                          │
│  2. 返回报告文件链接给用户                       │
└─────────────────────────────────────────────┘
```

---

## Sub-agent 调用指令

> **⚠️ 前置步骤**：协调器在启动任何 Sub-agent 前，必须先执行：
> `mkdir -p {workspace}/{symbol}/`
> 确保输出目录已创建。所有 Sub-agent 的输出路径均指向 `{workspace}/{symbol}/`，**严禁写入策略规则目录**。

### Step 0：PDF 年报确保（协调器在启动 Phase 2 前必须执行）

协调器在启动 Phase 2 前，应确保传入的 PDF 年报为**当前可获取的最新完整财年年报**。

```
# 0-1. 确定目标年份
current_month = {当前月份}
if current_month <= 3:
    target_year = {当前年份} - 2   # 1-3月最新年报可能未发布，取上上年
else:
    target_year = {当前年份} - 1   # 4月及以后，取上一年

# 0-2. 检查用户上传的 PDF
uploaded_pdf = 检查 /sessions/*/mnt/uploads/ 目录中的 .pdf 文件

if uploaded_pdf 存在:
    # 校验文件名/标题中是否包含 target_year
    if uploaded_pdf 文件名含 target_year 或相近年份:
        pdf_path = uploaded_pdf   # 使用用户上传的 PDF
    else:
        # 用户上传的 PDF 并非最新年报，仍尝试自动下载最新的
        → 执行 Step 0-3 自动下载
        if 下载成功:
            pdf_path = 下载的PDF路径
        else:
            pdf_path = uploaded_pdf  # 回退使用用户上传的版本
else:
    # 用户未上传 PDF → 自动下载
    → 执行 Step 0-3 自动下载
    if 下载成功:
        pdf_path = 下载的PDF路径
    else:
        pdf_path = null  # 下载失败，Phase 2 将跳过（降级执行）

# 0-3. 自动下载（通过 snowball-report-downloader 插件）
Skill("snowball-report-downloader:report-download")
# 使用插件下载最新完整财年年报：
#   stock_code = {stock_code}
#   report_type = "年报"
#   year = target_year
#   save_dir = {workspace}/{symbol}/
#
# 下载脚本路径：${PLUGIN_PATH}/skills/report-download/scripts/download_report.py
# 参数：--url <PDF_URL> --stock-code <code> --report-type "年报" --year <year> --save-dir <dir>
#
# 若下载失败，记录失败原因，后续 Phase 2 跳过，Phase 3 使用降级方案
```

### 当 pdf_path 有效时（并行启动 Phase 1 + Phase 2）

```
# 前置：创建输出目录
Bash("mkdir -p {workspace}/{symbol}/")

# Phase 1 和 Phase 2 并行
Task(
  subagent_type = "general-purpose",
  prompt = """
  请阅读 {strategy_dir}/phase1_数据采集.md 中的完整指令。
  目标股票：{stock_code}
  持股渠道：{channel}
  将采集结果写入：{workspace}/{symbol}/data_pack_market.md
  ⚠️ Phase 1 无需执行年报PDF下载步骤（已由协调器完成）。
  ⚠️ 严禁向 {strategy_dir}/ 写入任何文件。
  """,
  description = "Phase1 数据采集"
)

Task(
  subagent_type = "general-purpose",
  prompt = """
  请阅读 {strategy_dir}/phase2_PDF解析.md 中的完整指令。
  PDF文件路径：{pdf_path}
  公司名称：{company_name}
  将解析结果写入：{workspace}/{symbol}/data_pack_report.md
  ⚠️ 严禁向 {strategy_dir}/ 写入任何文件。
  """,
  description = "Phase2 PDF解析"
)

# 等待两个都完成后，启动 Phase 3
Task(
  subagent_type = "general-purpose",
  prompt = """
  请阅读 {strategy_dir}/phase3_分析与报告.md 中的完整指令。
  数据包文件：
    - {workspace}/{symbol}/data_pack_market.md
    - {workspace}/{symbol}/data_pack_report.md （若存在）
  将分析报告写入：{workspace}/{symbol}/{company}_{code}_分析报告.md
  ⚠️ 严禁向 {strategy_dir}/ 写入任何文件。
  """,
  description = "Phase3 分析报告"
)
```

### 当 pdf_path 为 null 时（自动下载失败且无用户上传，降级执行）

```
# 前置：创建输出目录
Bash("mkdir -p {workspace}/{symbol}/")

# 仅 Phase 1（跳过 Phase 2）
Task(
  subagent_type = "general-purpose",
  prompt = """
  请阅读 {strategy_dir}/phase1_数据采集.md 中的完整指令。
  目标股票：{stock_code}
  持股渠道：{channel}
  将采集结果写入：{workspace}/{symbol}/data_pack_market.md
  ⚠️ Phase 1 无需执行年报PDF下载步骤（协调器已尝试下载但失败）。
  ⚠️ 严禁向 {strategy_dir}/ 写入任何文件。
  """,
  description = "Phase1 数据采集"
)

# Phase 1 完成后直接启动 Phase 3（无 data_pack_report.md）
Task(
  subagent_type = "general-purpose",
  prompt = """
  请阅读 {strategy_dir}/phase3_分析与报告.md 中的完整指令。
  数据包文件：
    - {workspace}/{symbol}/data_pack_market.md
  注意：本次分析无年报PDF（协调器自动下载失败，用户亦未上传）。
  仅有 data_pack_market.md。
  模块九母公司单体报表数据将不可用，使用降级方案。
  MD&A分析基于WebSearch获取的摘要信息。
  将分析报告写入：{workspace}/{symbol}/{company}_{code}_分析报告.md
  ⚠️ 严禁向 {strategy_dir}/ 写入任何文件。
  """,
  description = "Phase3 分析报告"
)
```

---

## 单位统一规则 ⭐

**全局单位约定**：

| 数据源 | 原始单位 | 说明 |
|-------|---------|------|
| data_pack_market.md（Phase 1 输出） | **{报表币种}百万元**（如 RMB百万、HKD百万） | yfinance 原始输出单位，币种取决于报表币种 |
| data_pack_report.md（Phase 2 输出） | **{报表原始单位}**（常见：千元 RMB'000、百万元、万元） | 年报原始披露单位，以文件头标注为准 |
| Phase 3 分析报告（最终输出） | **人民币亿元** | 报告展示单位，便于阅读 |

**换算规则**（Phase 3 agent 必须先读取各 data_pack 文件头的单位标注，再选用对应公式）：
- 百万元 → 亿元：**÷ 100**（例：9,890.67百万 = 98.91亿）
- 千元 → 亿元：**÷ 100,000**（例：9,082,254千元 = 90.82亿）
- 万元 → 亿元：**÷ 10,000**（例：908,225万 = 90.82亿）
- 百万元 → 千元：**× 1,000**（跨数据源比对时使用）
- 非人民币币种 → 人民币：**× 分析日即期汇率**（Phase 1 基础信息中标注汇率）

**⚠️ 禁止事项**：
- 禁止将百万元数值直接当作亿元使用（会产生10倍误差）
- 禁止混用不同单位的数值直接运算
- 禁止假设 Phase 2 输出单位固定为千元——必须读取文件头标注
- Phase 1/Phase 2 输出必须在文件头部标注币种和单位

---

## 报表时效性规则

协调器在启动 Phase 2 前，**必须**确保传入的 PDF 年报为**当前可获取的最新完整财年年报**。

**目标年份判断**：
- 若当前日期在 1–3月：`target_year = 当前年份 - 2`（最新年报可能尚未发布，取上上财年年报）
- 若当前日期在 4月及以后：`target_year = 当前年份 - 1`（最新财年年报通常已发布）

**PDF 来源优先级**：
1. **用户上传的 PDF**：若用户上传了 PDF 且经校验确认为 `target_year` 财年年报，直接使用
2. **自动下载**：若用户未上传 PDF，或上传的 PDF 并非最新财年年报，协调器**必须**通过 `snowball-report-downloader` 插件自动搜索并下载 `target_year` 年度年报（详见 Sub-agent 调用指令 → Step 0）
3. **降级执行**：仅当自动下载也失败时，Phase 2 才会被跳过，Phase 3 使用降级方案

**支付率等关键指标必须基于所分析年报中的同币种数据计算**（股息总额与归母净利润均取报表币种），不依赖 yfinance 的 payoutRatio 等衍生字段。

---

## EV口径与现金保护规则摘要

Phase 3 新增以下分析框架（详见 phase3_分析与报告.md）：

1. **合同负债计入类现金**：适用于先款后货/预收制业务（物业管理、白酒、订阅SaaS）。合同负债纳入"超广义现金"口径，用于安全垫和EV计算，不用于可分配现金。
2. **EV口径双轨制**：当广义净现金/市值 > 40% 时，自动触发EV口径穿透回报率计算（R_EV、GG_EV），分母从市值切换为EV=市值−广义净现金。需同时通过"现金分配意愿检验"。
3. **现金保护等级**：按广义净现金/市值比例分四档（无保护<20%、轻度20-40%、强40-60%、极强>60%），对应安全边际折扣从30%递减至15%。
4. **派息可持续年限**：可自由支配现金÷年派息总额，>15年为"极强现金保护"。

---

## 异常处理

| 异常情况 | 处理方式 |
|---------|---------|
| Phase 1 无法获取股价/市值 | 终止全部流程，通知用户检查股票代码 |
| Phase 1 财报数据不足5年 | 继续执行，在 data_pack 中标注实际覆盖年份 |
| Step 0 年报自动下载失败 | 跳过 Phase 2，Phase 3 使用降级方案，通知用户可手动上传年报重新分析 |
| Phase 2 PDF 无法解析 | 跳过 Phase 2，Phase 3 使用降级方案 |
| Phase 3 某因子触发否决 | 按框架规则停止后续因子，输出否决报告 |
| Phase 3 context 接近上限 | 通过 checkpoint 机制已将中间结果持久化到文件 |

---

## 策略目录只读规则 ⚠️

**策略规则目录（如 `龟龟投资策略_v0.15/`）为只读目录，严禁在其中添加、修改或删除任何文件或子文件夹。**

所有运行时生成的文件（中间数据、最终报告等）必须输出到按标的 symbol 命名的独立目录中（见下方文件路径约定）。

违反此规则会导致策略文件污染、版本混乱和文件夹冲突。

---

## 文件路径约定

```
{workspace}/
│
├── 龟龟投资策略_v0.15/            ← 策略规则目录（⚠️ 只读，禁止修改/添加）
│   ├── coordinator.md
│   ├── phase1_数据采集.md
│   ├── phase2_PDF解析.md
│   └── phase3_分析与报告.md
│
└── {symbol}/                      ← 按标的 symbol 命名的输出目录（运行时创建）
    ├── data_pack_market.md        ← Agent 1 输出
    ├── data_pack_report.md        ← Agent 2 输出（可选）
    └── {公司}_{代码}_分析报告.md   ← Agent 3 输出（最终报告）
```

**路径变量说明**：
- `{workspace}`：用户的工作空间根目录（如 `/sessions/xxx/mnt/保利物业/`）
- `{symbol}`：股票代码的纯数字部分（如 `06049`、`0001`），不含 `.HK` 后缀
- `{strategy_dir}`：策略规则目录路径（如 `{workspace}/龟龟投资策略_v0.15/`）

**协调器在启动 Phase 1/Phase 2 前，必须先创建输出目录**：`mkdir -p {workspace}/{symbol}/`

---

*龟龟投资策略 v0.15 | 多阶段 Sub-agent 架构 | Coordinator*
