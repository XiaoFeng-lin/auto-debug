# Changelog

## 1.0.0 (2026-05-12)

### 架构重构
- **拆分 SKILL.md**：将 728 行的 monolithic Skill 拆分为 435 行通用核心 + 按需加载的 `references/*.md`
- **渐进式披露**：项目特定知识（smart-sync、sync-core-service）仅在识别到对应项目时动态加载，减少无关上下文
- **插件化适配**：新增 `config/project-adapters-config.json` 适配器注册表，新增项目支持无需修改 SKILL.md

### 配置简化
- **取消项目级配置**：不再读取项目根目录的 `.auto-debug-config.json`
- **取消全局配置**：不再读取 `~/.auto-debug/config.json`
- **唯一配置来源**：skill 安装目录下的 `config/default-config.json`

### 输出路径调整
- **统一子目录**：所有输出文件（提取日志、诊断报告）统一放在日志路径下的 `auto-debug/` 子目录中
- **内联日志场景**：输出保存到系统临时目录（`%TEMP%/auto-debug/` 或 `/tmp/auto-debug/`）
- **零项目污染**：不再在项目目录或日志根目录直接创建文件

### 工程化
- 新增 `package.json`，配置 npm `files` 白名单，开发文档（docs/、tests/、CLAUDE.md 等）不进入安装包
- 新增 `CHANGELOG.md`
- 更新 `README.md`，反映新的项目结构和扩展方式
