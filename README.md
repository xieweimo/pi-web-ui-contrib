# pi-web-ui：重连 / 重启服务（界面插件 + 可选 companion watchdog）

给 pi-web-ui 加一个「重新连接 / 重启服务」入口；并且在服务卡死或崩溃时，还能把它救回来。

两部分可以分开用，也可以一起用：

| 部分 | 是什么 |
| --- | --- |
| `reconnect-plugin/` | 界面插件：状态显示 +「刷新并重新连接」+「重启服务」（按 descriptor 为每个服务动态生成 重启/启动/停止） |
| `pi-web-ui-recovery-watchdog.js` | companion watchdog：独立 Node 进程，只监听 `127.0.0.1`，按登记的命令停/起服务 |

## 为什么需要外部 watchdog

插件跑在 pi-web-ui 服务进程里。服务卡死、崩溃，或者处在“进程还在但页面连不上”的状态时，插件自己也没了 —— 无法自救。
所以真正的恢复能力必须放在**服务之外**：一个只监听本机端口的小进程，由它停掉并重新拉起服务。

## 快速开始

### 1. 写一份 restart descriptor

```json
{
  "command": "npm",
  "args": ["run", "dev"],
  "cwd": "/path/to/pi-web-ui",
  "servicePort": 8788,
  "healthUrl": "http://localhost:5173/",
  "watchdogPort": 8790,
  "shell": true,
  "stop": { "mode": "process-tree", "portFallback": false, "allowUnowned": false }
}
```

完整字段与多服务写法见 `examples/restart-descriptor.example.json`。

**关键点：watchdog 不猜你是怎么启动的**，它只执行 descriptor 里登记的 `command`。
所以 `npm run dev`、`npm start`、直接跑 server、CLI 启动都能覆盖 —— 由启动方登记即可。

### 2. 起 watchdog

```bash
node pi-web-ui-recovery-watchdog.js --descriptor ./restart-descriptor.json
```

### 3. 调它（HTTP，仅本机）

```bash
curl http://127.0.0.1:8790/state                              # 状态：健康检查 / PID / 阶段 / 最近失败原因
curl -X POST http://127.0.0.1:8790/start                      # 启动全部服务
curl -X POST http://127.0.0.1:8790/restart                    # 重启全部服务
curl -X POST http://127.0.0.1:8790/stop                       # 停止全部服务
curl -X POST http://127.0.0.1:8790/services/web/restart       # 只重启某个服务
```

（`watchdogPort` 默认 8790。`npm run dev` 的后端固定用 8788，所以 8790 不会撞。）

## 多服务：可以只重启一层

descriptor 里可以登记多个服务，各自独立生命周期：

```json
{
  "watchdogPort": 8791,
  "primaryServiceId": "backend",
  "services": {
    "backend":  { "command": "node", "args": ["--import", "tsx", "server/index.ts"],
                  "servicePort": 8788, "healthUrl": "http://localhost:8788/api/health", "order": 10 },
    "frontend": { "command": "node", "args": ["node_modules/vite/bin/vite.js"],
                  "servicePort": 5173, "healthUrl": "http://localhost:5173/", "order": 20 }
  }
}
```

实测：只重启前端不动后端，或只重启后端不动前端（两边 PID 互不影响）。

## 与已有 supervisor 的边界

默认只操作「自己启动并且记录了 PID」的进程，不会去动别人的服务：

| 情况 | 行为 |
| --- | --- |
| 端口上的进程由本 watchdog 启动 | 正常重启 / 停止 |
| 端口上有别的进程（例如 `server install` / systemd / launchd 托管的） | **拒绝**，并提示它由哪个管理器托管 |
| 确实要接管（显式确认之后） | 带 `?force=1` 才接管；界面里是「接管并重启」按钮 + 二次确认弹窗 |

建议分工：`server install` / systemd / launchd / Docker 继续用平台自己的机制；
`npm run dev`、`npm start`、直接跑 server 这类**没有 supervisor** 的场景才用它。

## 界面插件

把 `reconnect-plugin/` 放进 `<dataDir>/plugins/reconnect/`（默认 `~/.pi-web/plugins/reconnect/`），刷新页面即可。

- 面板按运行环境切换文案：开发环境讲“哪一层、会不会影响 HMR”；普通环境只讲“什么时候点、会不会丢东西”
- 断连时插件客户端会**直连 watchdog**（`127.0.0.1:<watchdogPort>`），所以“页面连不上后端”时依然能重启服务
- 入口、状态、按钮全部走插件 API（`manifest.ui` + `host.route`），不改包内源码

## 实测记录（Windows）

- `npm run dev`：整棵进程树（npm → concurrently → node --watch / vite）停止并重启，恢复后前后端都健康
- 多服务：单服务重启不影响另一层（另一个的 PID 不变）
- 拒绝未授权接管：端口上放一个外部进程 → 不带 `force` 返回 409，带 `force` 才接管
- 停止：Windows 用 `taskkill /T`，并显式 `windowsHide`（不闪控制台）

## 未验证

进程树停止的 Linux/macOS 分支用的是进程组信号（`kill(-pid)`）：代码已写，但**没有在真机上验证过**，
需要 CI 或另一台机器跑一遍。

## 文件

```
reconnect-plugin/                         界面插件（manifest + 服务端路由 + 客户端视图）
pi-web-ui-recovery-watchdog.js            companion watchdog（单文件、零依赖，只用 Node 内置模块）
examples/restart-descriptor.example.json  单服务 / 多服务 descriptor 示例
```

## License

MIT（与 pi-web-ui 一致）
