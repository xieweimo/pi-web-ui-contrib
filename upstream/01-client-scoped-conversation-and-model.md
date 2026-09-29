# [API] 向插件提供按客户端的会话快照与当前模型事件

## 问题

`host.getActiveConversation()` 当前只能返回全局“最近活跃”会话，且快照不包含 session 当前选择的模型。两个浏览器标签页分别查看不同对话时，插件会读取到另一个标签页的会话；用户刚切模型、尚未发送下一条消息时，插件仍显示旧模型。

这迫使本地使用 `patch-pi-web-ui-plugin-per-client-conversation.js` 与 `patch-pi-web-ui-plugin-live-model.js` 修改宿主运行产物。

## 复现

1. 打开两个浏览器标签页，各进入不同会话；
2. A 页选择 DeepSeek，B 页选择 openai-codex；
3. 在 A 页观察依赖 `getActiveConversation()` 的插件；
4. 它可能显示 B 页会话/模型；A 页刚切模型且不发消息时，它仍显示切换前模型。

## 提议 API（v12，新增且兼容）

```ts
type ClientConversationSnapshot = {
  clientId: string;
  conversationId: string | null;
  isSubagent: boolean;
  activeModel: { provider: string | null; model: string | null } | null;
  // 保留现有 PluginConversationSnapshot 的安全字段
};

host.getActiveConversation(options?: {
  clientId?: string;
  fallback?: "latest-non-subagent";
}): ClientConversationSnapshot | null;

host.onClientConversationChanged(
  callback: (snapshot: ClientConversationSnapshot) => void,
): () => void;

host.onClientModelChanged(
  callback: (snapshot: ClientConversationSnapshot) => void,
): () => void;
```

## 安全与语义

- `clientId` 必须来自当前插件消息的宿主连接上下文；插件不得任意指定 id 读取其它用户/标签页会话。
- `activeModel` 必须来自当前 session 选择，而不是从最后一条历史消息猜测；没有选择时返回 `null`。
- `onClientModelChanged` 只在 `setModel()` 成功后发出，不轮询；失败切换不发事件。
- 无 `clientId` 时才回落最近活跃的非子代理会话，保持现有兼容行为。

## 验收

- 两标签页各取得自己的 conversation id、模型与消息计数；
- 切模型后不发消息仍立即收到新模型；
- 失败切换不产生模型事件；
- 子代理不污染默认 fallback；重连不产生重复事件。

## 非目标

不下发 provider 密钥、完整 session 文件、其它客户端的消息内容。
