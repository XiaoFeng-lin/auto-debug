---
name: auto-debug
description: |
  基于用户提供的 Bug 描述与日志，自主分析定位问题根因，输出诊断报告与修复方案，经用户确认后执行代码修复的自动化工具。
  支持 smart-sync、sync-core-service 及通用项目。
  PROACTIVELY activate when: 用户描述软件 Bug 并提供日志（内联或路径）时。
when_to_use: 当用户描述一个软件 Bug 并提供日志（内联或文件路径）时使用
allowed-tools: Read, Grep, Glob, Bash, Agent, Edit, TaskCreate, TaskUpdate
user-invocable: true
---

# auto-debug Skill

## 概述

本 Skill 实现完整的自动化 Bug 诊断与修复流程：

```
1️⃣ 输入校验 → 2️⃣ 环境检查 → 3️⃣ 日志提取（子 Agent）
→ 4️⃣ 代码分析（子 Agent） → 5️⃣ 报告生成 → 6️⃣ 用户确认 → 7️⃣ 修复执行 → 8️⃣ 修复验证
```

**核心原则**：
- **不猜测**：不确定之处必须与用户确认
- **不擅自写入**：分析阶段仅读取，修复需用户明确确认
- **不污染项目目录**：不在项目根目录创建任何非项目文件
- **全程可追踪**：使用 TaskCreate/TaskUpdate 记录进度
- **上下文隔离**：日志提取与代码分析由子 Agent 执行
- **渐进式披露**：项目特定知识按需加载，减少无关上下文

---

## 1. 配置加载

**任务创建**：`TaskCreate -subject "配置加载" -status "in_progress"`

**配置来源**（唯一来源）：
使用 Read 工具读取 skill 安装目录下的 `config/default-config.json`，获取默认配置参数。

**配置参数包括**：
- `log_context_lines`：匹配行前后上下文行数（默认 5）
- `sub_agent_timeout`：子 Agent 超时（默认 300 秒）
- `max_matches_per_file`：单文件匹配上限（默认 500）
- `max_total_matches`：总匹配上限（默认 1000）
- `sub_agent_result_max_size`：子 Agent 返回结果大小上限（默认 100KB），超过则要求摘要
- `log_level_filter`：日志级别过滤（默认 ERROR,WARNING）
- `max_total_duration`：全流程最大耗时（默认 1800 秒 = 30 分钟），超过则在报告中标注
- `max_log_files`：最大分析文件数（默认 50）
- `enable_commit_comparison`：是否启用 commit 比对（默认 true）
- `file_read_timeout`：单文件读取超时（默认 30 秒）
- `grep_timeout`：单次 Grep 超时（默认 30 秒）

**注意**：不再读取项目根目录的 `.auto-debug-config.json` 或全局的 `~/.auto-debug/config.json`。

---

## 2. 输入解析

用户调用格式：
```
/auto-debug "<Bug描述>" [--log <内联日志或路径>] [--time <时间范围>] [--keyword <关键词>]
```

**解析逻辑**：
1. 提取第一个双引号内的内容作为 `bug_description`
2. `--log` 参数：
   - 路径以 `D:\` 或 `/` 开头 → `log_path`（文件或目录）
   - 否则 → `inline_log`
3. `--time` → `time_range`（格式：`YYYY-MM-DD HH:MM~YYYY-MM-DD HH:MM`）
4. `--keyword` → `keywords`（逗号分隔）

---

## 3. 阶段 1：输入校验与歧义澄清

**任务创建**：`TaskCreate -subject "输入校验与歧义澄清" -status "in_progress"`

| 检查项 | 条件 | 不满足时行为 |
|--------|------|-------------|
| Bug 描述非空 | 长度 > 5 字符且含有效汉字/英文 | 暂停，提示 E0101 |
| 日志输入存在 | `inline_log` 或 `log_path` 至少一个 | 暂停，提示 E0102 |
| Bug 与日志相关 | 描述中的关键词在日志中存在匹配 | 不匹配则暂停询问 E0103 |
| 时间范围格式 | 如提供，必须符合 `YYYY-MM-DD HH:MM~...` | 格式错误则暂停提示 |

**通过后输出结构化输入**：
```json
{
  "bug_description": "...",
  "log_source": "inline" | "path",
  "log_path": "可选",
  "inline_log": "可选",
  "keywords": ["..."],
  "time_range": "可选"
}
```

---

## 4. 阶段 2：环境检查

**任务创建**：`TaskCreate -subject "环境检查" -status "in_progress"`

### 4.1 项目目录有效性
检查当前目录是否存在 `.git`、`package.json`、`pom.xml`、`build.py` 等标识文件。至少存在一个才视为有效项目。
不满足：暂停，提示 E0201。

### 4.2 项目代码可读性
随机测试读取 2-3 个源代码文件。任何文件无法读取：暂停，提示 E0202。

### 4.3 日志路径可达性（仅当 log_source=path）
检查路径是否存在、可读、可写。任意失败：暂停，提示 E0203/E0204。

### 4.4 项目类型识别（关键步骤）

1. 使用 Read 工具读取 `config/project-adapters-config.json`，获取适配器注册表
2. 遍历 `adapters` 列表，按 `priority` 降序检查 `match` 规则：
   - `match.directories`：使用 Glob 检查目录是否存在
   - `match.files`：使用 Glob 检查文件是否存在
3. 第一个完全匹配的 adapter 即为当前项目类型
4. **精确验证**：
   - 若匹配到 `sync-core-service`，额外使用 Read 读取 `package.json`，确认包含 `"name": "node_addon"`。若不满足，继续匹配下一个 adapter
5. 若匹配成功且 `adapter.reference` 不为 null：
   - 使用 Read 工具读取对应的 reference 文件（如 `references/smart-sync.md`）
   - 将 reference 内容加入当前上下文
6. 若都不匹配：使用 `fallback`（通用项目，`project_type = "generic"`）

**通过后输出**：
```json
{
  "project_type": "smart-sync" | "sync-core-service" | "generic",
  "project_name": "项目名称"
}
```

---

## 5. 阶段 3：日志提取与清洗（子 Agent）

**任务创建**：`TaskCreate -subject "日志提取与清洗" -status "in_progress"`

**调用子 Agent**：
```
Agent(description="日志提取与清洗", prompt="...", timeout=300000)
```

### 子 Agent Prompt 模板

```
# 日志提取任务

## 输入
- Bug 描述：{bug_description}
- 日志来源：{log_source}
- 日志路径：{log_path}
- 内联日志：{inline_log}
- 时间范围：{time_range}
- 项目类型：{project_type}
- 关键词：{keywords}
- 配置：上下文行数 {log_context_lines}，单文件上限 {max_matches_per_file}，总上限 {max_total_matches}，日志级别 {log_level_filter}，最大文件数 {max_log_files}

## 任务要求

### 内联日志（log_source = "inline"）
直接结构化，提取时间、级别、文件名、行号、消息。无需文件发现。

### 单文件（log_source = "path" 且为文件）
1. 使用 Grep-first 策略，关键词检索（从 bug_description 和 keywords 提取）
2. 按日志级别过滤（优先 ERROR, WARNING）
3. 时间范围过滤（如提供）
4. 对每个匹配行提取前后各 {log_context_lines} 行上下文
5. 合并重叠区间
6. 若匹配 > {max_matches_per_file}，执行聚合摘要，保留最重要 100 个匹配点

### 目录（log_source = "path" 且为目录）
1. 使用 Glob 发现 `.log`、`.txt` 文件（smart-sync 还包括 `.log.1` ~ `.log.10`）
2. 按修改时间降序，只分析最近 {max_log_files} 个
3. 对每个文件执行 Grep 关键词检索
4. 按级别和时间过滤
5. 提取上下文，合并重叠区间
6. 若总计匹配 > {max_total_matches}，执行聚合摘要
7. 提取错误堆栈（连续 "at " 或 "Traceback" 行）

### 项目特定增强
若 project_type ≠ "generic"，参考当前已加载的 reference 文件内容，执行其中"诊断增强规则"章节规定的步骤。

### 结果输出
1. 输出文件路径规则：
   - log_source = "path"：在 log_path 下创建 `auto-debug/` 子目录，写入 `auto-debug/auto_debug_extracted_{YYYYMMDD_HHMMSS_ffffff}.log`
   - log_source = "inline"：获取系统临时目录（Windows: %TEMP%，Linux/macOS: $TMPDIR 或 /tmp），在其下创建 `auto-debug/`，写入 `auto-debug/auto_debug_extracted_{YYYYMMDD_HHMMSS_ffffff}.log`
2. 返回结构化 JSON（< 100KB）：
```json
{
  "extracted_file_path": "完整路径",
  "total_matches": 数字,
  "log_entries": [{"timestamp": "...", "level": "...", "file": "...", "line": 行号, "message": "...", "context": "..."}],
  "commit_info": {"found": true/false, "hash": "...", "branch": "...", "note": "..."},
  "aggregated": true/false,
  "warnings": ["..."]
}
```

## 约束
- Grep-first，绝不全文读取 > 100MB 文件
- 单次 Grep 超时 {grep_timeout} 秒，超则跳过
- 文件读取失败则跳过并记录警告
- 返回数据 < 100KB，必要时摘要
```

---

## 6. 阶段 4：代码分析与根因定位（子 Agent）

**任务创建**：`TaskCreate -subject "代码分析与根因定位" -status "in_progress"`

**调用子 Agent**：
```
Agent(description="代码分析与根因定位", prompt="...", timeout=300000)
```

### 子 Agent Prompt 模板

```
# 代码分析任务

## 输入
- Bug 描述：{bug_description}
- 提取的日志信息：{sub_agent_3_result}
- 项目类型：{project_type}
- 配置：文件读取超时 {file_read_timeout}，上下文行数 {code_context_lines}

## 任务要求

### 步骤 1：定位出错位置
从日志中提取文件名和行号：
- `filename[line:XX]`（Python）或 `filename:XX`（C++）
- 或通过堆栈推断
多个错误按 ERROR > WARNING 排序。

### 步骤 2：读取出错代码
对每个候选位置，使用 Read 工具读取前后各 {code_context_lines} 行。超时则跳过。

### 步骤 3：分析调用链
向上追溯调用者，向下查看被调用者，识别关键数据（参数、返回值、全局变量）。

### 步骤 4：分析数据流
追踪关键变量的来源与去向，检查 null/None、空列表、边界值等。

### 步骤 5：识别根因模式
与日志错误消息比对，确定类别：

| 类别 | 典型模式 |
|------|----------|
| 逻辑缺陷 | 条件判断错误、循环边界、状态机遗漏 |
| 遗漏场景 | 未处理边界、空值、异常未捕获 |
| 并发问题 | 竞态条件、死锁、资源未释放 |
| 数据问题 | 格式不匹配、编码、序列化错误 |
| 配置错误 | 环境变量缺失、配置冲突 |
| 集成问题 | API 调用错误、参数不匹配 |
| 性能问题 | 内存泄漏、无限循环、资源耗尽 |

### 步骤 6：验证与不确定处理
- 在代码中找到支持/反驳证据
- 多个可能根因则全部列出并标注置信度
- 无法确定唯一根因、日志与代码严重不符、需要补充信息时 → 暂停，向用户说明瓶颈

## 输出格式（返回主 Agent）
```json
{
  "analysis_summary": "分析概要",
  "root_cause": {
    "category": "类别",
    "description": "根因描述",
    "confidence": 85,
    "evidence": [{"file": "...", "line": 行号, "code_snippet": "...", "explanation": "..."}]
  },
  "alternative_causes": [{"category": "...", "description": "...", "confidence": 60, "evidence": [...]}],
  "suggested_fix": {
    "description": "修复思路",
    "approach": "修改方式",
    "affected_files": [{"path": "...", "change_type": "edit/insert/delete", "line_range": "...", "current_content": "...", "proposed_change": "..."}]
  },
  "uncertainties": ["..."],
  "needs_user_input": ["..."]
}
```

## 约束
- 仅读取代码，不修改任何文件
- > 1MB 文件只读取出错位置附近 ±200 行
- Read 超时则跳过并记录警告
- 返回数据 < 100KB
```

---

## 7. 阶段 5：诊断报告生成与修复方案

**任务创建**：`TaskCreate -subject "诊断报告生成与修复方案" -status "in_progress"`

**处理逻辑**：
1. 收集阶段 3 和 4 的结果
2. 计算置信度评分：
   ```
   score = 日志证据充分度 × 40% + 代码分析确定度 × 35% + 复现可能性 × 25%
   ```
3. 生成 markdown 报告，同时展示并保存到文件

### 输出文件路径规则
- `log_source = "path"`：保存到 `{log_path}/auto-debug/auto_debug_report_{YYYYMMDD_HHMMSS}.md`
- `log_source = "inline"`：保存到系统临时目录下的 `auto-debug/auto_debug_report_{YYYYMMDD_HHMMSS}.md`
- 如果目录不可写：仅终端展示并提示

### 报告核心章节
1. 基本信息（时间、项目、分支、Bug 描述）
2. 问题摘要
3. 根因分析（出错位置、代码片段、触发条件、数据流、分类）
4. 影响范围（直接 + 潜在）
5. 复现步骤
6. 修复方案（描述 + diff + 修改文件列表）
7. 置信度评分（X/100，附三维度评分依据）
8. 项目特定信息（若 project_type ≠ generic，包含 commit 比对结果等）

### 用户确认
展示报告后**必须等待用户决策**：
1. **确认修复** → 进入阶段 6
2. **修改方案** → 用户提意见，重新生成修复方案
3. **仅采纳分析** → 认可诊断但暂不修复
4. **否决** → 流程终止

---

## 8. 阶段 6：代码修复执行

**任务创建**：`TaskCreate -subject "代码修复执行" -status "in_progress"`

**前置条件**：阶段 5 中用户选择"确认修复"。

**执行规则**：
1. 严格按照阶段 4 的 `suggested_fix.affected_files` 依次执行
2. 每处修改使用 Edit 工具，指定 `file_path`、`old_string`、`new_string`
3. 若 `old_string` 无法匹配 → 暂停，提示 E0601
4. 若 Edit 报错 → 暂停，提示 E0602

---

## 9. 阶段 7：修复验证

**任务创建**：`TaskCreate -subject "修复验证" -status "in_progress"`

**验证方式**（按项目实际情况选择）：

| 验证项 | 执行方式 |
|--------|----------|
| 语法检查 | Python: `python -m py_compile`；Node: ESLint |
| 单元测试 | `pytest`、`npm test`、`make test` |
| 静态分析 | `flake8`、`pylint`、`eslint` |

**自动发现测试命令**：
- 存在 `pytest.ini` / `tox.ini` / `setup.py` → `pytest`
- 存在 `package.json` 的 `scripts.test` → `npm test`
- 存在 `Makefile` 的 `test` → `make test`
- 无测试配置 → 降级为语法检查

**结果处理**：
- 全部通过 → 任务完成
- 存在失败 → 展示详情，提示用户选择：回滚 / 继续修复 / 终止

---

## 10. 错误码与中断处理

| 阶段 | 错误码 | 触发条件 | 恢复动作 |
|------|--------|----------|----------|
| 1 | E0101-E0103 | 输入校验失败 | 暂停，提示用户补充 |
| 2 | E0201-E0204 | 环境检查失败 | 暂停，提示环境问题 |
| 3 | E0301 | 子 Agent 超时 | 评估部分结果或询问用户 |
| 3 | E0303 | 匹配行数超限 | 触发聚合摘要 |
| 4 | E0401 | 子 Agent 超时 | 评估部分结果或询问用户 |
| 4 | E0402 | 无法定位代码 | 请求用户补充信息 |
| 5 | E0501 | 报告写入失败 | 重试一次，仍失败则仅终端展示 |
| 6 | E0601-E0602 | 修复执行失败 | 暂停，提示用户 |
| 7 | E0701 | 验证失败 | 等待用户决策 |

---

## 11. 进度追踪

每个阶段使用 TaskCreate/TaskUpdate 跟踪：
```
TaskCreate -subject "<阶段名>" -status "pending"
→ TaskUpdate -status "in_progress"
→ TaskUpdate -status "completed"
```

---

## 12. 调用示例

```
> /auto-debug "同步重命名后旧文件残留" --log "D:\bug_log\smart_sync\Fangcloud\SmartSync"

[系统] 开始诊断...

[阶段1] 输入校验完成
[阶段2] 环境检查通过，识别为 smart-sync 项目，已加载 references/smart-sync.md
[阶段3] 日志提取中...（子 Agent 执行）
[阶段4] 代码分析中...（子 Agent 执行）
[阶段5] 报告已生成，保存至 D:\bug_log\smart_sync\Fangcloud\SmartSync\auto-debug\auto_debug_report_20260512_143022.md

# Bug 诊断报告
...

是否确认修复？ (确认修复/修改方案/仅采纳分析/否决)
```

---

## 13. 扩展新项目适配

新增项目支持时，**无需修改 SKILL.md**，只需：

1. 在 `config/project-adapters-config.json` 的 `adapters` 数组中新增条目：
   ```json
   {
     "id": "new-project",
     "name": "new-project",
     "match": {
       "directories": ["src"],
       "files": ["package.json", "src/main.js"]
     },
     "reference": "references/new-project.md",
     "priority": 10
   }
   ```
2. 在 `references/` 下新增 `new-project.md`，写入项目特定知识（日志格式、commit 提取规则、分支体系、诊断增强规则）

---
