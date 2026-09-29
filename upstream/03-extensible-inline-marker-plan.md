# [API] 允许插件注册内联 marker，并提供受控的 Plan 状态能力

## 问题

宿主内置 `[[todo:…]]`、`[[conv:…]]`、`[[notify:…]]` marker，但插件不能注册 namespace。Plan 看板需要模型仅写正文即可更新持久化状态；本地实现只能修改 marker service、plan manager、会话快照恢复和前端产物。

## 提议 API（v12）

```ts
host.registerInlineMarker({
  namespace: "plan",
  permissions: ["plan"],
  getGuidance(locale): string[],
  init(): unknown,
  apply(token, context, locale): { applied: boolean; error?: string },
}): () => void;

host.plan.get(conversationId): Plan | null;
host.plan.set(conversationId, steps, activeStepId?): Plan;
host.plan.clear(conversationId): void;
host.plan.onChanged(callback): () => void;
```

## 约束

- `namespace` 全局唯一；冲突必须拒绝且给出可读诊断。
- marker 只能处理自己注册的 namespace；卸载后正文保留原样，不再被解析。
- Plan 由宿主持久化在会话快照，插件不能依赖私有 session 文件格式。
- 宿主统一校验步骤 id、状态枚举与“最多一个 active”；`plan_update` 与 marker 走同一状态机。
- `permissions: ["plan"]` 独立于普通 `ui` 权限，用户能在插件设置中看到和撤销。

## 验收

- `[[plan:new:1=调研,2=实现,active=1]]` 正确创建看板；
- 非法状态/未知步骤返回错误且不污染旧状态；
- 服务重启后从快照恢复；
- 卸载插件后不再解析它的 namespace；
- `plan_update` 和 marker 混用时状态一致。

## 解除的本地改动

`patch-pi-web-ui-plan-marker.js` 及其对 `dist/server`、前端 bundle 的耦合修改。
