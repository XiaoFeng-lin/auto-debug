# auto-debug 测试用例

## 测试用例 1：内联日志 — smart-sync 逻辑缺陷

### 输入
```
/auto-debug "同步文件夹时，重命名操作后旧文件未删除" --log "2026-04-28 14:26:31,044 - 55448 - 35324 - ERROR - sync_handler.py[line:142] - Rename operation failed: target already exists
2026-04-28 14:26:31,045 - 55448 - 35324 - WARNING - sync_handler.py[line:150] - Skipping cleanup, lock held by process 3821"
```

### 预期行为
1. 阶段 1：校验通过，提取关键词 "sync_handler", "Rename", "target already exists", "lock"
2. 阶段 2：识别为 smart-sync 项目（如果在 smart-sync 项目目录下运行）
3. 阶段 3：内联日志直接结构化，跳过文件检索
4. 阶段 4：定位 `sync/sync_handler.py:142`，分析重命名和清理逻辑
5. 阶段 5：生成报告，根因分类为"并发问题"或"遗漏场景"
6. 置信度 ≥ 60/100

---

## 测试用例 2：目录日志 — smart-sync 并发问题

### 前置条件
- 在 smart-sync 项目根目录下运行
- 日志目录 `D:\bug_log\smart_sync\Fangcloud\SmartSync` 包含 debug.log 和 monitor.log

### 输入
```
/auto-debug "文件上传超时后无限重试" --log "D:\bug_log\smart_sync\Fangcloud\SmartSync" --time "2026-04-29 10:00~11:00"
```

### 预期行为
1. 阶段 1：校验通过，时间范围 "2026-04-29 10:00~11:00"
2. 阶段 2：识别为 smart-sync 项目，日志路径存在且可读
3. 阶段 3：子 Agent 在 debug.log* 和 monitor.log* 中搜索关键词
4. 阶段 4：分析上传重试逻辑
5. 提取 commit_hash 并比对分支

---

## 测试用例 3：单文件日志 — sync-core-service

### 前置条件
- 在 sync-core-service 项目根目录下运行
- 日志文件 `D:\bug_log\sync_core_service\FangCloud\Drive\node_addon.log` 存在

### 输入
```
/auto-debug "下载文件时进度卡在 99%" --log "D:\bug_log\sync_core_service\FangCloud\Drive\node_addon.log"
```

### 预期行为
1. 阶段 2：识别为 sync-core-service 项目
2. 阶段 3：子 Agent 搜索 node_addon.log
3. 提取 commit_id 或 version
4. 阶段 4：定位传输管理模块代码
5. 检查 DEBUG 日志是否启用

---

## 测试用例 4：通用项目

### 前置条件
- 在一个普通 Node.js 项目目录下运行

### 输入
```
/auto-debug "用户登录后 session 丢失" --log "/var/log/app/error.log"
```

### 预期行为
1. 阶段 2：识别为通用项目
2. 不执行 commit 比对
3. 报告中不包含"项目特定信息"章节

---

## 边界测试用例

### BT-1：Bug 描述为空
```
/auto-debug "" --log "..."
```
预期：暂停，提示 E0101

### BT-2：日志路径不存在
```
/auto-debug "程序崩溃" --log "D:\nonexistent\path"
```
预期：暂停，提示 E0203

### BT-3：当前目录非项目根目录
预期：暂停，提示 E0201

### BT-4：日志匹配行数超过上限
预期：触发聚合摘要，提示 E0303

### BT-5：修复方案与实际代码不符
预期：暂停，提示 E0601

### BT-6：修复验证失败
预期：提示用户选择 回滚/继续修复/终止

### BT-7：用户选择"否决"
预期：流程终止，不做任何代码修改
