# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

**auto-debug** 是一个自动化 Bug 诊断与修复的 Skill。根据用户提供的 Bug 描述与日志，自主分析定位根因，输出诊断报告与修复方案，经用户确认后执行代码修复。

**目标平台**：Claude Code、Codex 及其他支持 Skill 机制的 Agent/插件平台。

**当前状态**：MVP 开发完成 — Skill 文件在 `.claude/skills/auto-debug/SKILL.md`。

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
3. **日志提取与清洗**（子 Agent） — Grep-first 策略，绝不批量读取大日志；目录场景：发现 → 过滤 → 提取上下文 → 项目特定增强
4. **代码分析与根因定位**（子 Agent） — 定位出错代码，追溯调用链和数据流，识别根因模式；只读
5. **诊断报告与修复方案** — 结构化 markdown 报告，含置信度评分（X/100）；保存至用户指定日志路径
6. **用户确认** — 确认修复 / 修改方案 / 仅采纳分析 / 否决
7. **代码修复与验证** — 执行确认的修复，然后验证（语法、测试、lint、回归）

### 进度追踪

使用 TaskCreate/TaskUpdate 跟踪各阶段，每个 Task：`pending → in_progress → completed`。

## 项目特定适配

通过项目根目录特征自动识别项目类型：

### smart-sync（Python）
- **识别特征**：`sync/` 目录 + `build.py` + `requirements.txt` + `sync/smart_sync.py`
- **技术栈**：Python 3.6.8+, asyncio + aiohttp + tornado, SQLAlchemy, Sentry SDK
- **主要分支**：`production_hubble_gray`（常用）、`production_hubble_scatter`（散落同步扩展）、`production`、`private_cloud`
- **日志文件**：`debug.log`、`error.log`、`monitor.log`、`conflict.log`、`alive.log` — 总计可超 1GB
- **日志格式**：`{时间} - {进程ID} - {线程ID} - {日志级别} - {文件名}[line:{行号}] - {消息内容}`
- **Commit Hash**：从 `smart_sync.py[line:66] - meta: {"commit_hash": "..."}` 提取
- **日志优先级**：debug.log > monitor.log > error.log > conflict.log > alive.log

### sync-core-service（C++17）
- **识别特征**：`CMakeLists.txt` + `package.json`（name="node_addon"）+ `src/node_addon/node_addon.cpp`
- **技术栈**：C++17, CMake 3.12+, node-addon-api, cpprestsdk, easylogging++, SQLite
- **主要分支**：`production`（主）、`master`、`production-hotfix`
- **日志文件**：`node_addon.log`（单文件，最大 50MB × 10 个备份）
- **版本标识**：`commit_id: {8位短hash}`（新版）或 `version: {手动版本号}`（老版）；始终在日志开头
- **DEBUG 日志**：由 `logging.conf` 控制；不存在时默认关闭

### 通用项目
无 commit 比对或特殊处理，按通用流程执行。

## 日志提取策略（Grep-first）

绝不批量读取日志文件。流程：
1. 确定搜索范围（用户指定文件或默认值）
2. 从启动日志提取 commit hash / version
3. 通过 ripgrep/grep 跨文件关键词检索
4. 按时间范围和日志级别过滤（ERROR/WARNING 优先）
5. 提取匹配行 ±5 行上下文，合并重叠区间
6. 结果写入 `auto_debug_extracted_{timestamp}.log` 至用户指定路径

## 输出文件

- **提取日志**：`auto_debug_extracted_{timestamp}.log` — 保存至用户日志路径
- **诊断报告**：`auto_debug_report_{timestamp}.md` — 保存至用户日志路径
- **绝不修改原始日志文件**

## 安全约束

- 不修改原始日志文件
- 未经用户确认不修改任何项目代码
- 输出文件仅写入用户指定的日志路径
- 不执行有副作用的命令（数据库操作、网络请求等）

## 错误处理

规格文档第十二章定义了完整的错误码体系（E0101-E0702），覆盖 7 个阶段的关键失败场景。核心策略：
- 子 Agent 超时 120s → 主 Agent 评估已有结果，可用则继续，不可用则询问用户
- 匹配行数超限（单文件 500 / 总计 1000）→ 触发聚合摘要
- 修复方案与实际代码不符 → 暂停，告知用户差异，不擅自调整

## 配置参数

规格文档第十三章定义了完整的可配置参数清单。配置优先级：命令行参数 > 项目级 `.auto-debug-config.json` > 全局 `~/.auto-debug/config.json` > 内置默认值。

关键默认值：
- 上下文行数：5 | 子 Agent 超时：120s | 全流程最大耗时：5 分钟
- 单文件匹配上限：500 行 | 总匹配上限：1000 行 | 子 Agent 结果上限：100KB

## 性能基线（MVP）

- 单日志文件 ≤ 100MB | 日志文件总数 ≤ 50 | 源代码文件 ≤ 1MB
- 超出基线时降级处理并在报告中标注

## 调用方式

```
/auto-debug <Bug描述> [--log <内联日志或路径>] [--time <时间范围>] [--keyword <关键词>]
```

## 项目文件结构

```
auto-debug/
├── SKILL.md                    ← 核心 Skill 文件
├── CLAUDE.md                   ← 本文件（Claude Code 项目指引）
├── AGENTS.md                   ← 通用 Agent 项目指引
├── README.md                   ← 项目说明
├── .gitignore
├── config/
│   └── default-config.json     ← 配置文件模板
├── docs/
│   ├── PRD.md                  ← 产品规格文档
│   ├── USER_GUIDE.md           ← 用户指南
│   ├── RUNBOOK.md              ← 运维手册
│   ├── ARCHITECTURE.md         ← 技术设计与实施计划
│   └── archive/
│       └── review-report-v1.md ← 过程评审报告
└── tests/
    └── test-cases.md           ← 测试用例
```

## 待确认事项

见规格文档第十章：结构化日志支持、多语言项目支持、子 Agent 脚本实现方式。
