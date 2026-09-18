# dsh-cursor-subscription v0.6.3

A finished Cursor answer no longer gets cancelled and re-analysed. The adapter
now ends the step on Cursor's `turn_ended` update instead of waiting for the
HTTP/2 stream to close, which the server does not reliably do.

## Fixed

### An answer that had already streamed was discarded and the step re-ran

- **Symptom:** the conclusion appears in the conversation, then ~60 s later it
  is cancelled, the model "thinks again" and produces a new conclusion, and the
  same thing can repeat for several more steps.
- **Affected:** every build up to and including 0.6.2. It needs the server to
  end a turn without closing the run, which is intermittent — the same session
  can complete most steps normally and hang on a few. Observed on
  2026-09-18 with `claude-opus-5-high` (plugin 0.6.2, DSH 0.1.6-alpha.2).
- **Cause:** Cursor terminates each assistant turn with the `turnEnded`
  interaction update (field 14). The adapter decoded it but never acted on it,
  so the read loop kept waiting for the stream to end. When Cursor keeps the run
  alive with heartbeats instead of closing it, the progress watchdog
  (`STREAM_PROGRESS_TIMEOUT_MS`, 60 s) aborts the run with
  `Cursor stream progress timeout: no content for 60000ms`, classified as
  `TIMEOUT`. `TIMEOUT` is in DSH's retryable set, so DSH discarded the answer it
  had already streamed and re-ran the whole step — up to its retry limit.
- **Fix:** `turnEnded` now ends the read loop (`lib/index.js`, the
  `interactionUpdate` branch) when no MCP tool call is pending, so the step
  finishes with a clean `stop` the moment the turn is over. The final
  conversation checkpoint always arrives *before* `turnEnded`, so the persisted
  session state stays current; a pending MCP tool call still waits for its
  checkpoint, keeping the live bridge resumable.

## Evidence

The plugin's own session log for a Cursor turn (`llm/retry` records) shows four
steps where a full answer was streamed and then thrown away:

| Step | Answer | Streamed for | Silent for | Result |
| --- | --- | --- | --- | --- |
| 14 | 1522 chars | 19.8 s | 81.8 s | `TIMEOUT: Cursor stream progress timeout: no content for 60000ms` |
| 16 | 405 chars | 6.1 s | 71.9 s | same |
| 22 | 2446 chars | 12.7 s | 71.9 s | same |
| 26 | 1124 chars | 14.9 s | 69.8 s | same |

Each `TIMEOUT` was immediately followed by `llm/retry` (`retry: 1/5`), which is
exactly the "conclusion cancelled, analysed again" behaviour.

## Verification

- **Unit:** `tests/proto.test.mjs` gained
  *"Cursor adapter finishes the step when the server signals turn_ended"*. It
  replays text → checkpoint → `turnEnded` and then only heartbeats, and asserts
  a `stop` finish. Before the fix it fails with the real error
  (`TIMEOUT: Cursor stream progress timeout: no content for 60ms`); after it,
  72/72 tests pass.
- **Live:** an A/B probe against the real Agent API (same model as the affected
  session) drove the tool-call → resume → text-answer path through both the
  pre-fix adapter and the fixed one. Both runs' frame order ends with
  `TXT … CP(589B) | UPD:unknown | CP(589B) | UPD:turnEnded`, confirming that
  `turnEnded` is the turn terminator and that the newest checkpoint precedes it;
  the fixed adapter stops there and reports `finish/stop`, no longer depending on
  an `ENDSTREAM` frame arriving.

The probe run itself received an `ENDSTREAM({})` right after `turnEnded`, so the
silent variant was not reproduced live; it is covered by the unit test, and the
fix removes the dependency on that frame either way.

## Changed

- **Default tool-round cap:** `maxToolRounds` now defaults to `200` instead of
  `64`; the accepted range is unchanged (1–1000). A profile that already saved a
  value under **Settings → Cursor** keeps its stored value — only the default
  for a fresh section moves.
- **Tests:** 72 in total — 43 agent-protocol, 8 panel-language contract,
  6 channel mount, 6 version label, 6 native fetch, 3 image input.

## Before installing: pnpm's 24-hour release-age gate

pnpm 11+ ships a built-in `minimumReleaseAge` of 24 hours, so a range such as
`^0.6.0` never selects a version published within the last day. Right after a
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
dsh plugin --profile web add dsh-cursor-subscription@0.6.3

# or, once the release-age exemption above is in place
dsh plugin --profile web add dsh-cursor-subscription
dsh plugin --profile web update dsh-cursor-subscription

dsh plugin --profile web list dsh-cursor-subscription --depth 0
dsh --profile web --dump-config
```

Restart DSH manually afterwards (`patchReload` does not pick up new bundles or
client modules), then check:

1. **Settings → Cursor** loads, with `v0.6.3` next to the title.
2. `cursor-subscription` appears in the model picker, and sign-in, usage and
   model cards work as before.
3. A Cursor answer that used to be cancelled after a while now stays, and the
   conversation moves on without a `TIMEOUT` retry.

## Compatibility and known notes

- A stall that is *not* announced by `turnEnded` is still a stall: the progress
  watchdog keeps aborting it with a retryable `TIMEOUT`. Only a turn the server
  declared finished is treated as finished.
- Cursor's Agent protocol remains undocumented and changes over time; a
  transport or `CURSOR_ERROR` failure on the chat path is expected to need a
  plugin update.
