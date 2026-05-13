# auto-debug 项目实施计划

> 计划版本：1.1
> 最后更新：2026-05-12
> 所有操作在项目根目录 `D:/code_test/auto-debug` 下执行

## 1. 项目目标

实现一个名为 `auto-debug` 的 Claude Code Skill，能够：

- 根据用户提供的 Bug 描述与日志，自动分析定位根因
- 生成标准化诊断报告与修复方案
- 经用户确认后执行代码修复
- 验证修复效果

**核心设计目标（v1.1 新增）**：
- **渐进式披露**：项目特定知识按需加载，减少无关上下文
- **插件化适配**：新增项目支持无需改动 SKILL.md
- **零项目污染**：不在项目目录创建任何非项目文件

**最终交付物**：

- `auto-debug/SKILL.md` — 精简通用核心（< 500 行）
- `auto-debug/config/default-config.json` — 运行时参数默认值
- `auto-debug/config/project-adapters-config.json` — 项目适配器注册表
- `auto-debug/references/*.md` — 项目特定知识文件（按需加载）
- `auto-debug/scripts/` — 可选的 Python 辅助脚本（后续迭代）
- `tests/` — 测试用例
- `docs/USER_GUIDE.md` — 用户指南
- `docs/RUNBOOK.md` — 运维手册

## 2. 核心架构

```
主 Agent (Coordinator)
  ├─ 子 Agent A: 日志提取与清洗 (context: fork, agent: Explore)
  ├─ 子 Agent B: 代码分析与根因定位 (context: fork, agent: Explore)
  └─ 主 Agent: 协调调度 + 报告生成 + 修复执行
```

### 渐进式披露架构（v1.1）

```
SKILL.md（通用核心，始终加载）
  ├─ 输入解析、公共错误码、通用约束
  ├─ 配置加载 → config/default-config.json
  ├─ 项目识别 → config/project-adapters-config.json
  │    └─ 匹配成功 → Read(references/{project-type}.md)
  │         └─ 项目特定知识按需进入上下文
  └─ 7 阶段流程框架
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

### Phase 1：Skill 核心重构（已完成）

| 步骤 | 任务 | 输出 |
|------|------|------|
| 1.1 | 拆分 SKILL.md | 精简通用核心（435 行）+ references/*.md |
| 1.2 | 创建适配器注册表 | `config/project-adapters-config.json` |
| 1.3 | 调整配置加载逻辑 | 取消项目级/全局配置，改为 skill 内置 |
| 1.4 | 调整输出路径 | `{log_path}/auto-debug/` 或 `%TEMP%/auto-debug/` |
| 1.5 | 支持 npx skills add 安装 | Skill 内容移至 `auto-debug/` 子目录 |

### Phase 2：配套文件更新（目标：1天）

| 步骤 | 任务 | 输出 |
|------|------|------|
| 2.1 | 更新用户指南 | `docs/USER_GUIDE.md` |
| 2.2 | 更新运维手册 | `docs/RUNBOOK.md` |
| 2.3 | 更新实施计划文档 | `docs/plan/IMPLEMENTATION_PLAN.md` |
| 2.4 | 更新产品规格文档 | `docs/spec/PRD.md` |
| 2.5 | 更新 README.md | 新的安装说明和项目结构 |
| 2.6 | 创建 CHANGELOG.md | 版本变更记录 |

### Phase 3：测试与验证（目标：1天）

| 步骤 | 任务 | 输出 |
|------|------|------|
| 3.1 | 验证 npx skills add 安装 | 确认 `--skill auto-debug` 安装后目录结构正确 |
| 3.2 | 验证渐进式加载 | smart-sync 项目只加载 smart-sync.md，通用项目不加载 |
| 3.3 | 验证配置来源 | 确认只读取 skill 目录的 default-config.json |
| 3.4 | 验证输出路径 | 确认报告写入 auto-debug/ 子目录 |
| 3.5 | 验证扩展性 | 在 project-adapters-config.json 注册一个测试 adapter，确认无需改 SKILL.md |

### Phase 4：部署与发布（目标：0.5天）

| 步骤 | 任务 | 输出 |
|------|------|------|
| 4.1 | 提交所有更改到 Git | `git add` + `commit` + `push` |
| 4.2 | 测试 npx skills add 安装 | `npx skills add https://github.com/XiaoFeng-lin/auto-debug --skill auto-debug` |
| 4.3 | 测试 npx skills update | `npx skills update auto-debug -y` |
| 4.4 | 验证本地 skill 调用 | `/auto-debug` 是否可用 |

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
| 子 Agent 上下文限制 | 子 Agent 最大输出约 100KB |
| skills CLI 版本差异 | 不同版本的 `npx skills` 行为可能有差异 |

## 5. 风险与缓解措施

| 风险 | 缓解措施 |
|------|----------|
| 日志文件过大导致上下文溢出 | 在子 Agent 中实现聚合摘要，只返回结构化关键信息 |
| 代码分析无法定位根因 | 提供多个候选根因，标注置信度，请求用户补充信息 |
| 用户不确认就执行修复 | 严格遵循"不擅自写入"原则，必须用户确认后才执行 Edit |
| 配置文件来源单一 | 所有配置集中在 skill 安装目录，避免与用户项目冲突 |
| skills CLI 只复制 SKILL.md | Skill 核心内容放在 `auto-debug/` 子目录，`--skill` 参数确保完整安装 |

## 6. 成功指标

| 指标 | 目标值 |
|------|--------|
| 诊断准确率 | ≥ 80%（根因在候选列表前 2 名中） |
| 平均诊断耗时 | ≤ 30 分钟（从启动到报告生成） |
| 用户确认率 | ≥ 70%（用户选择"确认修复"的比例） |
| 修复验证通过率 | ≥ 85%（验证阶段通过的比例） |
| Skill 加载成功率 | ≥ 95%（npx skills add 安装后可用） |

## 7. 开发工具与规范

- **开发语言**：Markdown（Skill） + Bash（命令）
- **文件编码**：UTF-8
- **缩进**：2 个空格
- **换行符**：LF（Unix）
- **Git 策略**：所有更改提交到 `main` 分支

## 8. 验证清单（完成时检查）

- [ ] `auto-debug/SKILL.md` 存在且格式正确，< 500 行
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
- [ ] 配置来源为 skill 内置 `config/default-config.json`
- [ ] 新增项目适配无需修改 SKILL.md（验证 project-adapters-config.json）
- [ ] 输出文件写入 `{log_path}/auto-debug/` 或 `%TEMP%/auto-debug/`
- [ ] 错误处理覆盖所有 E0101~E0702 错误码
- [ ] 无未授权文件写入
- [ ] 用户指南和运维手册已更新至 v1.1
- [ ] `npx skills add https://github.com/XiaoFeng-lin/auto-debug --skill auto-debug` 安装成功
- [ ] `npx skills update auto-debug -y` 更新成功

---

> **注**：本计划已随 v1.1 重构更新。渐进式披露和插件化适配是本次重构的核心架构变更。
