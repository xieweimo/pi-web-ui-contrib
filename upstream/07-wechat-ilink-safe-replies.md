# [Plugin PR] wechat-ilink：过滤内部控制 marker，并按微信用户隔离无头会话

## 问题

微信转发会把面向 pi-web-ui 的内部控制 marker 原样发给外部用户，例如 `[[plan:…]]`、`[[todo:…]]`、`[[conv:…]]`、`[[notify:…]]`。此外，多个微信用户若复用无头会话标识，可能串上下文。

## 提议修改（仅 `plugins/wechat-ilink`）

1. 对外发送前在 `externalReplyText()` 过滤上述内部 marker，保留用户可读正文；
2. `accountIdForPeer()` 基于 peer id 生成稳定、彼此隔离的无头会话 id；
3. 不改工具名称、token 存储、扫码/配对流程和已有配置格式。

## 验收

- 含 Plan/Todo/Conv/Notify marker 的回复，微信侧只见可读正文；
- 两个微信用户并发对话，互不共享上下文；
- 重启后同一 peer 仍回到自己的会话；
- 既有配对与存储不迁移、不丢失；
- `wechat-ilink` 工具名和 manifest id 保持兼容。

## 本地实现证据

受管同 id fork：`projects/wechat-ilink-plugin/`；测试：`projects/wechat-ilink-plugin/tests/fork-policy.test.mjs`。上游接受后应删除 fork 和兼容补丁。
