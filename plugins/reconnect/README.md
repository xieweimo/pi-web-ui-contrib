# reconnect（连接与服务控制）

独立插件，与「同步」无关。它读取 recovery watchdog 的 descriptor 状态，在同一个页面提供两组操作：

## 用户推荐使用

| 操作 | 影响范围 |
| --- | --- |
| 刷新页面并重新连接 | 只重载当前浏览器页面和 WebSocket，不重启任何进程 |

## 开发者 / AI 改代码推荐使用

按钮不是写死的。插件根据 descriptor 的 `services` 动态生成服务卡片：

| 操作 | 影响范围 |
| --- | --- |
| 重启 Vite 前端 | 只重启开发页面与 HMR 服务，Node 后端不动 |
| 重启 Node 后端 | 只重启 API、WebSocket、会话和插件后端，Vite 不动 |
| 完整重启全部服务 | 停止 descriptor 中的全部受管进程，再按 `order` 启动新进程 |
| 启动/停止某项服务 | 只操作对应服务，其他服务不动 |

正式版通常只有一个网页服务，因此只生成一个服务卡片；开发版分别登记 `frontend` 和 `backend`。以后 descriptor 增加其他服务，界面会自动增加对应卡片。

## 多服务 restart descriptor

示例见 [`examples/restart-descriptor.example.json`](../../examples/restart-descriptor.example.json)。主要结构：

```json
{
  "label": "pi-web-ui 源码开发环境",
  "profile": "development",
  "watchdogPort": 8791,
  "primaryServiceId": "backend",
  "services": {
    "backend": {
      "label": "Node 后端",
      "kind": "backend",
      "command": "node",
      "args": ["npm-cli.js", "run", "dev:server"],
      "servicePort": 8788,
      "healthUrl": "http://localhost:8788/api/health",
      "actions": ["restart", "start", "stop"],
      "order": 10
    },
    "frontend": {
      "label": "Vite 前端",
      "kind": "frontend",
      "command": "node",
      "args": ["npm-cli.js", "run", "dev:web"],
      "servicePort": 5173,
      "healthUrl": "http://localhost:5173/",
      "actions": ["restart", "start", "stop"],
      "order": 20
    }
  }
}
```

公共字段 `cwd`、`env`、`startupTimeoutMs` 和 `stop` 会被每个服务继承，也可在服务内覆盖。旧版单服务 descriptor 仍兼容。

## HTTP API

```text
GET  /state
POST /start                         启动全部服务
POST /restart                       重启全部服务
POST /stop                          停止全部服务
POST /services/<id>/start           启动指定服务
POST /services/<id>/restart         重启指定服务
POST /services/<id>/stop            停止指定服务
```

watchdog 只监听 `127.0.0.1`。默认只允许操作自己启动并记录 PID 的进程；端口上若是未知进程，会拒绝接管，避免与 `pi-web-ui server install`、systemd 或 launchd 冲突。只有 descriptor 明确设置 `stop.allowUnowned=true` 才允许按端口接管。

## 端口

- 正式版网页服务：通常为 `8787`；
- 开发后端：`8788`；
- 开发前端 Vite：`5173`；
- 正式版 watchdog：`8790`；
- 开发版 watchdog：`8791`。

插件从 `PI_WEB_UI_WATCHDOG_PORT` 读取当前 watchdog 端口。

## 开发页面里的转发依赖

开发版页面对 `/plugins-api/...` 的请求能被转发到 `:8788`，是因为 Vite 只配了
`/api`、`/themes`、`/plugins`、`/ws` 四条代理，而 `/plugins-api` 恰好命中 `/plugins`
的前缀匹配。这是隐含依赖：插件客户端不要改用其他前缀的相对路径。

## 安装

```bash
pi-web-ui install xieweimo/pi-web-ui-contrib/plugins/reconnect
```

也可以手工拷贝到数据目录（
`<dataDir>/plugins/reconnect/`，默认 `~/.pi-web/plugins/reconnect/`），刷新页面即生效；
卸载只需删掉该目录。

## 配套 watchdog

插件本身**不含**守护进程，它只是探测 `127.0.0.1:<watchdogPort>` 与 descriptor 状态。
真正让「服务卡死也能重启」生效的是本目录下的单文件守护进程：

```bash
node plugins/reconnect/watchdog/pi-web-ui-recovery-watchdog.js --descriptor ./restart-descriptor.json
```

没起 watchdog 时插件会提示「重启守护未运行」，但状态显示与「刷新并重新连接」照常可用。
