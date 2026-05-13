# smart-sync 项目特定诊断指南

## 项目识别

当前项目被识别为 **smart-sync** 项目。以下知识适用于本次诊断。

## 分支体系

| 分支名 | 用途 | 备注 |
|--------|------|------|
| `production_hubble_gray` | 普通同步版本（常用） | 本地已拉取 |
| `production_hubble_scatter` | 散落同步版本 | 基于 gray 扩展，公有云主要使用 |
| `production` | 公有云测试/生产包 | 远程分支 |
| `private_cloud` | 专有云生产包 | 远程分支 |

分支关系：`scatter` 是 `gray` 的扩展功能版本。

## 日志体系

### 日志文件清单

| 日志文件 | Logger 名称 | 级别 | 最大备份 | 单文件上限 |
|----------|-------------|------|----------|------------|
| `debug.log` | `sync.core` | DEBUG | 10 | 50MB |
| `error.log` | `sync.core` | ERROR | 2 | 50MB |
| `monitor.log` | `sync.monitor` | DEBUG | 10 | 50MB |
| `conflict.log` | `sync.conflict` | DEBUG | 2 | 50MB |
| `alive.log` | `sync.alive` | DEBUG | 1 | 50MB |

命名规则：当前写入 `debug.log`，轮转后 `debug.log` → `debug.log.1`，最多保留 N 个备份。

### 日志格式

```
{时间} - {进程ID} - {线程ID} - {日志级别} - {文件名}[line:{行号}] - {消息内容}
```

示例：
```
2026-04-28 14:26:31,044 - 55448 - 35324 - INFO - smart_sync.py[line:66] - meta: {"commit_hash": "557414e43f44211d7276bc5e537bba4af7c91e29"}
```

### 日志优先级

检索优先级：`debug.log` > `monitor.log` > `error.log` > `conflict.log` > `alive.log`

用户未指定文件名时，默认搜索 `debug.log*` 和 `monitor.log*`。

## Commit Hash 提取与比对

### 提取

从日志中检索启动行：
```
smart_sync.py[line:66] - meta: {"commit_hash": "<hash>"}
```

提取最长 40 位 hex 字符串作为 commit_hash。

### 比对流程

1. 在项目根目录执行 `git rev-parse HEAD` 获取当前 HEAD
2. 如果当前 HEAD 与日志中的 commit_hash 一致 → 日志来自当前分支
3. 如果不一致 → 执行 `git branch -r --contains <commit_hash>`（只查本地已拉取分支）
4. 根据匹配结果记录分支信息

### 分支标注规则

- commit 属于 `production_hubble_scatter` → 标注"此日志来自散落同步版本"
- commit 属于 `production_hubble_gray` → 标注"此日志来自普通同步版本"
- 未匹配到任何分支 → 标注"日志对应的 commit 不在已知分支中，可能来自已删除的分支或已 rebase 的提交"
- 日志中未找到 commit_hash → 标注"未找到 commit_hash 信息，无法执行分支比对"，置信度评分 -20 分

## 核心模块参考

| 模块路径 | 职责 |
|----------|------|
| `sync/sync_engine.py` | 同步引擎核心，协调本地/云端同步逻辑 |
| `sync/application.py` | 应用主类，初始化各组件 |
| `sync/task/` | 任务队列与调度 |
| `sync/command/` | 同步命令体系（上传/下载/删除/重命名等） |
| `sync/filesystem/` | 文件系统监控与操作 |
| `sync/httplayer/` | HTTP 请求层，API 客户端 |
| `sync/realtime/` | 实时消息处理 |
| `sync/storage/` | 本地数据库存储 |
| `sync/communication/` | 进程间通信 |
| `sync/system_extension/` | 系统扩展（图标覆盖、统计等） |
| `sync/dlp/` | 数据防泄漏 |
| `sync/logger/` | 日志体系 |

## 诊断增强规则（阶段 3 执行）

作为 smart-sync 项目，日志提取阶段需要额外执行以下步骤：

1. 默认搜索范围：用户未指定文件名时，优先搜索 `debug.log*` 和 `monitor.log*`
2. 从提取的日志中搜索 commit_hash 启动行
3. 执行 commit 比对流程（见上文）
4. 在返回的结构化数据中包含 `commit_info` 字段
