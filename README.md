# auto-debug

基于用户提供的 Bug 描述与日志，自主分析定位问题根因，输出诊断报告与修复方案，经用户确认后执行代码修复的自动化 Skill。

**目标平台**：Claude Code、Codex 及其他支持 Skill 机制的 Agent/插件平台。

**核心特性**：
- 渐进式披露 — 项目特定知识按需加载，减少无关上下文
- 插件化适配 — 新增项目支持只需修改配置 + 新增 reference 文件，无需改动 SKILL.md
- 零项目污染 — 不在项目目录创建任何文件，所有输出集中在日志路径的 `auto-debug/` 子目录或系统临时目录

---

## 支持的项目

| 项目类型 | 识别特征 | 特有诊断能力 |
|----------|----------|-------------|
| **smart-sync** | `sync/` + `build.py` + `requirements.txt` + `sync/smart_sync.py` | commit hash 比对、分支标注、日志优先级排序 |
| **sync-core-service** | `CMakeLists.txt` + `package.json`(node_addon) + `src/node_addon/` | commit_id/version 比对、DEBUG 日志检查 |
| **通用项目** | 其他 | 基本日志分析和根因定位 |

---

## 安装

### 方式一：npx skills add（推荐）

本 Skill 支持通过 Vercel 的 `skills` CLI 工具一键安装：

```bash
# 安装到 Claude Code（指定子目录，只安装 skill 内容）
npx skills add XiaoFeng-lin/auto-debug --skill auto-debug

# 或使用完整 GitHub URL
npx skills add https://github.com/XiaoFeng-lin/auto-debug --skill auto-debug
```

**为什么需要 `--skill auto-debug`**：
本仓库根目录包含开发文档（docs/、tests/、CLAUDE.md 等），Skill 的实际内容位于 `auto-debug/` 子目录中。使用 `--skill auto-debug` 可确保只安装必要的 skill 文件，避免把开发文档带入你的 skill 目录。

### 方式二：手动复制

```bash
# 克隆仓库
git clone https://github.com/XiaoFeng-lin/auto-debug.git

# 手动复制 skill 内容到 Claude Code Skill 目录
mkdir -p ~/.claude/skills/auto-debug
cp auto-debug/SKILL.md ~/.claude/skills/auto-debug/
cp -r auto-debug/config/ ~/.claude/skills/auto-debug/
cp -r auto-debug/references/ ~/.claude/skills/auto-debug/
```

---

## 更新

```bash
# 更新到最新版本
npx skills update auto-debug

# 或跳过确认，直接更新
npx skills update auto-debug -y
```

---

## 使用

```
/auto-debug "<Bug描述>" [--log <内联日志或路径>] [--time <时间范围>] [--keyword <关键词>]
```

### 示例

```bash
# 日志目录路径
/auto-debug "同步重命名后旧文件残留" --log "D:\bug_log\smart_sync\Fangcloud\SmartSync"

# 单文件路径
/auto-debug "下载文件时进度卡在 99%" --log "D:\bug_log\sync_core_service\FangCloud\Drive\node_addon.log"

# 内联日志
/auto-debug "同步重命名后旧文件残留" --log "[2026-04-29 10:23] ERROR sync_handler.py:142 - Rename operation failed"

# 带时间范围和关键词
/auto-debug "传输失败后重试无效" --log "D:\bug_log\smart_sync\Fangcloud\SmartSync" --time "2026-04-29 10:00~10:30" --keyword "retry,transfer,timeout"
```

---

## 项目结构

```
auto-debug/                          # GitHub 仓库根目录
├── README.md                        # 本文件
├── CHANGELOG.md                     # 版本变更记录
├── LICENSE
├── .gitignore
│
├── auto-debug/                      # ← Skill 实际内容（安装此目录）
│   ├── SKILL.md                     # 核心 Skill 文件（通用流程，< 500 行）
│   ├── config/
│   │   ├── default-config.json      # 运行时参数默认值
│   │   └── project-adapters-config.json # 项目适配器注册表
│   ├── references/
│   │   ├── smart-sync.md            # smart-sync 特定知识（按需加载）
│   │   └── sync-core-service.md     # sync-core-service 特定知识（按需加载）
│   └── scripts/                     # （后续迭代）辅助脚本
│
├── docs/                            # 开发文档（不进入 skill 安装）
│   ├── spec/
│   │   ├── PRD.md
│   │   └── PRD-review-report-v1.md
│   ├── plan/
│   │   └── IMPLEMENTATION_PLAN.md
│   ├── USER_GUIDE.md
│   ├── RUNBOOK.md
│   └── dev-log/
├── tests/                           # 测试（不进入 skill 安装）
├── CLAUDE.md                        # Claude Code 开发指引
└── AGENTS.md                        # 通用 Agent 开发指引
```

**`npx skills add --skill auto-debug` 只安装 `auto-debug/` 子目录的内容**，docs/、tests/、CLAUDE.md、AGENTS.md 等开发文档保留在仓库中，不会进入用户的 skill 目录。

---

## 扩展新项目适配

新增项目支持时，**无需修改 SKILL.md**，只需两步：

### 1. 注册适配器

在 `auto-debug/config/project-adapters-config.json` 中新增条目：

```json
{
  "id": "my-project",
  "name": "my-project",
  "match": {
    "directories": ["src"],
    "files": ["package.json", "src/main.js"]
  },
  "reference": "references/my-project.md",
  "priority": 10
}
```

### 2. 编写项目知识文件

在 `auto-debug/references/` 下新增 `my-project.md`，写入：
- 项目识别说明
- 日志格式与文件清单
- Commit/版本提取与比对规则
- 分支体系
- 诊断增强规则
- 核心模块参考

完成后 push 到 GitHub，用户即可通过 `npx skills add` 获取更新后的版本。

---

## 输出文件位置

| 场景 | 提取日志 | 诊断报告 |
|------|----------|----------|
| 提供日志路径 | `{log_path}/auto-debug/auto_debug_extracted_*.log` | `{log_path}/auto-debug/auto_debug_report_*.md` |
| 内联日志 | `%TEMP%/auto-debug/auto_debug_extracted_*.log` | `%TEMP%/auto-debug/auto_debug_report_*.md` |

**注意**：所有输出都集中在 `auto-debug/` 子目录中，不会和原始日志混放，也不会污染项目目录。

---

## 文档索引

- **产品需求**：[`docs/spec/PRD.md`](docs/spec/PRD.md)
- **PRD 评审报告**：[`docs/spec/PRD-review-report-v1.md`](docs/spec/PRD-review-report-v1.md)
- **实施计划**：[`docs/plan/IMPLEMENTATION_PLAN.md`](docs/plan/IMPLEMENTATION_PLAN.md)
- **用户指南**：[`docs/USER_GUIDE.md`](docs/USER_GUIDE.md)
- **运维手册**：[`docs/RUNBOOK.md`](docs/RUNBOOK.md)

---

## 状态

MVP 重构完成 — Skill 已拆分为通用核心 + 按需加载的 reference 文件，支持插件化项目适配和 `npx skills add` 安装。
