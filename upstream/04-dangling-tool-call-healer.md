# [Bug] 中断的 assistant tool call 被 healer 追加为孤立 toolResult，重试后 provider 400

## 复现

1. 模型发起 tool call；
2. 本轮因 `error` 或 `aborted` 中断；
3. 点击重试，触发 `healDanglingToolCallFile()`；
4. 请求 OpenAI 或 DeepSeek。

## 实际结果

healer 把合成 `toolResult` 追加到文件尾。中断 assistant 的 toolCall 在适配层不会被送给 provider，却仍被 healer 当作悬空调用补结果，造成孤立输出：

- OpenAI：`No tool call found for function call output with call_id …`
- DeepSeek：`Messages with role 'tool' must be a response to a preceding message with 'tool_calls'`

## 期望

只修复真正会进入 provider 请求链的正常尾部 assistant tool call；`stopReason: "error" | "aborted"` 的调用不得补合成结果。

## 最小修复方向

在 `tailAssistantToolCallIds` 回溯中跳过 `error` / `aborted` assistant，再判断正常链尾。保持 append-only 和正常悬空调用的修复语义不变。

## 验收 / 回归用例

- error 中断后重试：不新增 toolResult，请求不 400；
- aborted 中断后重试：同上；
- 正常链尾遗留 toolCall：仍补齐；
- 历史分支遗留条目：不误补。

## 本地临时修复

`patches/patch-pi-web-ui-dangling-tool-calls.js`（`dangling-active-chain-filter-v2`）。上游修复发布并验证后删除该补丁。
