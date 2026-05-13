# sync-core-service 项目特定诊断指南

## 项目识别

当前项目被识别为 **sync-core-service** 项目。以下知识适用于本次诊断。

## 分支体系

| 分支名 | 用途 | 备注 |
|--------|------|------|
| `production` | 生产分支 | 当前本地分支 |
| `master` | 主开发分支 | 远程分支 |
| `production-hotfix` | 热修复分支 | 本地已拉取 |

所有发布的版本都通过 `production` 分支构建。

## 日志体系

### 日志文件

| 日志文件 | 默认路径 | 日志库 | 最大文件大小 | 最大备份数 |
|----------|----------|--------|--------------|------------|
| `node_addon.log` | `{appdata}/FangCloud/{app_name}/` | easylogging++ | 50MB | 10 |

命名规则：当前文件 `node_addon.log`，轮转后 `node_addon.log` → `node_addon.log.1` → ... `.10`。

### 日志格式

```
{时间} - {进程ID} - {线程ID} - {日志级别} - {文件名}[line:{行号}] - {消息内容}
```

示例：
```
2026-04-29 14:26:31,044 - 55448 - 35324 - INFO - transfer_manager.cpp[line:121] - init addon, commit_id: 3e80ae3
```

### 日志优先级

检索优先级：`node_addon.log`（当前文件优先，commit ID 一定在启动日志中）。

用户未指定文件名时，默认搜索 `node_addon.log*`。

## Commit ID / Version 提取与比对

### 两种版本标识

| 版本类型 | 日志格式 | 示例 | 适用范围 |
|----------|----------|------|----------|
| 新版本（commit_id） | `init addon, commit_id: {8位短hash}` | `init addon, commit_id: 3e80ae3` | 最新版本 |
| 老版本（手动版本号） | `init addon, version:{gAddonVersion}` | `init addon, version:2.0.6` | 大量存量用户 |

### 提取逻辑

1. 从日志中检索启动行（node_addon 初始化阶段，总在日志最前面）
2. 匹配到 `commit_id:` → 提取 8 位短 hash
   - 比对：`git rev-parse HEAD` → 截取前 8 位 → 比对
   - 一致 → 日志来自当前分支
   - 不一致 → `git branch -r --contains <full_hash>` → 标注对应分支
3. 匹配到 `version:` → 提取手动版本号字符串
   - 在 `git log` 和 tag 中搜索匹配的版本标签
   - 找到 → 标注日志来自该 tag 对应版本
   - 找不到 → 标注"版本号为手动维护，无法自动关联到具体 commit，请用户确认日志对应的代码版本"

### 特别处理

- 日志中同时缺失 commit_id 和 version → 标注"日志中未找到版本信息，可能来自极早期版本或日志被截断"
- 老版本号无法精确映射到 commit 时，**不得猜测**，必须向用户确认

## DEBUG 日志检查

通过 `logging.conf` 配置文件控制（放置于应用安装目录下的 `resources/add_ons/` 目录）：

```ini
DEBUG=TRUE
```

- 存在且含 `DEBUG=TRUE` → DEBUG 日志已启用
- 不存在或含 `DEBUG=FALSE` → DEBUG 日志未启用

若未启用 DEBUG 日志，诊断报告中应提示"当前环境未启用 DEBUG 日志，排查信息可能不足"。

## 核心模块参考

| 模块路径 | 职责 |
|----------|------|
| `src/transfer_manager/` | 传输任务核心管理（上传/下载/暂停/取消） |
| `src/token_manager/` | 令牌管理，认证相关 |
| `src/http_client/` | HTTP 请求封装，基于 cpprestsdk 和 cURL |
| `src/network/` | 网络层抽象 |
| `src/filesystem/` | 文件系统操作封装 |
| `src/api/` | API 响应处理与 JSON 解析 |
| `src/error/` | 错误码与异常体系 |
| `src/easyloggingpp/` | 日志库配置与自定义分发器 |
| `src/node_addon/` | Node.js 绑定层，暴露 API 给 Electron 客户端 |
| `src/lightning_client/` | （可选）基于 Qt 的独立客户端 |

## 诊断增强规则（阶段 3 执行）

作为 sync-core-service 项目，日志提取阶段需要额外执行以下步骤：

1. 默认搜索范围：用户未指定文件名时，搜索 `node_addon.log*`（包括轮转文件 `.1` ~ `.10`）
2. 从日志开头提取 commit_id 或 version
3. 执行版本比对流程（见上文）
4. 检查 DEBUG 日志可用性：查找 `logging.conf` 文件
5. 在返回的结构化数据中包含 `commit_info` 和 `debug_logging_enabled` 字段
