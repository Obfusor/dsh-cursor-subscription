# dsh-cursor-subscription v0.6.2

Parallel Cursor MCP tool calls now resume cleanly instead of failing the turn
with `Tool result not provided`, and three shipped features that a bad merge
reverted on `master` are back.

**No published version ever contained the revert.** 0.6.1 on npm is unaffected;
the regression existed only in git `master` between the merge of PR #3
(`ec5e6af`) and this release.

## Fixed

### Parallel MCP tool calls no longer produce phantom results

- **Affected:** any 0.6.x build containing the PR #3 merge. When Cursor issued
  more than one MCP tool call in a single turn, tool continuation failed with
  `Tool result not provided`, or a tool call appeared with no result of its own.
- **Cause:** three defects in the same code path.
  1. Every MCP exec reused one fixed DSH block index (`TOOL_BLOCK_INDEX`), so
     the agent loop kept only a single tool-call block while the bridge still
     waited for every sibling call.
  2. Incomplete `partialToolCall` frames were emitted as tool-call blocks before
     the call had an id or name, creating nameless blocks that competed with the
     real `mcpArgs` frames.
  3. Empty or placeholder MCP execs were accepted as pending bridge entries, so
     the resume path kept waiting on calls that could never complete.
- **Fix:** partial tool calls are now buffered and never emitted on their own;
  each valid call gets its own block index (`TOOL_BLOCK_INDEX + pendingExecs.length`)
  so parallel calls stay distinct; an empty or unknown MCP exec is answered
  immediately with an `McpError` instead of becoming a pending entry; and the
  tool name is resolved against the run's text tools with the arguments
  normalized before the block is emitted.

### Features restored after the PR #3 merge reverted them

PR #3 was authored from a tree older than 0.5.8, so merging it deleted later
work. v0.6.2 restores all of it while keeping the MCP fix:

- **The account-channel mount fix is back.** The channel is mounted again
  through a registry owner Context that declares `webServer`, with
  `connection.rpc.handle` kept as a fallback for a Connection plugin that
  declares `webServer` itself. Without this, DSH 0.1.5-rc.1 fails the entire
  plugin tree with
  `cannot get property "webServer" without inject`. If you build from `master`
  at `ec5e6af`, this is the failure you would hit.
- **The panel's version label is back.** The `version` endpoint and the exported
  `VERSION` were deleted while `lib/client.js` kept requesting them, which
  silently dropped the `v0.6.x` chip next to the **Settings → Cursor** title.
- **Image input is back.** `prepareCursorImages`, the durable-attachment
  resolver, the `image` input modality, and the conversation-history image
  frames were deleted, so every model advertised text only.

## Changed

- **Tests:** 71 in total — 42 agent-protocol, 8 panel-language contract,
  6 channel mount, 6 version label, 6 native fetch, 3 image input. The
  agent-protocol suite gained coverage for the parallel MCP path: valid calls
  get distinct block indexes, empty execs are rejected, and no phantom call
  survives the resume.
- **Provenance:** the MCP fix comes from PR #3 by
  [@dengzhilong](https://github.com/452926826); the reverts that merge carried
  are undone in `adde4a5`.

## Before installing: pnpm's 24-hour release-age gate

pnpm 11+ ships a built-in `minimumReleaseAge` of 24 hours, so a range such as
`^0.5.0` never selects a version published within the last day. Right after a
release, a plain `dsh plugin --profile web add dsh-cursor-subscription` resolves
to the newest release *older than 24 hours*. Either request the exact version,
or exempt the package by name in the profile's `pnpm-workspace.yaml`:

```yaml
minimumReleaseAgeExclude:
  - dsh-cursor-subscription
```

The exemption keeps the gate active for every other package. See
[AGENTS.md](https://github.com/orrinzeng/dsh-cursor-subscription/blob/master/AGENTS.md)
for the full install and verification procedure.

## Install or upgrade

```sh
# exact version (works regardless of the 24-hour gate)
dsh plugin --profile web add dsh-cursor-subscription@0.6.2

# or, once the release-age exemption above is in place
dsh plugin --profile web add dsh-cursor-subscription
dsh plugin --profile web update dsh-cursor-subscription

dsh plugin --profile web list dsh-cursor-subscription --depth 0
dsh --profile web --dump-config
```

Restart DSH manually afterwards (`patchReload` does not pick up new bundles or
client modules), then check:

1. **Settings → Cursor** loads, with `v0.6.2` next to the title.
2. `cursor-subscription` appears in the model picker, and sign-in, usage and
   model cards work as before.
3. A turn that triggers several MCP tool calls at once completes without
   `Tool result not provided`.

## Compatibility and known notes

- Restoring the channel mount matters on DSH 0.1.5-rc.1, where
  `@deepseek-ai/dsh-client-connection` declares `credentials` alone. On
  0.1.0-rc.6 the `authority: "loopback"` registry option still exists.
- The account channel reuses Connection's envelope, trust fence, and browser
  authentication, so the browser half needed no change.
- Image input requires the durable attachment service; without it the request
  fails as `UNSUPPORTED_CONTENT` rather than silently dropping the image.
- Cursor's Agent protocol remains undocumented and changes over time; a
  transport or `CURSOR_ERROR` failure on the chat path is expected to need a
  plugin update.
