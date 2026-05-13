# auto-debug 运维手册

## 安装与部署

### 推荐方式：npx skills add

```bash
npx skills add https://github.com/XiaoFeng-lin/auto-debug --skill auto-debug
```

### 手动安装

```bash
# 克隆仓库
git clone https://github.com/XiaoFeng-lin/auto-debug.git

# 复制 skill 内容到 Claude Code Skill 目录
mkdir -p ~/.claude/skills/auto-debug
cp auto-debug/SKILL.md ~/.claude/skills/auto-debug/
cp -r auto-debug/config/ ~/.claude/skills/auto-debug/
cp -r auto-debug/references/ ~/.claude/skills/auto-debug/
```

### 更新

```bash
npx skills update auto-debug
# 或跳过确认
npx skills update auto-debug -y
```

### 验证安装

启动 Claude Code，输入 `/auto-debug`，应能看到技能出现在自动补全列表中。

## 常见问题（FAQ）

### Q1: 调用 `/auto-debug` 后提示"当前目录不是有效的项目根目录"

**原因**：当前目录不包含 `.git`、`package.json` 等项目标识文件。

**解决**：切换到项目根目录后重新执行。

### Q2: 日志提取超时

**原因**：日志文件过大（如 debug.log 全系列 > 500MB）或数量过多。

**解决**：
1. 在 `--time` 参数中指定时间范围缩小搜索
2. 指定具体的日志文件而非目录
3. 如频繁超时，可在 skill 安装目录的 `config/default-config.json` 中增大 `sub_agent_timeout` 值

### Q3: 代码分析无法定位根因

**原因**：日志中缺少明确的错误位置信息（如无堆栈跟踪、无文件名行号）。

**解决**：
1. 在 `--keyword` 参数中提供更多搜索关键词
2. 在 Bug 描述中明确提及出错文件或模块名
3. 检查日志是否来自当前代码分支（commit 比对结果）

### Q4: 修复验证失败

**原因**：修复方案可能不完整或引入了新问题。

**解决**：
1. 查看失败详情，判断是新引入的问题还是原有问题未修复
2. 选择"回滚"恢复到修复前状态
3. 选择"继续修复"基于失败结果调整方案

### Q5: commit 比对显示"不在已知分支中"

**原因**：日志中的 commit 来自未拉取的远程分支或已删除的分支。

**解决**：
1. 执行 `git fetch --all` 拉取所有远程分支
2. 重新运行 auto-debug
3. 如果分支已删除，确认日志对应的代码版本并手动切换

### Q6: sync-core-service 项目提示"DEBUG 日志未启用"

**原因**：用户环境中未启用 DEBUG 级别日志，排查信息可能不足。

**解决**：
1. 在应用安装目录的 `resources/add_ons/` 下创建 `logging.conf`
2. 写入 `DEBUG=TRUE`
3. 重启应用以生效

## 故障排除

### Skill 无法调用

1. 检查 `~/.claude/skills/auto-debug/SKILL.md` 文件是否存在
2. 检查 `~/.claude/skills/auto-debug/config/` 和 `references/` 目录是否存在
3. 检查 frontmatter 格式是否正确（YAML 语法）
4. 检查 `name` 字段是否为 `auto-debug`
5. 重启 Claude Code 重新加载技能

### 子 Agent 超时

1. 检查日志文件大小（`ls -la <log_path>`）
2. 检查 ripgrep 是否可用（`rg --version`）
3. 在 skill 安装目录的 `config/default-config.json` 中增大 `sub_agent_timeout`
4. 尝试缩小搜索范围（指定 `--time` 和 `--keyword`）

### 报告文件无法写入

1. 检查日志路径的写入权限
2. 检查日志路径下是否能创建 `auto-debug/` 子目录
3. 如果路径不可写，报告仍会在终端展示
4. 内联日志场景下，检查系统临时目录（`%TEMP%` 或 `/tmp`）是否可写

## 监控指标

auto-debug 每次执行后建议记录以下指标（目前需手动）：

| 指标 | 说明 | 记录方式 |
|------|------|----------|
| 执行耗时 | 从启动到报告生成的时间 | 记录起止时间戳 |
| 用户决策 | 确认修复 / 修改方案 / 仅分析 / 否决 | 记录用户选择 |
| 验证结果 | 通过 / 失败 | 记录验证命令输出 |
| 置信度 | 报告中的评分 | 记录评分值 |

## 版本管理

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0 | 2026-04-30 | 初始版本，支持 smart-sync、sync-core-service、通用项目 |
| 1.1 | 2026-05-12 | 重构：Skill 拆分为通用核心 + 按需加载的 reference 文件；支持 npx skills add 安装；配置来源简化为 skill 内置；输出路径统一为 auto-debug/ 子目录 |

## 升级指南

### 从 1.0 升级到 1.1

1. 删除旧版本 skill 目录
   ```powershell
   # Windows
   Remove-Item -Recurse -Force "$env:USERPROFILE\.claude\skills\auto-debug"
   ```
2. 用 npx skills add 重新安装
   ```bash
   npx skills add https://github.com/XiaoFeng-lin/auto-debug --skill auto-debug
   ```

### 常规更新

```bash
npx skills update auto-debug -y
```
