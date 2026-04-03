# 私有 GitLab 一句话发起 MR + 合并后钉钉通知 设计文档

## 1. 背景与目标

团队希望开发者通过一句命令完成 MR 发起，并在 MR 合并后自动通知相关人员（重点通过手机号 @）。

本方案目标：
- 开发者本地执行 `mr <源分支> 到 <目标分支>` 即可自动 push 并创建 MR。
- MR 状态变为 merged 后，系统自动向钉钉群机器人发送通知。
- 通知对象为“固定手机号 + MR 关联人员手机号”合并去重后的集合。

非目标：
- 不在 CLI 里轮询 merged 状态。
- 不实现钉钉企业内部应用消息。

## 2. 方案选型

采用方案 A：CLI + GitLab API + GitLab Webhook 通知服务 + 钉钉群机器人。

原因：
- 职责清晰：CLI 只负责创建 MR，服务端只负责 merged 通知。
- 稳定性高：通知链路不依赖开发者本地进程存活。
- 可运维：服务端日志、幂等、重试、审计更容易实现。

## 3. 系统架构

### 3.1 `mr-cli`（开发者本地）
职责：
- 解析一句话命令。
- 校验分支与远程仓库状态。
- 执行 `git push -u origin <source>`。
- 调用 GitLab API 创建 MR。
- 若同源/同目标已存在未关闭 MR，则返回已有 MR 链接并结束。

不负责：
- 监听 MR merged。
- 直接调用钉钉通知。

### 3.2 `merge-notifier`（服务端）
职责：
- 接收 GitLab Merge Request Hook。
- 仅处理 `state=merged` 的事件。
- 聚合相关人员手机号（动态 + 固定），去重后调用钉钉机器人。
- 执行幂等去重与失败重试。

## 4. 命令与配置设计

### 4.1 命令格式

```bash
mr <源分支> 到 <目标分支> [--title "标题"] [--desc "描述"]
```

示例：

```bash
mr feat/login 到 main
mr release/1.2.0 到 prod --title "release: 1.2.0"
```

### 4.2 CLI 执行流程

1. 校验当前目录是 Git 仓库且存在 `origin`。
2. 校验源分支存在。
3. 执行 `git push -u origin <源分支>`。
4. 查询是否已有同源/同目标的 opened MR。
5. 若存在则返回 MR URL；若不存在则创建新 MR 并返回 URL。

### 4.3 配置项

优先级：命令行参数 > 环境变量 > 配置文件。

- `GITLAB_BASE_URL`
- `GITLAB_TOKEN`
- `GITLAB_PROJECT_ID`
- `GITLAB_WEBHOOK_TOKEN`（notifier 校验 GitLab webhook）
- `DINGTALK_WEBHOOK`
- `DINGTALK_SECRET`（可选）
- `NOTIFY_FIXED_MOBILES`（逗号分隔）

## 5. 通知规则

### 5.1 触发条件

仅当 GitLab webhook 事件满足以下条件时发送钉钉通知：
- `object_kind == "merge_request"`
- `object_attributes.state == "merged"`

### 5.2 相关人员手机号来源

1. 固定手机号：来自 `NOTIFY_FIXED_MOBILES`。
2. 动态手机号：MR 关联人员（作者、reviewers、assignees）通过映射获得手机号。

说明：动态手机号映射优先策略：
- 优先读取维护的映射表（`gitlab_user_id -> mobile`）。
- 如后续确定私有 GitLab 提供可用手机号字段，可扩展直接读取用户资料。

### 5.3 通知内容

钉钉消息包含：
- 项目名
- MR 标题
- 源分支 -> 目标分支
- 合并人
- 合并时间
- MR 链接

并使用 `atMobiles` @ 去重后的手机号集合。

## 6. 幂等、重试与错误处理

### 6.1 幂等键

使用 `project_id + mr_iid + merged_at` 作为幂等键。

行为：
- 首次处理：发送并记录幂等键。
- 重复回调：直接返回 200，不重复发送。

### 6.2 重试策略

钉钉发送失败时进行最多 3 次重试：
- 第 1 次立即
- 第 2 次延迟 2s
- 第 3 次延迟 4s

重试后仍失败：
- 返回 500 给 GitLab（允许其按 webhook 策略再次投递）
- 记录结构化错误日志（含事件标识、响应码、错误信息）

### 6.3 入参校验

- webhook token 不匹配：401
- 非 MR 事件或非 merged：200（忽略）
- 必需字段缺失：400（记录日志）

## 7. 测试设计（TDD）

### 7.1 CLI 测试

- 命令解析：`mr a 到 b` 解析正确。
- 已存在 opened MR：不重复创建。
- push 失败：返回明确错误。
- 创建 MR 失败：返回 API 错误并退出非 0。

### 7.2 Notifier 测试

- merged 事件会触发发送。
- 非 merged 事件被忽略。
- 固定+动态手机号合并去重正确。
- 幂等重复事件不会重复发送。
- 钉钉失败触发重试并按策略终止。

## 8. 可观测性与运维

- 统一结构化日志（JSON）字段：`trace_id`、`project_id`、`mr_iid`、`event`、`status`。
- 提供 `/healthz` 健康检查端点。
- 建议在部署层配置告警：连续发送失败次数超阈值时告警。

## 9. 风险与约束

- 动态手机号依赖映射数据完整性；缺失时只能通知固定手机号。
- 私有 GitLab 版本差异可能影响 MR 字段结构，需在实现阶段做兼容测试。

## 10. 里程碑

1. 里程碑 A：CLI MVP（命令解析 + push + 创建 MR）。
2. 里程碑 B：Notifier MVP（merged 触发 + 钉钉发送）。
3. 里程碑 C：幂等重试、日志、手机号映射与补全测试。
