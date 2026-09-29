# [Performance] 在首帧区分“插件清单未到达”与“确实为空”

## 问题

页面初始快照到达前，插件清单与空清单不可区分；顶栏插件入口要等会话 attach 和插件重扫，产生明显空白。实测：冷启动约 982ms，热启动 156–290ms。

本地 workaround 把最小 manifest 顶栏信息缓存到 `localStorage`，并用 `pluginsLoaded` 区分“未到达”与“空清单”；权威清单到达后立即接管。

## 提议

把以下语义纳入稳定协议：

```ts
type PluginSnapshot = {
  plugins: UiPluginInfo[];
  pluginsLoaded: boolean;
  pluginsEpoch?: number;
};
```

- `pluginsLoaded: false`：前端可以使用受限的首帧缓存；
- `pluginsLoaded: true, plugins: []`：权威空清单，必须立即移除缓存入口；
- `pluginsEpoch` 可用于失效缓存，不缓存 bundle、工具或敏感配置。

缓存内容仅限 `id/name/icon` 和 `topbar.primary` 的声明式信息。

## 验收

- 装三个顶栏插件后刷新：入口在首帧预算内出现，无幽灵条目；
- 服务端删除插件后刷新：旧入口立即消失；
- 未安装插件：不显示缓存入口；
- 清单到达后 UI 与不使用缓存时完全一致。

## 本地临时修复

`patch-pi-web-ui-plugin-topbar-cache.js`；当协议语义稳定并在真实页面回归后移除。
