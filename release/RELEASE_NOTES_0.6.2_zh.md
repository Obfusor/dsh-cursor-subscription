# dsh-cursor-subscription v0.6.2

并行的 Cursor MCP 工具调用现在能正常续跑，不再以 `Tool result not provided`
结束整轮；同时，一次错误合并从 `master` 上抹掉的三项已发布功能全部回归。

**没有任何已发布版本受影响。** npm 上的 0.6.1 是好的；那次回退只存在于 git
`master` 上 PR #3 合并（`ec5e6af`）之后到本版本之前的区间里。

## 修复

### 并行 MCP 工具调用不再产生幽灵结果

- **影响范围**：任何包含 PR #3 合并的 0.6.x 构建。当 Cursor 在一轮里发出多个
  MCP 工具调用时，工具续跑会以 `Tool result not provided` 失败，或者出现一个
  没有对应结果的工具调用。
- **根因**：同一条代码路径上的三个缺陷。
  1. 所有 MCP exec 共用一个固定的 DSH block index（`TOOL_BLOCK_INDEX`），于是
     agent loop 只保留了一个工具调用块，而 bridge 仍在等待每一个兄弟调用。
  2. 尚未完整的 `partialToolCall` 帧在拿到 id 与名字之前就被当成工具调用块发出，
     产生了与真正的 `mcpArgs` 帧竞争的匿名块。
  3. 空的/占位的 MCP exec 会被登记为 pending bridge 条目，于是续跑路径一直等着
     永远不可能完成的调用。
- **修法**：partial 工具调用只缓冲、不再单独发出；每个有效调用拿到自己的
  block index（`TOOL_BLOCK_INDEX + pendingExecs.length`），并行调用之间不再互相
  覆盖；空或未知的 MCP exec 立刻回一个 `McpError`，而不是变成 pending 条目；
  工具名先在本轮可用的文本工具里解析、参数归一化后再发出该块。

### 恢复被 PR #3 合并回退的功能

PR #3 是从早于 0.5.8 的树上做出来的，合并它等于删掉了后续的工作。v0.6.2 把这些
全部恢复，同时保留 MCP 修复：

- **账户通道挂载修复回归。** 通道重新通过一个**声明了 `webServer` 的注册 owner
  Context** 挂载，并对自行声明 `webServer` 的 Connection 版本保留
  `connection.rpc.handle` 回退。缺了它，DSH 0.1.5-rc.1 会让整棵插件树加载失败：
  `cannot get property "webServer" without inject`。如果你从 `ec5e6af` 的
  `master` 构建，命中的就是这个错误。
- **面板版本号回归。** `version` 端点与导出的 `VERSION` 被删除，而
  `lib/client.js` 仍在请求它们，于是 **设置 → Cursor** 标题旁的 `v0.6.x`
  徽标静默消失。
- **图片输入回归。** `prepareCursorImages`、持久化附件解析器、`image` 输入模态
  以及会话历史里的图片帧都被删除，导致所有模型只声明文本能力。

## 其他变更

- **测试**：共 71 个——协议 42、面板文案契约 8、通道挂载 6、版本号 6、
  native fetch 6、图片输入 3。协议套件新增了并行 MCP 路径的覆盖：有效调用拿到
  互不相同的 block index、空 exec 被拒绝、续跑后不残留幽灵调用。
- **来源**：MCP 修复来自 [@dengzhilong](https://github.com/452926826) 的 PR #3；
  该合并带进来的回退在 `adde4a5` 中撤销。

## 安装前必读：pnpm 的 24 小时发布年龄门限

pnpm 11 及以后内置了 24 小时的 `minimumReleaseAge` 供应链防护：形如 `^0.5.0`
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
dsh plugin --profile web add dsh-cursor-subscription@0.6.2

# 或者先加上前面的按包名豁免，再用普通命令
dsh plugin --profile web add dsh-cursor-subscription
dsh plugin --profile web update dsh-cursor-subscription

dsh plugin --profile web list dsh-cursor-subscription --depth 0
dsh --profile web --dump-config
```

之后**手动重启 DSH**（`patchReload` 不会重载新增 bundle 与客户端模块），然后确认：

1. **设置 → Cursor** 能打开，标题旁显示 `v0.6.2`；
2. 模型选择器里出现 `cursor-subscription`，登录、用量与模型卡片照旧可用；
3. 一轮里触发多个 MCP 工具调用时能正常完成，不再报 `Tool result not provided`。

## 兼容性与已知说明

- 恢复通道挂载在 DSH 0.1.5-rc.1 上尤为关键：该版本的
  `@deepseek-ai/dsh-client-connection` 只声明 `credentials`。在 0.1.0-rc.6 上，
  注册选项 `authority: "loopback"` 仍然存在。
- 账户通道复用 Connection 的信封、信任栅栏与浏览器鉴权，因此浏览器侧无需改动。
- 图片输入依赖持久化附件服务；没有该服务时请求会以 `UNSUPPORTED_CONTENT`
  失败，而不是静默丢弃图片。
- Cursor 的 Agent 协议仍是未公开接口且会变化；聊天链路上的传输错误或
  `CURSOR_ERROR` 通常意味着需要更新插件。
