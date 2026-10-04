# upstream —— 提给 pi-web-ui 上游的 issue / PR 草案

这里放**准备提交给上游**的提案（一项能力或一个缺陷一份），以及它们的当前状态。上游作者已公开表示欢迎这类贡献。

## 当前状态

| 草案 | 类型 | 解决什么 | 解除哪些本地补丁 | 状态 |
| --- | --- | --- | --- | --- |
| `01-client-scoped-conversation-and-model.md` | API 新增 | 插件只能拿到「全客户端最近活跃对话」，也不知道模型何时切换 → 多标签/并行对话/子代理时插件串页 | `plugin-per-client-conversation`、`plugin-live-model` | 待提交（0.99.0 已原生提供 `sessionFile`/`sessionId`/`sessionDir`/`model`，本项已收窄为 `clientId` + `isSubagent` + 模型变更事件） |
| `02-client-scoped-ui-widget.md` | API 新增 | 插件想往按页面隔离的状态栏放动态内容，只能靠宿主私有 DOM | 额度插件的 `.statusbar` DOM 注入 | 待提交 |
| `03-extensible-inline-marker-plan.md` | API 新增 | 新协议必须改宿主产物，插件无法自行扩展 | `plan-marker` | 待提交 |
| `04-dangling-tool-call-healer.md` | bug | 悬空 toolCall 修复会补出孤立 `function_call_output` → 重试 400 且越修越多 | `dangling-tool-calls` | **上游 0.96.0 已内建收紧**，草案转为验证资料 |
| `05-plugin-topbar-first-paint.md` | 性能 | 刷新后插件顶栏入口晚几百毫秒出现 | `plugin-topbar-cache` | 待提交 |
| `06-plan-board-clear-nonblocking.md` | UX/性能 | 看板清空用同步 `confirm()` 冻结主线程 | `plan-board-clear` | 待提交 |
| `07-wechat-ilink-safe-replies.md` | 插件 PR | 微信通道安全回复 | fork 与兼容补丁 | 待提交 |

## 待办：插件市场条目路径更新

本仓库从 `pi-web-ui-reconnect-watchdog` 改名为 `pi-web-ui-contrib`，插件目录从 `reconnect-plugin/` 改为 `plugins/reconnect/`：

- git / 网页形式的旧仓库链接会被 GitHub 自动重定向；
- 但 `raw.githubusercontent.com` 的旧**路径**不会重定向；
- 上游默认插件列表里 `reconnect` 条目的 `source` 仍是 `xieweimo/pi-web-ui-reconnect-watchdog/reconnect-plugin`，需要在 `xing-shuyin/pi-web-ui` 的 `plugins/catalog.json` 里改成 `xieweimo/pi-web-ui-contrib/plugins/reconnect`（一行 PR）。

> 已安装过旧来源的用户不受影响：更新检查读的是插件目录里的 `.pi-source.json`，不是市场条目。

## 提交要求

1. 先确认上游没有同义 issue；
2. 草案被认可后单独建分支、单独实现，不混入其他改动；
3. PR 必须带草案里的回归用例，并保持旧 API 兼容；
4. 上游版本在真实页面验收通过后，才从本地移除对应补丁。
