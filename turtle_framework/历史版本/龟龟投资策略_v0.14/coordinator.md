# 龟龟投资策略 v0.14 — 协调器（Coordinator）

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
┌─────────────────────── 并行启动 ──────────────────────┐
│                                                       │
│  ┌──────────────────────┐   ┌──────────────────────┐  │
│  │  Phase 1: 数据采集     │   │  Phase 2: PDF解析     │  │
│  │  Agent 1              │   │  Agent 2              │  │
│  │                       │   │                       │  │
│  │  输入：股票代码        │   │  输入：PDF文件路径     │  │
│  │        持股渠道        │   │        提取清单       │  │
│  │        采集清单        │   │                       │  │
│  │                       │   │  ⚠️ 仅当有PDF时启动   │  │
│  │  输出：               │   │                       │  │
│  │  data_pack_market.md  │   │  输出：               │  │
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

### 当有 PDF 年报时（并行启动 Phase 1 + Phase 2）

```
# Phase 1 和 Phase 2 并行
Task(
  subagent_type = "general-purpose",
  prompt = """
  请阅读 {phase1_prompt_path} 中的完整指令。
  目标股票：{stock_code}
  持股渠道：{channel}
  将采集结果写入：{workspace}/data_pack_market.md
  """,
  description = "Phase1 数据采集"
)

Task(
  subagent_type = "general-purpose",
  prompt = """
  请阅读 {phase2_prompt_path} 中的完整指令。
  PDF文件路径：{pdf_path}
  公司名称：{company_name}
  将解析结果写入：{workspace}/data_pack_report.md
  """,
  description = "Phase2 PDF解析"
)

# 等待两个都完成后，启动 Phase 3
Task(
  subagent_type = "general-purpose",
  prompt = """
  请阅读 {phase3_prompt_path} 中的完整指令。
  数据包文件：
    - {workspace}/data_pack_market.md
    - {workspace}/data_pack_report.md （若存在）
  将分析报告写入：{workspace}/{company}_{code}_分析报告.md
  """,
  description = "Phase3 分析报告"
)
```

### 当没有 PDF 年报时（仅启动 Phase 1，跳过 Phase 2）

```
# 仅 Phase 1
Task(
  subagent_type = "general-purpose",
  prompt = "... Phase 1 指令 ...",
  description = "Phase1 数据采集"
)

# Phase 1 完成后直接启动 Phase 3（无 data_pack_report.md）
Task(
  subagent_type = "general-purpose",
  prompt = """
  ... Phase 3 指令 ...
  注意：本次分析无用户上传的年报PDF，仅有 data_pack_market.md。
  模块九母公司单体报表数据将不可用，使用降级方案。
  MD&A分析基于WebSearch获取的摘要信息。
  """,
  description = "Phase3 分析报告"
)
```

---

## 异常处理

| 异常情况 | 处理方式 |
|---------|---------|
| Phase 1 无法获取股价/市值 | 终止全部流程，通知用户检查股票代码 |
| Phase 1 财报数据不足5年 | 继续执行，在 data_pack 中标注实际覆盖年份 |
| Phase 2 PDF 无法解析 | 跳过 Phase 2，Phase 3 使用降级方案 |
| Phase 3 某因子触发否决 | 按框架规则停止后续因子，输出否决报告 |
| Phase 3 context 接近上限 | 通过 checkpoint 机制已将中间结果持久化到文件 |

---

## 文件路径约定

```
{workspace}/
├── coordinator.md              ← 本文件（调度逻辑）
├── phase1_数据采集.md           ← Agent 1 的完整 prompt
├── phase2_PDF解析.md            ← Agent 2 的完整 prompt
├── phase3_分析与报告.md          ← Agent 3 的完整 prompt
├── data_pack_market.md          ← Agent 1 输出（运行时生成）
├── data_pack_report.md          ← Agent 2 输出（运行时生成，可选）
└── {公司}_{代码}_分析报告.md     ← Agent 3 输出（最终报告）
```

---

*龟龟投资策略 v0.14 | 多阶段 Sub-agent 架构 | Coordinator*
