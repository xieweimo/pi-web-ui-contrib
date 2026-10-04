# patches —— 对 pi-web-ui / pi 的补丁

这些脚本**就地修改 pi-web-ui（或 pi 内核）的编译产物**，因此与版本强绑定：

- 升级 pi-web-ui / pi 之后必须重新执行；
- 脚本按锚点匹配，源码变了会以退出码 2 拒绝盲改，而不是把文件改坏；
- 大多数脚本是幂等的（已应用则跳过），部分支持 `--remove` 回滚。

## 依赖

所有补丁都 require 同仓库的定位模块，请保持 `patches/` 与 `scripts/` 的相对位置：

```bash
node patches/patch-pi-web-ui-plugin-per-client-conversation.js
node patches/patch-pi-invalid-toolcall-names.js
```

- `scripts/pi-web-ui-locate.js` —— 定位 pi-web-ui 包目录；
- `scripts/pi-core-locate.js` —— 定位机器上的 pi 内核副本（可能有全局 CLI / 内嵌 / 源码项目多份，补丁必须逐份处理）。

## 现役补丁

| 补丁 | 作用 |
| --- | --- |
| `patch-pi-web-ui-plugin-live-model.js` | 插件会话快照补「当前模型」，并在 `setModel()` 成功后立刻通知插件 |
| `patch-pi-web-ui-plugin-per-client-conversation.js` | 插件快照按浏览器客户端（`clientId`）取「该页面正在看的对话」，并补 `isSubagent`；全局兜底跳过子代理会话。**会话文件路径 `sessionFile` 自 pi-web-ui 0.99.0 起由宿主原生提供，本补丁不再自己加** |
| `patch-pi-web-ui-plugin-topbar-cache.js` | 插件顶栏入口首帧时序（刷新后不再晚几百毫秒出现） |
| `patch-pi-web-ui-topbar-menu-buttons.js` | 把原生菜单项提升到顶栏 |
| `patch-pi-web-ui-plan-board-clear.js` | 看板清空不用同步 `window.confirm()`，避免冻结主线程 |
| `patch-pi-web-ui-plan-marker.js` | 可注册的内联 marker（插件能自行扩展新协议） |
| `patch-pi-web-ui-sw-entry-revalidate.js` | Service Worker 对入口 bundle 强制回源重校验（所有前端补丁的缓存前提） |
| `patch-pi-invalid-toolcall-names.js` | 唯一改 **pi 内核**的补丁：把被正文污染的非法 `toolCall.name` 净化，避免 `Invalid 'input[N].name'` 400 永久毒化历史 |
| `patch-pi-web-ui-free-models-only.js` | 模型下拉只下发免费/订阅/白名单模型（**依赖本项目的白名单配置，属项目专用**） |
| `patch-pi-web-ui-free-model-badge.js` | 给免费模型加绿色「免费」徽标（项目专用，与上一条配套） |

> `sw-entry-revalidate` 只在前端补丁存在时需要；当全部前端改动都被上游吸收后，它也应一并退役。

## 已被上游原生实现，不再需要

| 能力 | 上游实现位置 |
| --- | --- |
| 代理传递 / Codex 连接（`fetch failed`） | `server/http-proxy.ts`（启动时读 `settings.json` 的 `httpProxy` 与系统代理，注入全局 dispatcher） |
| 最近项目删除的全局持久化（tombstone） | `server/client-state.ts` |
| Fork 会话历史去重 | `server/agent-service.ts`（按 `parentSessionPath` 隐藏被替代的父会话） |
| 逐条 assistant 消息 `usageCost` | `server/protocol.ts`、`server/serialize.ts` |
| 当前模型同步到插件 | `server/plugins.ts`、`web/src/plugin-host.ts`、`web/src/App.tsx`（含 `host.models.active()` / `onChange`） |
| 停止按钮高对比 + 脉冲 | `web/src/styles.css` |
| 快捷短语排队 | 上游内建右键排队 |

因此下面这些补丁已从本项目退役，不再分发：`usage-cost`、`hide-forked-sessions`、`permanent-project-ignore`、`dangling-tool-calls`、`quick-phrase-queue`、`apply-page-picker-all-urls`（上游出于同源与隐私明确不合并全站免授权）。

## 风险提示

- 补丁改的是**产物**，不是源码：上游每次发版都可能让锚点失配；
- 只在你自己的机器上执行，别把打过补丁的产物当成上游发行版分发；
- 长期方向是把这些改动提成上游 PR（见 `../upstream/`），补丁只是过渡。
