# auto-debug 用户指南

## 快速开始

### 安装

通过 Vercel 的 `skills` CLI 工具一键安装：

```bash
npx skills add https://github.com/XiaoFeng-lin/auto-debug --skill auto-debug
```

**为什么需要 `--skill auto-debug`**：
本仓库根目录包含开发文档（docs/、tests/ 等），Skill 的实际内容位于 `auto-debug/` 子目录中。使用 `--skill auto-debug` 可确保只安装必要的 skill 文件。

### 更新

```bash
npx skills update auto-debug
# 或跳过确认
npx skills update auto-debug -y
```

### 调用方式

```
/auto-debug "<Bug描述>" [--log <内联日志或路径>] [--time <时间范围>] [--keyword <关键词>]
```

### 参数说明

| 参数 | 必需 | 说明 | 示例 |
|------|------|------|------|
| `Bug描述` | 是 | 用双引号包裹，包含：现象、预期行为、实际行为 | `"同步重命名后旧文件残留"` |
| `--log` | 是 | 内联日志内容或日志文件/目录路径 | `--log "D:\bug_log\smart_sync"` |
| `--time` | 否 | Bug 发生的大致时间范围 | `--time "2026-04-29 10:00~10:30"` |
| `--keyword` | 否 | 额外的搜索关键词（逗号分隔） | `--keyword "rename,lock"` |

### 使用示例

**示例 1：内联日志**
```
/auto-debug "同步重命名后旧文件残留" --log "[2026-04-29 10:23] ERROR sync_handler.py:142 - Rename operation failed: target already exists"
```

**示例 2：日志文件路径**
```
/auto-debug "同步重命名后旧文件残留" --log "D:\bug_log\smart_sync\Fangcloud\SmartSync"
```

**示例 3：带时间范围**
```
/auto-debug "同步重命名后旧文件残留" --log "D:\bug_log\smart_sync\Fangcloud\SmartSync" --time "2026-04-29 10:00~10:30"
```

**示例 4：带关键词**
```
/auto-debug "传输失败后重试无效" --log "D:\bug_log\sync_core_service\FangCloud\Drive" --keyword "retry,transfer,timeout"
```

## 支持的项目类型

auto-debug 会自动识别以下项目类型并启用特定诊断逻辑：

| 项目 | 识别特征 | 特有功能 |
|------|----------|----------|
| **smart-sync** | `sync/` + `build.py` + `requirements.txt` + `sync/smart_sync.py` | commit hash 比对、分支标注、日志优先级排序 |
| **sync-core-service** | `CMakeLists.txt` + `package.json`(node_addon) + `src/node_addon/` | commit_id/version 比对、DEBUG 日志检查 |
| **通用项目** | 其他 | 基本日志分析和根因定位 |

## 工作流程

```
输入校验 → 环境检查 → 日志提取 → 代码分析 → 报告生成 → 用户确认 → 代码修复 → 修复验证
```

1. **输入校验**：解析你的 Bug 描述和日志，如有歧义会暂停询问
2. **环境检查**：确认当前目录是有效的项目目录，日志路径可达
3. **日志提取**：从日志中提取与 Bug 相关的关键信息（使用 Grep-first 策略，不全文读取大文件）
4. **代码分析**：结合日志和源代码定位根因
5. **报告生成**：输出标准化诊断报告，包含根因分析、修复方案、置信度评分
6. **用户确认**：你必须明确确认后才会执行代码修复
7. **代码修复**：按确认的方案修改代码
8. **修复验证**：运行测试或静态检查验证修复效果

## 诊断报告结构

| 章节 | 内容 |
|------|------|
| 基本信息 | 时间、项目、分支、Bug 描述 |
| 问题摘要 | 一句话总结根因 |
| 根因分析 | 出错代码位置、触发条件、数据流/控制流分析、分类 |
| 影响范围 | 直接影响和潜在影响 |
| 复现步骤 | 前置条件、操作步骤、预期 vs 实际 |
| 修复方案 | 修复思路 + diff 代码变更 + 修改文件列表 |
| 置信度评分 | X/100，附带三个维度的评分依据 |
| 项目特定信息 | commit 比对结果、分支信息等（如适用） |

## 用户确认选项

| 选项 | 说明 |
|------|------|
| 确认修复 | 批准方案，执行代码修改 |
| 修改方案 | 提出调整意见，重新生成修复方案 |
| 仅采纳分析 | 认可诊断但暂不修复 |
| 否决 | 不认可分析结果，流程终止 |

## 配置

auto-debug 的所有配置参数存储在 skill 安装目录的 `config/default-config.json` 中，由 Skill 自动读取，**无需在项目目录下创建任何配置文件**。

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `log_context_lines` | 5 | 匹配行前后各提取的行数 |
| `sub_agent_timeout` | 300 | 子 Agent 最大执行时间（秒） |
| `max_matches_per_file` | 500 | 单文件匹配行数上限 |
| `max_total_matches` | 1000 | 总匹配行数上限 |
| `log_level_filter` | ERROR,WARNING | 优先检索的日志级别 |
| `max_log_files` | 50 | 目录下最多分析的日志文件数 |
| `enable_commit_comparison` | true | 是否启用 commit 比对 |

## 输出文件位置

| 场景 | 提取日志 | 诊断报告 |
|------|----------|----------|
| 提供日志路径 | `{log_path}/auto-debug/auto_debug_extracted_*.log` | `{log_path}/auto-debug/auto_debug_report_*.md` |
| 内联日志 | `%TEMP%/auto-debug/auto_debug_extracted_*.log` | `%TEMP%/auto-debug/auto_debug_report_*.md` |

## 注意事项

- 必须在**项目根目录**下执行 auto-debug
- 修复前必须用户确认，不会擅自修改代码
- 日志提取使用搜索策略，不会全文读取大文件
- 原始日志文件不会被修改
- **所有输出文件集中在 `auto-debug/` 子目录中**，不会污染项目目录或日志根目录
- 不在项目目录下创建任何非项目文件（包括配置文件）
