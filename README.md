# auto-debug

基于用户提供的 Bug 描述与日志，自主分析定位问题根因，输出诊断报告与修复方案，经用户确认后执行代码修复的自动化 Skill。

## 支持的项目

| 项目类型 | 识别特征 | 特有诊断能力 |
|----------|----------|-------------|
| **smart-sync** | `sync/` + `build.py` + `requirements.txt` + `sync/smart_sync.py` | commit hash 比对、分支标注、日志优先级排序 |
| **sync-core-service** | `CMakeLists.txt` + `package.json`(node_addon) + `src/node_addon/` | commit_id/version 比对、DEBUG 日志检查 |
| **通用项目** | 其他 | 基本日志分析和根因定位 |

## 安装

将本目录复制到 Claude Code 的用户级 Skill 目录：

```bash
# Claude Code
xcopy /E /I auto-debug %USERPROFILE%\.claude\skills\auto-debug

# 或手动复制 auto-debug/SKILL.md 到 ~/.claude/skills/auto-debug/SKILL.md
```

## 使用

```
/auto-debug "<Bug描述>" [--log <内联日志或路径>] [--time <时间范围>] [--keyword <关键词>]
```

## 项目结构

```
auto-debug/
├── SKILL.md              ← Skill 本体（部署时只需此文件）
├── CLAUDE.md             ← Claude Code 项目指引
├── AGENTS.md             ← 通用 Agent 项目指引
├── README.md             ← 本文件
├── config/
│   └── default-config.json
├── docs/
│   ├── PRD.md            ← 产品需求规格
│   ├── USER_GUIDE.md     ← 用户指南
│   ├── RUNBOOK.md        ← 运维手册
│   ├── ARCHITECTURE.md   ← 技术设计文档
│   └── archive/          ← 归档文档
└── tests/
    └── test-cases.md
```

## 文档索引

- **产品需求**：[`docs/PRD.md`](docs/PRD.md)
- **用户指南**：[`docs/USER_GUIDE.md`](docs/USER_GUIDE.md)
- **运维手册**：[`docs/RUNBOOK.md`](docs/RUNBOOK.md)
- **技术设计**：[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md)

## 状态

MVP 开发完成。详见 [`docs/PRD.md`](docs/PRD.md) 中的待确认事项。
