---
name: auto-debug
description: 基于用户提供的 Bug 描述与日志，自主分析定位问题根因，输出诊断报告与修复方案，经用户确认后执行代码修复的自动化工具。支持 smart-sync 和 sync-core-service 两个项目的特定诊断逻辑。
when_to_use: 当用户描述一个软件 Bug 并提供日志（内联或文件路径）时使用
allowed-tools: Read, Grep, Glob, Bash(git *, rg), Agent, Edit, TaskCreate, TaskUpdate
user-invocable: true
---

# auto-debug Skill

## 概述

本 Skill 实现完整的自动化 Bug 诊断与修复流程：

```
1️⃣ 输入校验与歧义澄清
2️⃣ 环境检查（项目目录、路径可达性）
3️⃣ 日志提取与清洗（子 Agent 执行）
4️⃣ 代码分析与根因定位（子 Agent 执行）
5️⃣ 诊断报告生成 + 修复方案输出
6️⃣ 用户确认（确认/修改/仅分析/否决）
7️⃣ 代码修复执行（用户确认后）
8️⃣ 修复验证（测试/静态检查）
```

**核心原则**：
- 不猜测：任何不确定之处必须与用户确认
- 不擅自写入：分析阶段仅读取，修复需用户明确确认
- 全程可追踪：使用 TaskCreate/TaskUpdate 记录进度
- 上下文隔离：日志提取与代码分析由子 Agent 执行
- 项目强绑定：必须在项目根目录下执行

---

## 1. 配置加载

首先读取配置文件（如果存在）：

| 配置文件路径 | 优先级 |
|-------------|--------|
| `.auto-debug-config.json`（项目根目录） | 1（最高） |
| `~/.auto-debug/config.json`（全局） | 2 |
| 内置默认值 | 3 |

```json
{
  "log_context_lines": 5,
  "sub_agent_timeout": 120,
  "max_matches_per_file": 500,
  "max_total_matches": 1000,
  "sub_agent_result_max_size": "100KB",
  "log_level_filter": "ERROR,WARNING",
  "max_total_duration": 300,
  "file_read_timeout": 10,
  "grep_timeout": 30,
  "max_log_files": 50,
  "enable_commit_comparison": true
}
```

**加载配置的步骤**：
1. 检查项目根目录是否存在 `.auto-debug-config.json`，存在则读取并合并
2. 检查全局配置文件 `~/.auto-debug/config.json`，合并配置（低优先级覆盖）
3. 缺失的配置使用内置默认值

---

## 2. 输入解析

用户的调用格式：

```
/auto-debug "<Bug描述>" [--log <内联日志或路径>] [--time <时间范围>] [--keyword <关键词>]
```

**解析逻辑**：

1. 提取第一个双引号内的内容作为 `bug_description`
2. 从 `--log` 参数判断日志输入方式：
   - 路径以 `D:\` 或 `/` 开头 → `log_path`（文件或目录）
   - 否则 → `inline_log`
3. `--time` → `time_range`（格式：`YYYY-MM-DD HH:MM~YYYY-MM-DD HH:MM`）
4. `--keyword` → `keywords`（逗号分隔）

**如果解析失败或参数缺失**：
- 显示正确用法并暂停
- 提示用户补充必要信息

---

## 3. 阶段 1：输入校验与歧义澄清

**任务创建**：
```bash
TaskCreate -subject "输入校验与歧义澄清" -status "in_progress"
```

**校验内容**：

| 检查项 | 条件 | 不满足时行为 |
|--------|------|-------------|
| Bug 描述非空 | 长度 > 5 字符且含有效汉字/英文 | 暂停，提示"Bug 描述不明确，请提供：1) 现象 2) 预期行为 3) 实际行为"（错误码 E0101） |
| 日志输入存在 | `inline_log` 或 `log_path` 至少一个 | 暂停，提示"必须提供内联日志或日志路径"（E0102） |
| Bug 与日志相关 | 描述中的关键词（文件名、模块名、错误类型）在日志中存在匹配 | 不存在匹配则暂停询问"日志内容与描述是否匹配？请确认"（E0103） |
| 时间范围格式 | 如提供，必须符合 `YYYY-MM-DD HH:MM~...` | 格式错误则暂停提示并给出示例 |

**通过后输出**：
```json
{
  "bug_description": "...",
  "log_source": "inline" | "path",
  "log_path": "可选，当 log_source=path 时",
  "inline_log": "可选，当 log_source=inline 时",
  "keywords": ["关键词1", "关键词2"],
  "time_range": "可选"
}
```

---

## 4. 阶段 2：环境检查

**任务创建**：
```bash
TaskCreate -subject "环境检查" -status "in_progress"
```

**检查项**：

1. **项目目录有效性**
   - 检查当前目录是否存在 `.git`、`package.json`、`pom.xml`、`build.py` 等标识文件
   - 至少存在一个标识文件才视为有效项目
   - 不满足：暂停，提示"当前目录不是有效的项目根目录（未找到 .git / package.json 等标识文件），请切换到项目根目录后重新执行"（E0201）

2. **项目代码可读性**
   - 随机测试读取 2-3 个源代码文件（`*.py`, `*.js`, `*.ts`, `*.cpp`, `*.h`, `*.go`, `*.java` 等）
   - 任何文件无法读取：暂停，提示"项目代码文件无法读取，请检查文件权限"（E0202）

3. **日志路径可达性**（仅当 `log_source=path`）
   - 检查路径是否存在（`test -d` 或 `test -f`）
   - 检查可读权限（尝试读取前几字节）
   - 检查可写权限（尝试在路径下创建临时文件，用于后续输出）
   - 任意检查失败：暂停，提示相应错误（E0203 / E0204）

4. **项目类型识别**
   - smart-sync：`sync/` + `build.py` + `requirements.txt` + `sync/smart_sync.py`
   - sync-core-service：`CMakeLists.txt` + `package.json`（name="node_addon"）+ `src/node_addon/node_addon.cpp`
   - 通用项目：不匹配上述特征

**通过后输出**：
```json
{
  "project_type": "smart-sync" | "sync-core-service" | "generic",
  "project_name": "项目名称（从配置或路径提取）"
}
```

---

## 5. 阶段 3：日志提取与清洗（子 Agent）

**任务创建**：
```bash
TaskCreate -subject "日志提取与清洗" -status "in_progress"
```

**调用子 Agent**：
```bash
Agent(
  description="日志提取与清洗",
  prompt="日志提取子 Agent 指令见下方",
  timeout=120000
)
```

### 子 Agent 指令内容（prompt 模板）

```
# 日志提取任务

## 输入信息
- Bug 描述：{bug_description}
- 日志来源：{log_source}（inline 或 path）
- 日志路径（如果提供）：{log_path}
- 内联日志（如果提供）：{inline_log}
- 时间范围（如果提供）：{time_range}
- 项目类型：{project_type}
- 关键词：{keywords}
- 配置：
  - 上下文行数：{log_context_lines}
  - 最大匹配行数（单文件）：{max_matches_per_file}
  - 总匹配行数上限：{max_total_matches}
  - 日志级别过滤：{log_level_filter}
  - 最大分析文件数：{max_log_files}

## 任务要求

### 如果 log_source = "inline"
1. 内联日志已经是关键信息，直接进行结构化：
   - 提取时间、级别、文件名、行号、消息
   - 保留原始文本
2. 无需文件发现和检索

### 如果 log_source = "path" 且 log_path 是文件
1. 读取该文件（注意：该文件可能很大，使用 Grep-first 策略）
2. 关键词检索：
   - 使用 Grep 工具搜索关键词（从 bug_description 和提供的 keywords 提取）
   - 按日志级别过滤（优先 ERROR, WARNING）
   - 如提供 time_range，过滤时间
3. 上下文提取：对于每个匹配行，提取前后各 `{log_context_lines}` 行
4. 去重：合并重叠的上下文区间（两段起始行差 < 2 × `{log_context_lines}` 时合并）
5. 如果匹配行数 > `{max_matches_per_file}`，执行聚合摘要：
   - 按时间排序，汇总关键事件
   - 保留最重要的 100 个匹配点
   - 在输出中标注"已聚合摘要"
6. 写入提取结果文件：在 `log_path` 下创建 `auto_debug_extracted_{YYYYMMDD_HHMMSS_ffffff}.log`
7. 返回结构化结果给主 Agent

### 如果 log_source = "path" 且 log_path 是目录
#### 步骤 A — 日志文件发现
1. 使用 Glob 工具发现目录下所有日志文件（`.log`, `.txt`），对于 smart-sync 还包括 `.log.1` ~ `.log.10`
2. 按修改时间降序排序
3. 只分析最近 `{max_log_files}` 个文件（如 50 个）
4. 记录跳过了哪些旧文件

#### 步骤 B — 智能检索与过滤（对每个文件）
1. 使用 Grep 工具搜索关键词（空格分离的多个关键词）
2. 按日志级别过滤（ERROR, WARNING 优先）
3. 如提供 time_range，过滤时间
4. 记录每个文件的匹配行数

#### 步骤 C — 信息提取
1. 对每个匹配行，提取前后各 `{log_context_lines}` 行上下文
2. 合并重叠的上下文区间
3. 如果总匹配行数 > `{max_total_matches}`，执行聚合摘要：
   - 按时间排序，提取关键事件时间线
   - 丢弃次要信息，保留最重要的匹配点
4. 提取错误堆栈信息（连续以 "at " 或 "Traceback" 开头的行）
5. 提取相关变量值（如文件路径、ID 等）

#### 步骤 D — 项目特定增强（仅当 project_type 为 smart-sync 或 sync-core-service）

**对于 smart-sync**：
1. 从提取的日志中搜索 `smart_sync.py[line:66] - meta: {"commit_hash": "..."}`
2. 提取 commit_hash（最长 40 位 hex）
3. 执行 commit 比对：
   - 在项目根目录执行 `git rev-parse HEAD` 得到当前 HEAD commit
   - 如果当前 HEAD 在 commit_hash 中 → 日志来自当前分支
   - 否则执行 `git branch -r --contains <commit_hash>`（只查本地已拉取分支）
   - 根据结果，记录日志对应的分支信息
4. 如果 commit 属于 `production_hubble_scatter` → 记录"散落同步版本"
5. 如果 commit 属于 `production_hubble_gray` → 记录"普通同步版本"

**对于 sync-core-service**：
1. 搜索启动行：`init addon, commit_id: {hash}` 或 `init addon, version:{ver}`
2. 如果匹配到 commit_id（8 位）：
   - 执行 `git rev-parse HEAD` → 截取前 8 位比对
   - 不一致 → `git branch -r --contains <full_hash>` 查找分支
3. 如果匹配到 version（如 `2.0.6`）：
   - 在 `git log --oneline` 和 tag 中搜索匹配的版本标签
   - 找到则记录，找不到则标注"版本号为手动维护，无法自动关联到具体 commit"
4. 检查 DEBUG 日志可用性：查找 `logging.conf` 文件，如果不存在或内容不含 `DEBUG=TRUE` → 记录"DEBUG 日志未启用"

**对于通用项目**：跳过步骤 D

#### 步骤 E — 结果输出
1. 将提取的上下文写入文件到原日志路径：`auto_debug_extracted_{YYYYMMDD_HHMMSS_ffffff}.log`
2. 返回结构化数据给主 Agent（JSON 格式，不超过 100KB）：

```json
{
  "extracted_file_path": "完整路径",
  "total_matches": 匹配行数（聚合后）,
  "log_entries": [
    {
      "timestamp": "时间字符串",
      "level": "ERROR/WARNING/...",
      "file": "文件名",
      "line": 行号,
      "message": "消息内容",
      "context": "前后各 N 行的完整上下文"
    }
  ],
  "commit_info": {
    "found": true/false,
    "hash": "如果找到",
    "branch": "对应分支（如果找到）",
    "note": "散落同步版本/普通同步版本/其他备注"
  },
  "aggregated": true/false,
  "warnings": ["警告信息列表"]
}
```

3. 如果结果大小超过 `{sub_agent_result_max_size}`，添加 `"summary"` 字段进行摘要：
   - 时间范围
   - 错误级别分布
   - 关键文件列表
   - 可能的相关事件概述

## 约束
- 使用 Grep-first 策略，绝不全文读取大日志文件（除单文件且小于 100MB）
- 单次 Grep 超时 `{grep_timeout}` 秒，超时则跳过该文件
- 任何文件读取失败（权限、损坏）均跳过并记录警告
- 保持输出小于 100KB，必要时进行聚合
```

---

## 6. 阶段 4：代码分析与根因定位（子 Agent）

**任务创建**：
```bash
TaskCreate -subject "代码分析与根因定位" -status "in_progress"
```

**调用子 Agent**：
```bash
Agent(
  description="代码分析与根因定位",
  prompt="代码分析子 Agent 指令见下方",
  timeout=120000
)
```

### 子 Agent 指令内容（prompt 模板）

```
# 代码分析任务

## 输入信息
- Bug 描述：{bug_description}
- 提取的日志信息：{sub_agent_3_result}（阶段 3 返回的 JSON）
- 项目类型：{project_type}
- 配置：
  - 文件读取超时：{file_read_timeout}
  - 上下文行数：{code_context_lines}

## 任务要求

### 步骤 1：定位出错位置
从日志信息中提取明确的文件名和行号：
- 日志格式中的 `filename[line:XX]`（Python 项目）
- 或 `filename:XX`（C++ 项目）
- 或通过堆栈信息推断

如果有多个错误位置，按优先级排序（ERROR > WARNING）。

### 步骤 2：读取出错代码
对每个候选位置：
1. 使用 Read 工具读取该文件（设置行数限制：前后各 `{code_context_lines}` 行）
2. 注意：单文件读取超时 `{file_read_timeout}` 秒，超时则跳过

### 步骤 3：分析调用链
对每个出错代码位置：
1. 向上追溯：找到调用该函数的上级函数（搜索定义和调用点）
2. 向下查看：该函数调用的下级函数
3. 关键数据：识别函数参数、返回值、全局变量

### 步骤 4：分析数据流
追踪关键变量的来源与去向：
1. 变量如何被赋值？（来自参数、全局、配置、数据库）
2. 变量如何使用？（传递给其他函数、用于条件判断、写入文件）
3. 变量是否为空或边界值？（检查 null/None、空列表、负数等）

### 步骤 5：识别根因模式
与日志中的错误消息比对，确定 bug 类别：

| 类别 | 典型模式 | 确认线索 |
|------|----------|----------|
| 逻辑缺陷 | 条件判断错误、循环边界错误、状态机转换遗漏 | 代码逻辑与预期不符，日志显示"target already exists"等 |
| 遗漏场景 | 未处理的边界条件、空值未处理、异常未捕获 | 日志显示"AttributeError: 'NoneType' object"等 |
| 并发问题 | 竞态条件、死锁、资源未释放 | 日志中有多个进程/线程 ID，出现"Skipping cleanup, lock held"等 |
| 数据问题 | 数据格式不匹配、编码问题、序列化错误 | 日志显示 JSON decode error、Unicode error 等 |
| 配置错误 | 环境变量缺失、配置项冲突 | 日志显示"missing required configuration"等 |
| 集成问题 | API 调用方式错误、接口参数不匹配 | 日志显示 HTTP 4xx/5xx、参数验证失败等 |
| 性能问题 | 内存泄漏、无限循环、资源耗尽 | 日志显示内存增长、执行时间过长等 |

### 步骤 6：验证分析结论
1. 在代码中找到支持当前假设的证据（如特定的条件分支）
2. 检查是否有反驳证据（如分支已被注释掉）
3. 如果存在多个可能的根因，全都列出并标注各自的置信度

### 步骤 7：不确定时暂停
如果出现以下情况，暂停分析：
- 无法确定唯一的根因（多个可能性且无法排除）
- 日志与代码严重不符（如日志显示的版本与当前代码 commit 不匹配）
- 需要用户提供更多信息（如具体的操作步骤、配置值）

暂停时，向用户说明当前瓶颈和需要补充的信息。

## 输出格式（返回主 Agent）

```json
{
  "analysis_summary": "分析概要，一句话",
  "root_cause": {
    "category": "逻辑缺陷/遗漏场景/并发问题/...",
    "description": "根因详细描述",
    "confidence": 85,
    "evidence": [
      {
        "file": "文件路径",
        "line": 行号,
        "code_snippet": "相关代码片段",
        "explanation": "为什么这里显示了问题"
      }
    ]
  },
  "alternative_causes": [
    {
      "category": "其他类别",
      "description": "替代解释",
      "confidence": 60,
      "evidence": [...]
    }
  ],
  "suggested_fix": {
    "description": "修复思路简述",
    "approach": "修改方式（如：修改条件判断、添加异常处理、释放资源等）",
    "affected_files": [
      {
        "path": "文件路径",
        "change_type": "修改类型（edit/insert/delete）",
        "line_range": "起始行-结束行",
        "current_content": "当前内容片段（用于匹配定位）",
        "proposed_change": "建议的修改内容"
      }
    ]
  },
  "uncertainties": ["不确定性列表，如有"],
  "needs_user_input": ["需要用户补充的信息，如无则省略"]
}
```

## 约束
- 仅读取代码，不修改任何文件
- 注意文件大小：超过 1MB 的文件仅读取出错位置附近 ±200 行
- 如果 Read 超时 `{file_read_timeout}` 秒，记录警告并跳过该文件
- 返回数据必须小于 `{sub_agent_result_max_size}`，必要时省略次要证据
```

---

## 7. 阶段 5：诊断报告生成与修复方案

**任务创建**：
```bash
TaskCreate -subject "诊断报告生成与修复方案" -status "in_progress"
```

**处理逻辑**：

1. 收集阶段 3 和阶段 4 的结果
2. 识别项目类型和分支信息（从阶段 3 结果中提取 `commit_info`）
3. 计算置信度评分（见下方公式）
4. 生成诊断报告 markdown
5. 同时展示给用户并保存到文件

### 置信度评分算法

```
score = 日志证据充分度 × 40% + 代码分析确定度 × 35% + 复现可能性 × 25%

日志证据充分度：
- 匹配到明确错误日志（ERROR堆栈）→ 100
- 仅匹配到 WARNING → 70
- 仅匹配到 INFO → 30
- 无明确错误日志 → 0

代码分析确定度：
- 根因置信度 ≥ 90 → 100
- 根因置信度 ≥ 70 → 80
- 根因置信度 ≥ 50 → 60
- 根因置信度 < 50 → 30
- 找不到根因 → 0

复现可能性：
- 有完整复现步骤 → 100
- 能推断出复现步骤 → 70
- 复现条件不明确 → 30
- 无法复现 → 0
```

### 报告模板（ markdown）

```markdown
# Bug 诊断报告

## 基本信息
- 生成时间：{YYYY-MM-DD HH:MM:SS}
- 项目：{project_name}
- 当前分支：{git_branch}（通过 `git rev-parse --abbrev-ref HEAD` 获取）
- Bug 描述：{bug_description}

## 问题摘要
{一句话总结根本原因（root_cause.description 的简单版）}

## 根因分析

### 出错代码位置
{列出所有相关文件:行号}

#### 代码片段
```python
# 文件:path/to/file.py:XX
def buggy_function(...):
    # 相关代码
    if condition:  # 问题所在
        ...
```

### 错误触发条件
{描述什么情况下触发该 bug}

### 数据流/控制流分析
{简要描述数据如何流动、控制如何转移，导致错误发生}

### 根因分类
{从 categories 中选择最匹配的一个：逻辑缺陷 / 遗漏场景 / 并发问题 / 数据问题 / 配置错误 / 集成问题 / 性能问题}

## 影响范围
- **直接影响**：{哪些功能/模块受影响}
- **潜在影响**：{修复可能引起的连锁影响（如：修改了公共函数可能影响其他调用点）}

## 复现步骤
{如果能为可复现，则列出步骤，否则标注"部分推断"并说明依据}

1. 前置条件
2. 操作步骤
3. 预期结果 vs 实际结果

## 修复方案

### 方案描述
{修复思路说明，用自然语言描述}

### 代码变更
```diff
--- a/path/to/file.py
+++ b/path/to/file.py
@@ -XX,7 +XX,7 @@
  上下文行
- 要删除或修改的行
+ 要新增或修改为的行
  上下文行
```

{可能需要多个 diff 块，每个文件一个}

### 修改文件列表
- `{file_path}`: {修改说明}
- ...

## 置信度评分
**{score}/100**

评分依据：
- 日志证据充分度：{高/中/低}（{分?}）
- 代码分析确定度：{高/中/低}（{分?}）
- 复现可能性：{高/中/低}（{分?}）

## 项目特定信息
{仅适用于 smart-sync / sync-core-service：
- 日志中的 commit_hash：{hash}
- 对应分支：{branch}
- 版本备注：{如"散落同步版本"}或"此日志对应的 commit 不在当前分支中，建议切换到对应分支后重新分析"
}

## 疑问与不确定
- {列出所有 uncertainties}
```

### 报告输出
- 在终端/对话中展示完整 markdown
- 同时写入文件：`auto_debug_report_{YYYYMMDD_HHMMSS}.md` 到用户指定的日志路径（如果不可写，仅终端展示并提示）

### 用户确认环节

展示报告后，**必须等待用户决策**才能继续。

提供选项：
1. **确认修复** — 批准方案，进入阶段 6
2. **修改方案** — 用户提出调整意见（如：添加入参检查、使用不同修复策略）
3. **仅采纳分析** — 认可诊断但暂不修复
4. **否决** — 不认可分析结果，流程终止

**用户确认超时**：不设硬性超时，流程挂起，等待用户回来后继续（AC-07-6）。

---

## 8. 阶段 6：代码修复执行

**任务创建**：
```bash
TaskCreate -subject "代码修复执行" -status "in_progress"
```

**前置条件**：阶段 5 中用户选择了"确认修复"。

**执行规则**：

1. 严格按照阶段 4 返回的 `suggested_fix.affected_files` 列表依次执行
2. 对于每个修改：
   - 使用 `Edit` 工具，指定 `file_path`、`old_string`、`new_string`
   - 如果修复方案涉及多处修改，按顺序逐个执行
   - 每个 `Edit` 成功后更新任务状态
3. 如果执行中发现：
   - 文件内容与预期不符（如 `old_string` 无法匹配）→ 暂停，提示用户（E0601）
   - `Edit` 工具报错 → 暂停，展示错误信息（E0602）

**额外约束**：
- 不得修改范围外的文件
- 每处修改保留上下文，便于 Review
- 如果修复失败，流程暂停等待用户决策（回滚或调整方案）

---

## 9. 阶段 7：修复验证

**任务创建**：
```bash
TaskCreate -subject "修复验证" -status "in_progress"
```

**验证方式**（按项目实际情况选择）：

| 验证项 | 执行方式 |
|--------|----------|
| 语法检查 | Python: `python -m py_compile <files>`；C++: 主要编译器；Node: ESLint |
| 单元测试 | `pytest`, `npm test`, `make test` 等 |
| 静态分析 | `flake8`, `pylint`, `eslint`, `cppcheck` 等 |
| 集成测试（如项目有） | 执行集成测试套件 |

**判断项目测试命令的方式**：
1. 如存在 `pytest.ini`、`tox.ini`、`setup.py` → Python 项目，用 `pytest`
2. 如存在 `package.json` 的 `scripts.test` → Node 项目，用 `npm test`
3. 如存在 `Makefile` 的 `test` 目标 → 用 `make test`
4. 如不存在测试配置 → 降级为仅执行语法检查（如可用）

**结果处理**：
- 全部通过 → 任务完成，输出成功信息
- 存在失败 → 输出失败详情（失败用例、错误信息），提示用户选择：
  - **回滚** — 将修改的文件恢复到修复前的状态
  - **继续修复** — 基于失败结果调整修复方案
  - **终止** — 结束流程，保留当前修改状态

**回滚动作**：
1. 从 Git 读取修复前的文件版本（如果项目有 Git 且有 commit）
2. 使用 `Edit` 恢复原内容
3. 或如果未 commit，提示"项目未 commit，无法自动回滚，请手动处理"

---

## 10. 错误码与中断处理

参考规格文档第十二章的完整错误码体系。关键点：

| 阶段 | 错误 | 行为 |
|------|------|------|
| 1 | E0101-E0103 | 暂停，提示用户补充 |
| 2 | E0201-E0204 | 暂停，提示环境问题 |
| 3 | E0301 超时 → 主 Agent 评估部分结果；E0303 匹配超限 → 触发聚合摘要 |
| 4 | E0401 超时 → 评估部分结果；E0402 无法定位 → 请求补充 |
| 5 | E0501 写入失败 → 重试一次，失败则仅终端展示 |
| 6 | E0601-E0602 | 暂停，提示用户 |
| 7 | E0701 验证失败 → 等待用户决策；E0702 回滚失败 → 提示手动处理 |

---

## 11. 进度追踪

每个阶段开始时：

```bash
TaskCreate -subject "<阶段名称>" -status "pending"
```

阶段开始时更新：
```bash
TaskUpdate -taskId "<id>" -status "in_progress"
```

阶段完成时：
```bash
TaskUpdate -taskId "<id>" -status "completed"
```

如果阶段失败（错误码）：
```bash
TaskUpdate -taskId "<id>" -status "completed" -metadata '{"error": "E0301", "message": "日志提取超时"}'
```

用户可通过 `TaskList` 随时查看所有 Task 状态。

---

## 12. 调用示例

```
> /auto-debug "同步重命名后旧文件残留" --log "D:\bug_log\smart_sync\Fangcloud\SmartSync"

[系统] 开始诊断...

[阶段1] 输入校验完成
[阶段2] 环境检查通过，识别为 smart-sync 项目
[阶段3] 日志提取中...（子 Agent 执行）
[阶段4] 代码分析中...（子 Agent 执行）

# Bug 诊断报告

...

是否确认修复？ (确认修复/修改方案/仅采纳分析/否决)
```

---

## 13. 后续迭代建议

- Phase 2：引入 Python 辅助脚本处理复杂日志提取
- 支持更多项目类型的适配
- 实现完整的 test suite
- 添加性能监控和指标收集
- 支持多语言项目代码分析

---
