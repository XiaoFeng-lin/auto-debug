# auto-debug 项目实施计划

> 计划版本：1.0
> 最后更新：2026-04-30
> 所有操作在项目根目录 `D:/code_test/auto-debug` 下执行

## 1. 项目目标

实现一个名为 `auto-debug` 的 Claude Code Skill，能够：

- 根据用户提供的 Bug 描述与日志，自动分析定位根因
- 生成标准化诊断报告与修复方案
- 经用户确认后执行代码修复
- 验证修复效果

**最终交付物**：

- `auto-debug/SKILL.md` — 可被 Claude Code 加载的 Skill 文件
- `.auto-debug-config.json` — 配置文件模板
- `scripts/` — 可选的 Python 辅助脚本（MVP 阶段暂不实现）
- `tests/` — 测试用例
- `USER_GUIDE.md` — 用户指南
- `RUNBOOK.md` — 运维手册

## 2. 核心架构

```
主 Agent (Coordinator)
  ├─ 子 Agent A: 日志提取与清洗 (context: fork, agent: Explore)
  ├─ 子 Agent B: 代码分析与根因定位 (context: fork, agent: Explore)
  └─ 主 Agent: 协调调度 + 报告生成 + 修复执行
```

### 技能调用方式

```
/auto-debug <Bug描述> [--log <内联日志或路径>] [--time <时间范围>] [--keyword <关键词>]
```

### 技能规范

| 字段 | 值 |
|------|-----|
| `name` | `auto-debug` |
| `description` | 基于用户提供的 Bug 描述与日志，自主分析定位问题根因，输出诊断报告与修复方案，经用户确认后执行代码修复的自动化工具 |
| `when_to_use` | 当用户描述一个软件 Bug 并提供日志时 |
| `disable-model-invocation` | false |
| `user-invocable` | true |
| `allowed-tools` | Read, Grep, Glob, Bash, Agent, Edit, TaskCreate, TaskUpdate |
| `context` | none (主 Agent 运行) |
| `paths` | 无（任何项目目录都可运行） |
| `arguments` | bug_description log_path time_range keyword |

## 3. 实施步骤

### Phase 1：Skill 文件开发（目标：3天）

| 步骤 | 任务 | 输出 |
|------|------|------|
| 1.1 | 创建 Skill 目录结构 | `auto-debug/SKILL.md` |
| 1.2 | 编写 Skill Frontmatter | 完整的 YAML 前置信息 |
| 1.3 | 实现阶段1：输入校验 | 使用 Bash 和 Grep 解析用户输入 |
| 1.4 | 实现阶段2：环境检查 | 检测项目根目录、代码可读性、日志路径 |
| 1.5 | 实现阶段3：日志提取（子 Agent） | 使用 `Agent` 工具调用子 Agent 执行日志提取 |
| 1.6 | 实现阶段4：代码分析（子 Agent） | 使用 `Agent` 工具调用子 Agent 分析代码 |
| 1.7 | 实现阶段5：报告生成 | 生成 markdown 格式诊断报告 |
| 1.8 | 实现阶段6：修复执行 | 使用 `Edit` 工具修改代码 |
| 1.9 | 实现阶段7：修复验证 | 运行测试/静态检查 |
| 1.10 | 实现进度追踪 | 使用 `TaskCreate`/`TaskUpdate` 跟踪每个阶段 |
| 1.11 | 实现配置化 | 读取 `.auto-debug-config.json` 并应用默认值 |
| 1.12 | 实现错误处理 | 根据错误码体系返回清晰提示 |

### Phase 2：配套文件开发（目标：2天）

| 步骤 | 任务 | 输出 |
|------|------|------|
| 2.1 | 创建配置文件模板 | `.auto-debug-config.json` |
| 2.2 | 编写用户指南 | `USER_GUIDE.md` |
| 2.3 | 编写运维手册 | `RUNBOOK.md` |
| 2.4 | 创建测试目录 | `tests/` |
| 2.5 | 编写测试用例 | 单元测试、端到端测试 |
| 2.6 | 创建 Git 忽略文件 | `.gitignore` |

### Phase 3：测试与验证（目标：2天）

| 步骤 | 任务 | 输出 |
|------|------|------|
| 3.1 | 搭建测试环境 | 创建 smart-sync 与 sync-core-service 模拟项目 |
| 3.2 | 运行端到端测试 | 验证 7 个阶段全流程 |
| 3.3 | 边界测试 | 测试大日志、空日志、错误路径等 |
| 3.4 | 性能测试 | 测量平均诊断耗时 |
| 3.5 | 人工评审 | 邀请团队成员试用并反馈 |

### Phase 4：部署与发布（目标：1天）

| 步骤 | 任务 | 输出 |
|------|------|------|
| 4.1 | 安装到个人技能目录 | 复制 `auto-debug/` 到 `~/.claude/skills/` |
| 4.2 | 测试本地调用 | `/auto-debug` 是否可用 |
| 4.3 | 创建 Git 提交 | 提交所有文件到 `main` 分支 |
| 4.4 | 创建 PR | 将功能分支合并到 `main` |

## 4. 依赖项与限制

### 依赖项

| 依赖 | 用途 | 说明 |
|------|------|------|
| ripgrep (rg) | 日志关键词搜索 | 必须安装在系统 PATH 中 |
| git | 项目识别与 commit 比对 | 必须在项目目录下可用 |
| Python | smart-sync 项目识别 | 仅用于识别特征，不执行代码 |
| Node.js | sync-core-service 项目识别 | 仅用于识别特征，不执行代码 |

### 限制

| 限制 | 说明 |
|------|------|
| 不支持多用户并发 | 每次只能一个用户使用 |
| 不支持跨平台文件路径 | 假设路径为 Windows 格式（`D:\...`） |
| 无法读取加密文件 | 无法读取权限不足或加密的文件 |
| 子 Agent 上下文限制 | 子 Agent 最大输出约 100KB |

## 5. 风险与缓解措施

| 风险 | 缓解措施 |
|------|----------|
| 日志文件过大导致上下文溢出 | 在子 Agent 中实现聚合摘要，只返回结构化关键信息 |
| 代码分析无法定位根因 | 提供多个候选根因，标注置信度，请求用户补充信息 |
| 用户不确认就执行修复 | 严格遵循“不擅自写入”原则，必须用户确认后才执行 Edit |
| 配置文件缺失导致默认值不生效 | 检查配置文件存在性，缺失时使用内置默认值，并提示用户 |
| 命令行参数解析错误 | 使用正则表达式精确匹配输入格式，无效时提示正确格式 |

## 6. 成功指标

| 指标 | 目标值 |
|------|--------|
| 诊断准确率 | ≥ 80%（根因在候选列表前 2 名中） |
| 平均诊断耗时 | ≤ 5 分钟（从启动到报告生成） |
| 用户确认率 | ≥ 70%（用户选择"确认修复"的比例） |
| 修复验证通过率 | ≥ 85%（验证阶段通过的比例） |
| 用户满意度 | ≥ 4/5 |

## 7. 开发工具与规范

- **开发语言**：Markdown（Skill） + Bash（命令）
- **文件编码**：UTF-8
- **缩进**：2 个空格
- **换行符**：LF（Unix）
- **Git 策略**：所有更改提交到 `feature/auto-debug` 分支，然后合并到 `main`

## 8. 验证清单（完成时检查）

- [ ] `auto-debug/SKILL.md` 存在且格式正确
- [ ] Frontmatter 包含 `name`, `description`, `allowed-tools`
- [ ] 支持 `/auto-debug` 调用
- [ ] 可识别 smart-sync 与 sync-core-service 项目
- [ ] 可处理内联日志与路径日志
- [ ] 日志提取使用子 Agent（`Agent` 工具）
- [ ] 代码分析使用子 Agent（`Agent` 工具）
- [ ] 报告生成包含诊断、修复方案、置信度
- [ ] 修复执行使用 `Edit` 工具，且仅在用户确认后
- [ ] 修复验证运行测试/静态检查
- [ ] 所有阶段使用 `TaskCreate`/`TaskUpdate` 跟踪
- [ ] 配置文件 `.auto-debug-config.json` 可选且默认值有效
- [ ] 错误处理覆盖所有 E0101~E0702 错误码
- [ ] 无未授权文件写入
- [ ] 用户指南和运维手册存在
- [ ] 测试用例覆盖核心场景

---

> **注**：本计划将作为 `EnterPlanMode` 的计划文件，完成后调用 `ExitPlanMode` 请求用户批准。批准后按计划步骤逐一执行。