# dsh-cursor-subscription v0.6.3

已经输出的 Cursor 回答不会再被取消重来。适配器现在收到 Cursor 的 `turn_ended`
就结束该步，而不再等 HTTP/2 流关闭——服务端并不总会关。

## 修复

### 已经流式输出的回答被丢弃，整步重跑

- **现象**：结论已经出现在会话里，约 60 秒后被取消，模型「重新分析」并给出新的
  结论，之后还可能再来一次。
- **影响范围**：截至 0.6.2 的所有构建。触发条件是服务端结束一轮却没有关闭 run，
  这个行为是间歇性的——同一个会话里大部分步正常，少数几步会卡住。2026-09-18
  在 `claude-opus-5-high` 上复现（插件 0.6.2，DSH 0.1.6-alpha.2）。
- **根因**：Cursor 用 `turnEnded` 交互更新（field 14）标记每一轮助手回答的结束，
  适配器解码了它却从未处理，于是读循环继续等流结束。当 Cursor 只用心跳维持
  run、不关闭流时，进度看门狗（`STREAM_PROGRESS_TIMEOUT_MS`，60 秒）会以
  `Cursor stream progress timeout: no content for 60000ms` 中止该 run，错误被归类为
  `TIMEOUT`；`TIMEOUT` 属于 DSH 的可重试集合，于是 DSH 丢弃已经流式输出的回答并
  重跑整步，直到重试次数用尽。
- **修法**：在 `interactionUpdate` 分支中，`turnEnded` 且没有待处理的 MCP 工具
  调用时直接结束读循环（`lib/index.js`），该步随即以干净的 `stop` 收尾。最后一版
  会话 checkpoint **总是先于** `turnEnded` 到达，因此持久化的会话状态不会过期；
  有待处理工具调用时仍等 checkpoint，保持 live bridge 可续跑。

## 证据

插件自身的会话日志（`llm/retry` 记录）里，一轮里有四步是「回答已完整输出又被丢弃」：

| 步 | 回答 | 流式时长 | 之后静默 | 结果 |
| --- | --- | --- | --- | --- |
| 14 | 1522 字符 | 19.8 s | 81.8 s | `TIMEOUT: Cursor stream progress timeout: no content for 60000ms` |
| 16 | 405 字符 | 6.1 s | 71.9 s | 同上 |
| 22 | 2446 字符 | 12.7 s | 71.9 s | 同上 |
| 26 | 1124 字符 | 14.9 s | 69.8 s | 同上 |

每次 `TIMEOUT` 之后紧跟一条 `llm/retry`（`retry: 1/5`），这正是用户看到的
「结论被取消、重新分析」。

## 验证

- **单元测试**：`tests/proto.test.mjs` 新增
  《Cursor adapter finishes the step when the server signals turn_ended》：
  回放 文本 → checkpoint → `turnEnded`，之后只发心跳，断言最终是 `stop`。
  修复前该用例以真实错误失败（`TIMEOUT: Cursor stream progress timeout: no content
  for 60ms`），修复后 72/72 通过。
- **真实链路**：一次针对真实 Agent API 的 A/B 探测（模型与出问题的会话相同）把
  「工具调用 → 续跑 → 文本回答」这条路径分别跑过修复前与修复后的适配器。两次运行的
  帧序列都以 `TXT … CP(589B) | UPD:unknown | CP(589B) | UPD:turnEnded` 结束，证明
  `turnEnded` 就是该轮的终止标记，且最新 checkpoint 先于它到达；修复后的适配器
  在此处就结束并返回 `finish/stop`，不再依赖 `ENDSTREAM` 帧是否到达。

该次探测运行恰好紧跟着收到了 `ENDSTREAM({})`，所以「服务端不关流」这一变体没有
在真实链路上复现；它由单元测试覆盖，而修复本身已经消除了对该帧的依赖。

## 其他变更

- **工具轮次上限默认值**：`maxToolRounds` 默认从 `64` 改为 `200`，可选范围不变
  （1–1000）。已在 **设置 → Cursor** 保存过该值的 profile 仍沿用已保存的值，只有
  全新配置的默认值发生变化。
- **测试**：共 72 个——协议 43、面板文案契约 8、通道挂载 6、版本号 6、
  native fetch 6、图片输入 3。

## 安装前必读：pnpm 的 24 小时发布年龄门限

pnpm 11 及以后内置了 24 小时的 `minimumReleaseAge` 供应链防护：形如 `^0.6.0`
的版本范围**永远不会选中最近 24 小时内发布的版本**。所以在刚发布完之后执行
`dsh plugin --profile web add dsh-cursor-subscription`，解析到的是「发布超过 24 小时的
最新版本」。解决方式二选一：安装时显式指定版本，或在 profile 的
`pnpm-workspace.yaml` 里按包名豁免：

```yaml
minimumReleaseAgeExclude:
  - dsh-cursor-subscription
```

按包名豁免只对这个包放行，其他依赖仍受该防护约束。完整的安装与验收流程见
[AGENTS.md](https://github.com/orrinzeng/dsh-cursor-subscription/blob/master/AGENTS.md)。

## 安装 / 升级

```sh
# 显式指定版本（不受 24 小时门限影响）
dsh plugin --profile web add dsh-cursor-subscription@0.6.3

# 或者先加上前面的按包名豁免，再用普通命令
dsh plugin --profile web add dsh-cursor-subscription
dsh plugin --profile web update dsh-cursor-subscription

dsh plugin --profile web list dsh-cursor-subscription --depth 0
dsh --profile web --dump-config
```

之后**手动重启 DSH**（`patchReload` 不会重载新增 bundle 与客户端模块），然后确认：

1. **设置 → Cursor** 能打开，标题旁显示 `v0.6.3`；
2. 模型选择器里出现 `cursor-subscription`，登录、用量与模型卡片照旧可用；
3. 以前会被取消的 Cursor 回答现在会保留下来，会话继续往下走，不再出现 `TIMEOUT` 重试。

## 兼容性与已知说明

- 没有 `turnEnded` 宣告的卡死仍然是卡死：进度看门狗照旧以可重试的 `TIMEOUT` 中止。
  只有服务端明确宣告结束的一轮才被当作已结束。
- Cursor 的 Agent 协议仍是未公开接口且会变化；聊天链路上的传输错误或
  `CURSOR_ERROR` 通常意味着需要更新插件。
