# AGENTS.md

本项目为 **auto-debug Skill** 的开发仓库。以下信息帮助 Agent 工具（Kimi Code CLI 等）理解和处理本项目。

## 项目概述

**auto-debug** 是一个自动化 Bug 诊断与修复的 Skill。根据用户提供的 Bug 描述与日志，自主分析定位根因，输出诊断报告与修复方案，经用户确认后执行代码修复。

**目标平台**：Claude Code、Codex 及其他支持 Skill 机制的 Agent/插件平台。

**当前状态**：MVP 开发完成。

**所有回复使用中文。**

## 核心原则

- **不猜测**：不确定时必须与用户确认，严禁假设或推测
- **不擅自写入**：分析阶段仅读取代码，修复阶段需经用户确认后才写入
- **全程可追踪**：每个阶段的状态和结论都有记录
- **上下文隔离**：大量日志处理交由子 Agent 执行，避免污染主 Agent 上下文
- **项目强绑定**：必须在项目根目录下执行

## 架构

```
主 Agent（协调者）
  ├─ 子 Agent A: 日志提取与清洗（输入: 日志路径 + 关键词；输出: 结构化日志信息 + 提取文件）
  ├─ 子 Agent B: 代码分析与根因定位（输入: 日志信息 + 项目代码；输出: 根因 + 修复建议）
  └─ 主 Agent: 协调调度 + 报告生成 + 修复执行
```

### 7 阶段工作流

1. **输入校验与歧义澄清** — 解析 Bug 描述，识别日志输入方式（内联 vs 路径），有歧义则暂停询问
2. **环境检查** — 验证项目目录、代码可读性、日志路径可达性
3. **日志提取与清洗**（子 Agent） — Grep-first 策略，绝不批量读取大日志
4. **代码分析与根因定位**（子 Agent） — 定位出错代码，追溯调用链和数据流；只读
5. **诊断报告与修复方案** — 结构化 markdown 报告，含置信度评分（X/100）
6. **用户确认** — 确认修复 / 修改方案 / 仅采纳分析 / 否决
7. **代码修复与验证** — 执行确认的修复，然后验证（语法、测试、lint、回归）

## 项目特定适配

通过项目根目录特征自动识别项目类型：

### smart-sync（Python）
- **识别特征**：`sync/` 目录 + `build.py` + `requirements.txt` + `sync/smart_sync.py`
- **日志文件**：`debug.log`、`error.log`、`monitor.log`、`conflict.log`、`alive.log`
- **Commit Hash**：从 `smart_sync.py[line:66] - meta: {"commit_hash": "..."}` 提取

### sync-core-service（C++17）
- **识别特征**：`CMakeLists.txt` + `package.json`（name="node_addon"）+ `src/node_addon/node_addon.cpp`
- **日志文件**：`node_addon.log`
- **版本标识**：`commit_id: {8位短hash}` 或 `version: {手动版本号}`

### 通用项目
无 commit 比对或特殊处理，按通用流程执行。

## 安全约束

- 不修改原始日志文件
- 未经用户确认不修改任何项目代码
- 输出文件仅写入日志路径的 `auto-debug/` 子目录或系统临时目录
- 不执行有副作用的命令（数据库操作、网络请求等）

## 配置参数

**配置唯一来源**：skill 内置的 `auto-debug/config/default-config.json`。不在项目目录创建任何配置文件，不读取全局配置。

关键默认值：上下文行数 5 | 子 Agent 超时 300s | 单文件匹配上限 500 行 | 总匹配上限 1000 行。

## 项目文件结构

```
auto-debug/                          # GitHub 仓库根目录
├── README.md                        # 项目说明
├── CHANGELOG.md                     # 版本变更记录
├── CLAUDE.md                        # Claude Code 项目指引
├── AGENTS.md                        # 本文件（通用 Agent 指引）
│
├── auto-debug/                      # ← Skill 实际内容
│   ├── SKILL.md                     # 核心 Skill 文件（通用流程，< 500 行）
│   ├── config/
│   │   ├── default-config.json      # 运行时参数默认值（唯一配置来源）
│   │   └── project-adapters-config.json # 项目适配器注册表
│   └── references/
│       ├── smart-sync.md            # smart-sync 特定知识（按需加载）
│       └── sync-core-service.md     # sync-core-service 特定知识（按需加载）
│
├── docs/                            # 开发文档
│   ├── README.md                    # 文档导航索引
│   ├── spec/
│   │   ├── PRD.md                   # 产品规格文档
│   │   └── PRD-review-report-v1.md  # PRD 评审报告
│   ├── plan/
│   │   └── IMPLEMENTATION_PLAN.md   # 实施计划与验证清单
│   ├── USER_GUIDE.md                # 用户指南
│   ├── RUNBOOK.md                   # 运维手册
│   └── dev-log/                     # 开发过程记录
│
└── tests/
    └── test-cases.md                # 测试用例
```

## 编码规范

- 文件编码：UTF-8
- 缩进：2 个空格
- 换行符：LF（Unix）
- Skill 文件使用 Markdown 格式
- 所有面向用户的文档和回复使用中文
