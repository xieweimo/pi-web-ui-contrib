# [API] 为插件增加按客户端隔离的动态 bottombar widget

## 问题

现有 UI slot 适合声明式静态文本/badge；`host.ui.update` 是全局更新。额度、会话状态等内容需要每个浏览器客户端不同的实时值。当前插件只能查询 `.statusbar`、插入 DOM、匹配内置 title、注入全局样式并使用 `MutationObserver` 对抗 React 重渲染，既脆弱又无法隔离多标签页状态。

## 提议

manifest 增加 `kind: "widget"`：

```json
{
  "permissions": ["ui"],
  "ui": {
    "bottombar": [
      { "id": "summary", "kind": "widget", "renderer": "usageSummary", "order": 11, "align": "end" }
    ]
  }
}
```

client bundle：

```js
export default {
  widgets: {
    usageSummary: {
      mount(container, context) {
        // context.onData 仅接收当前 clientId 的 plugin_data
        // 返回 cleanup
      },
    },
  },
};
```

## 约束

- 宿主创建稳定容器并负责 React 生命周期；插件只写自己的容器，不查找/移动宿主节点。
- `context.onData` 必须按 `clientId` 隔离；全局 `host.ui.update` 继续服务声明式条目。
- 内置成本项应提供稳定 id（例如 `host:cost`），隐藏/恢复走 `ui.arrange` 或用户布局，不能匹配展示文案。
- 同一帧的数据变化合并为一次渲染；禁止全局 `MutationObserver`、轮询或向 `document.head` 注入样式。

## 验收

- 两标签页各显示自己的模型/额度，互不串页；
- 窄屏不压扁 widget，插件卸载时清理容器和订阅；
- 可在布局页恢复被隐藏的内置成本项；
- 宿主重渲染后 widget 不重复挂载、不泄漏监听器。

## 解除的本地改动

`projects/codex-usage-plugin/client/entry.mjs` 中的 `.statusbar` 查询、DOM 注入、`MutationObserver`、title 匹配与全局 CSS。
