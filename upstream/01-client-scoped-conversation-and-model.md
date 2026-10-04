# [API] 向插件提供按客户端的会话快照与模型变更事件

> 状态：草案（未提交）。**0.99.0 已经原生提供了** `sessionId` / `sessionFile` / `sessionDir` / `model`
> 四个快照字段，所以本文只申请**剩下的缺口**：按 `clientId` 取「这个页面正在看的对话」、
> 子代理标记，以及模型切换事件。已提供的部分记录在文末「0.99.0 已提供」一节，供参考，不再重复申请。

## 问题

`host.getActiveConversation()` 只能返回全局“最近活跃”会话（`readConversationForPlugins()` 里取
`lastActiveAt` 最大的那个），且没有按客户端的取用方式。于是：

1. 两个浏览器标签页分别查看不同对话时，插件会读取到另一个标签页的会话（典型现象：A 页选 DeepSeek，
   底部却显示 openai-codex 的额度窗口）；
2. 子代理会话往往比页面正在看的对话更活跃，会把后者挤掉；
3. 用户在页面上刚切模型、还没发下一条消息时，插件拿不到任何“模型变了”的通知，只能等下一轮刷新。

这迫使我们本地使用 `patch-pi-web-ui-plugin-per-client-conversation.js` 与
`patch-pi-web-ui-plugin-live-model.js` 修改宿主运行产物。

## 复现

1. 打开两个浏览器标签页，各进入不同会话；
2. A 页选择 DeepSeek，B 页选择 openai-codex；
3. 在 A 页观察依赖 `getActiveConversation()` 的插件：它可能显示 B 页的会话/模型；
4. A 页刚切模型且不发消息时，它仍显示切换前的模型；
5. 同时跑一个子代理（它更活跃），页面正在看的对话会被挤出快照。

## 提议 API（新增且兼容）

```ts
type ClientConversationSnapshot = {
  clientId: string;
  conversationId: string | null;
  /** 该对话是否为子代理会话（子代理不该占据页面默认快照）。 */
  isSubagent: boolean;
  // 以下字段 0.99.0 已原生提供，这里只列出调用方会用到的部分
  sessionId?: string;
  sessionFile?: string;
  sessionDir?: string;
  model?: string;
  title: string;
  at: number;
  isStreaming: boolean;
  messages: UiMessage[];
  streamingMessage: UiMessage | null;
  stats: {
    totalMessages: number;
    tokens: { input: number; output: number; total: number };
    cost: number;
  };
};

host.getActiveConversation(options?: {
  /** 取该客户端当前打开的对话；插件不得用它读取别的页面。 */
  clientId?: string;
  fallback?: "latest-non-subagent";
}): ClientConversationSnapshot | null;

host.onClientConversationChanged(
  callback: (snapshot: ClientConversationSnapshot) => void,
): () => void;

/** 模型切换后立即回调（0.99.0 的 host.models 只有 list()，没有变更事件）。 */
host.onClientModelChanged(
  callback: (snapshot: ClientConversationSnapshot) => void,
): () => void;
```

## 语义

- `clientId` 必须来自当前插件消息/接入的宿主连接上下文；插件不得任意指定 id 读取其它标签页或用户的会话。
- 未传 `clientId` 时才回落「最近活跃的**非子代理**会话」，保持现有行为兼容。
- `onClientModelChanged` 只在 `setModel()` 成功后发出，不轮询；切换失败不发事件。
- `isSubagent` 为真表示该对话是子代理：默认 fallback 必须跳过它。

## 验收

- 两标签页各取得自己的 `conversationId`、`model` 与消息计数；
- 子代理运行中，页面正在看的对话不会被挤出快照；
- 页面切模型后不发消息，也能立即收到模型变更事件；失败切换不产生事件；
- 重连/重复 attach 不产生重复事件。

## 非目标

不下发 provider 密钥、会话文件**内容**、其它客户端的消息内容。

## 0.99.0 已提供（不再申请）

以下字段已由宿主原生写进快照（`server/agent-service.js` 的 `readConversationForPlugins()` 返回值），
本项目对应的自造字段已从补丁里删除：

| 字段 | 用途 | 我们为什么要它（实测背景） |
| --- | --- | --- |
| `sessionFile?: string` | 会话 JSONL 绝对路径 | 成本/额度类插件要读**完整**账本；没有它只能按当前上下文估算，上下文一压缩、或关掉对话重开就显示 0 |
| `sessionId?: string` | 持久化会话 UUID | 跨重启稳定的身份键（内存态 `c1`/`c4` 重启即失效，也≠文件名） |
| `sessionDir?: string` | 会话物理目录 | 定位历史/归档，避免自己拼路径 |
| `model?: string` | 当前选中模型的 canonical id | 不必再从最后一条历史消息猜模型 |

> 经验记录：在 0.99.0 之前，我们只能在本地补丁里往快照塞 `sessionFile` 才让账本不归零；
> 对插件的正确姿势是**优先使用宿主原生字段**、字段缺失时才降级，绝不能自己造同名键覆盖宿主值。
