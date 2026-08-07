# Native Computer Use Integration Plan

**Status:** in progress
**Baseline:** `origin/main` at `7efe05623b687db9373191d323d58687ec383fb0`
**Tracking issue:** [omnigent-ai/omnigent#4327](https://github.com/omnigent-ai/omnigent/issues/4327)

## Progress

- **Step 0B (Codex discovery): complete.** Recorded fixtures and
  `docs/computer-use-protocol-discovery.md` define provisional start,
  authoritative completion, image, failure, and interruption behavior.
- **Step 0A (Claude discovery): deferred.** The available Claude login uses an
  API-key helper; built-in Computer Use requires an eligible claude.ai Pro/Max
  login. No production Claude parser will be written from guessed events.
- **Step 1 (artifact and conversation contract): complete.** Optional
  presentation/attachment metadata, hidden bounded frame storage, retention,
  ownership, migration/backfill, copy isolation, prompt/search isolation, and
  cleanup are implemented and covered by focused tests.
- **Steps 2–5:** pending.

## Summary

Surface computer-use activity from supported native harnesses in Omnigent without
building a second screen-control runtime.

Claude Code and Codex remain responsible for screenshots, app interaction,
permissions, clicks, typing, and their own safety controls. Omnigent preserves the
vendor tool lifecycle and selected screenshot frames, normalizes them into a
provider-neutral presentation contract, and renders them in a Computer workspace
panel. A later Electron enhancement may show the same latest frame in a detachable
picture-in-picture window.

The first release is an observability and control surface, not an automation
engine. Its fidelity is bounded by the public events and result content emitted by
each harness.

## User problem

Native Claude Code and Codex sessions can use vendor-supplied computer-use
capabilities, but Omnigent does not currently expose that activity as a first-class
surface:

- Claude-native tool results are mirrored as text and inline image blocks are
  intentionally replaced with placeholders.
- Codex-native `mcpToolCall` items are explicitly skipped because their event shape
  had not been verified when the forwarder was implemented.
- The web and Electron clients have no shared concept of a computer-use frame,
  current target app, or action state.
- Persisting raw base64 in conversation items would cause database and prompt
  growth, so simply forwarding the vendor payload is unsafe.

As a result, a user may know that an agent is busy but cannot see which app it is
working in or inspect the most recent visual state from Omnigent.

## Goals

1. Reuse the computer-use runtime provided by the user's installed and eligible
   Claude Code or Codex environment.
2. Preserve computer-use lifecycle events and screenshot results without storing
   inline base64 in conversation rows.
3. Give the web UI one provider-neutral model for current app, current action,
   provider, status, and latest frame.
4. Keep vendor permissions, approvals, interruption, and safety controls
   authoritative.
5. Make capability and setup failures understandable without silently changing a
   user's vendor configuration or Omnigent sandbox.
6. Keep normal session history, resume, sharing, and access-control behavior
   correct.

## Design principles

- **Vendor runtime owns execution.** Omnigent observes and presents vendor computer
  use; it does not synthesize clicks or keystrokes in the first implementation.
- **The native gate stays authoritative.** Omnigent never bypasses a Claude or Codex
  approval and does not silently enable a vendor MCP server or plugin.
- **Normalize presentation, not execution.** Provider-specific tool names and
  arguments remain intact for history and resume. Optional normalized metadata is
  only for the UI.
- **No raw base64 persistence.** Image bytes are decoded once, bounded, stored as
  session-scoped artifacts, and referenced by `file_id`.
- **No duplicate model context.** Presentation metadata and preview attachments are
  not added to reconstructed model input. The native harness already owns its
  computer-use context.
- **Public protocol only.** The integration may consume documented or publicly
  emitted harness events. It must not require a private desktop-only API or bundle
  modification.
- **Degrade safely.** If classification or preview extraction fails, the native
  harness keeps working and the transcript falls back to a generic tool card.

## Scope

### In scope for the first usable release

- macOS native Claude Code sessions that are eligible for Claude Code's built-in
  computer-use server.
- macOS native Codex sessions with a user-installed, enabled, and eligible Computer
  Use plugin.
- Safe discovery spikes to capture the exact vendor event and result shapes before
  production parsing is written.
- Generic Codex `mcpToolCall` forwarding, including started, progress, completed,
  and failed states.
- Claude tool-result image extraction before the existing history-sanitization
  step.
- A provider-neutral computer-use presentation contract.
- Session-scoped frame artifacts with ownership checks, bounds, retention, and
  normal-session cleanup.
- A Computer tab in the web workspace rail with latest-frame preview, current app,
  current action, provider, status, errors, and a Stop action using the existing
  session interrupt path.
- Auto-opening the Computer tab on the first computer-use action in a session.
- Reconnect and reload behavior using persisted metadata and artifact references.
- Setup and diagnostics for unsupported platforms, missing vendor capability,
  missing macOS permissions, and sandbox-related startup failures.
- Tests using recorded/synthetic protocol fixtures; CI does not control a real
  desktop.

### Follow-up within the broader project

- An optional Electron always-on-top picture-in-picture window that renders the
  same latest persisted frame as the workspace panel.
- Additional harness adapters when those harnesses expose a supported computer-use
  runtime and observable result protocol.

## Out of scope

- An Omnigent-owned screenshot, mouse, keyboard, or window-control engine.
- Redistributing the OpenAI or Anthropic computer-use runtimes or plugins.
- Enabling vendor computer-use settings, accepting app approvals, or weakening the
  sandbox on the user's behalf.
- Claiming live video, a fixed frame rate, or frame-by-frame parity with the vendor
  desktop UI. The first release updates when observable tool events contain a
  frame.
- Drawing a native border around the target window. Public tool events do not
  reliably expose window bounds; accurate highlighting would be a separate native
  macOS Accessibility/window-enumeration project.
- Replacing or suppressing vendor permission prompts.
- Sending clicks, text, or other interaction commands from the preview panel.
- OCR, screenshot search indexing, analytics collection, or automated redaction.
- Linux or Windows computer-use execution until a vendor runtime supports it and
  the integration is separately verified.
- Depending on undocumented private APIs from Codex Desktop or Claude Desktop.

## Current architecture and gaps

### Claude-native

`omnigent/claude_native_bridge.py` already converts Claude `tool_use` and
`tool_result` records into Omnigent `function_call` and `function_call_output`
items. `_tool_result_output` calls `_strip_inline_image_data`, which replaces
Anthropic base64 image blocks with text placeholders. This correctly prevents
large history rows, but it also means the UI cannot display computer-use frames.

The native launch passes an MCP config and settings without replacing the entire
vendor MCP configuration. The discovery spike must confirm that the built-in
computer-use server remains available, how its tools are named, and whether its
per-app approval requests are observable through Omnigent's existing permission
hook.

### Codex-native

`omnigent/codex_native_forwarder.py` mirrors known item types through
`_TOOL_ITEM_BUILDERS`, but intentionally omits `mcpToolCall`. The app-server event
contract supports a generic MCP tool lifecycle; the discovery spike must record
the installed version's exact started, progress, completed, and failure payloads
and confirm which image and app-context fields are actually emitted.

Computer Use should be classified using verified event metadata such as plugin,
server/tool, and app context while retaining generic MCP behavior for all other
plugins.

### Persistence

`FunctionCallOutputData` currently contains only `call_id` and a string `output`.
`StoredFile` and the `files` table are session-scoped but do not distinguish user
uploads from generated preview artifacts. Uploading every preview as an ordinary
file would clutter the Files panel.

### Web and Electron

The web block model also assumes text-only tool results. The right workspace rail
already has a strong precedent for conditional tabs and auto-surfacing behavior in
the embedded Browser integration. Electron already uses context isolation and
sender/origin validation for privileged browser IPC; a future preview window
should follow the same model.

### Sandbox

The generated `claude-native-ui` and `codex-native-ui` wrapper specs run as
`caller_process` with `sandbox.type: none`; Omnigent does not add a macOS Seatbelt
layer around those vendor-native terminals. The vendor runtime still applies its
own command sandbox and approval policy, while computer use is gated separately
by vendor per-app approval and macOS Accessibility/Screen Recording permissions.
Installed and enabled therefore does not imply usable, but discovery should not
attribute an OS-permission or vendor-sandbox failure to an Omnigent Seatbelt
profile that is not present.

## Proposed data contract

Add optional presentation-only fields to the existing function-call records rather
than introducing provider-specific conversation item types.

Illustrative wire shape:

```json
{
  "type": "function_call",
  "name": "vendor_tool_name",
  "arguments": "{...}",
  "call_id": "call_123",
  "presentation": {
    "kind": "computer_use",
    "provider": "claude",
    "app_name": "Safari",
    "app_id": "com.apple.Safari",
    "action_label": "Inspect page"
  }
}
```

```json
{
  "type": "function_call_output",
  "call_id": "call_123",
  "output": "sanitized textual result",
  "attachments": [
    {
      "kind": "computer_frame",
      "file_id": "file_123",
      "content_type": "image/png",
      "width": 1280,
      "height": 800
    }
  ]
}
```

Contract requirements:

- All new fields are optional so older clients and historical items continue to
  work.
- Provider values are an allowlisted enum; app/action strings are length-bounded
  untrusted display text.
- Prompt reconstruction continues to use only the original tool name, arguments,
  and sanitized text output.
- An output may reference zero or more frames, but the UI initially displays only
  the latest valid frame.
- Unknown attachment kinds are ignored by older/newer clients rather than causing
  item parsing to fail.
- A frame file is readable only through the owning session's existing access
  checks.

Add a file purpose or equivalent hidden artifact classification:

```text
user_upload                 normal Files-panel item
computer_use_frame          generated preview; hidden from normal file lists
```

Existing rows migrate to `user_upload`. Computer-use frames use a purpose-specific
bounded upload path so a native forwarder cannot accidentally create an unbounded
or unsupported file. The path accepts only supported image types, validates decoded
size, and does not emit a normal Files-panel resource entry.

## Execution plan

### Step 0: Maintainer alignment and protocol discovery

**Purpose:** Validate that the public vendor surfaces are sufficient before
committing Omnigent to a storage or UI contract.

#### 0A. Claude-native spike

- Use an eligible macOS machine with Claude Code authenticated through claude.ai.
- Enable the built-in computer-use server through Claude's normal interactive
  setup; do not edit it silently from Omnigent.
- Run one harmless, read-only inspection in a non-sensitive app.
- Capture sanitized `tool_use`, `tool_result`, permission, interruption, and error
  fixtures.
- Identify the exact tool/server classification fields and image-block locations.
- Determine whether per-app approval reaches Omnigent's `PermissionRequest` hook;
  retain the native TUI as the authoritative fallback.
- Record the generated native-wrapper sandbox and the vendor-reported command
  sandbox so later diagnostics identify the layer that actually blocked a call.

**Done when:**

- Sanitized fixtures cover start, text-only result, image result, failure, and
  interruption.
- The classification rule is fixture-backed rather than based on guessed names.
- Approval behavior is documented, including the authoritative fallback.
- The native-wrapper, vendor-sandbox, and macOS-permission boundaries are
  documented without proposing an unnecessary Omnigent sandbox exception.
- No private Claude Desktop API is required.

#### 0B. Codex-native spike

- Query the user's installed/enabled plugin state without redistributing the
  plugin.
- Start Codex app-server under Omnigent's session-specific `CODEX_HOME`.
- Run a harmless read-only application-state operation.
- Capture sanitized `item/started`, completed, failed, permission, and
  interruption fixtures for `mcpToolCall`; capture progress when emitted and
  explicitly document its absence otherwise.
- Record whether result content contains images and whether plugin/app context is
  available on each lifecycle edge.
- Record the generated native-wrapper sandbox and the vendor-reported command
  sandbox so later diagnostics identify the layer that actually blocked a call.

**Done when:**

- The installed app-server schema is represented by recorded fixtures for every
  lifecycle state Omnigent will support.
- Generic MCP mapping and Computer Use classification rules are documented.
- Frame availability and update cadence are known; no live-stream claim is made
  if only result screenshots are available.
- The plugin starts within Omnigent's private home and session lifecycle, or the
  blocker is documented and Codex delivery is deferred.
- No private Codex Desktop-only API is required.

#### Step 0 stop condition

If either provider requires bundle modification, redistribution, or a private
desktop-only API, stop that provider's implementation. Continue only with the
provider whose public native interface is sufficient; do not add an Omnigent
screen-control runner as an implicit fallback.

### Step 1: Artifact and conversation contract

**Likely files:**

- `omnigent/entities/conversation.py`
- `omnigent/entities/file.py`
- `omnigent/db/db_models.py`
- the repository's database migration path
- `omnigent/stores/file_store/__init__.py`
- `omnigent/stores/file_store/sqlalchemy_store.py`
- `omnigent/server/routes/sessions/routes_resources.py`
- `omnigent/server/routes/_sessions/helpers.py`
- `omnigent/runtime/prompt.py`
- matching backend tests

**Work:**

1. Add validated optional `presentation` metadata to function calls and optional
   attachment references to function-call outputs.
2. Add `purpose` (or a purpose-equivalent hidden artifact class) to stored file
   metadata and persistence.
3. Migrate existing rows to the normal user-upload purpose.
4. Add a purpose-specific internal upload path/helper for frame images with MIME,
   decoded-byte, dimension, and session-ownership validation.
5. Exclude computer-use frames from the normal Files-panel list while retaining
   owner-authorized direct reads.
6. Define configurable per-frame and per-session retention limits. Delete frame
   bytes and metadata together, and tolerate expired references in history.
7. Ensure session deletion cleans up frames through the existing file/artifact
   cleanup path.
8. Ensure prompt reconstruction and search indexing never include attachment
   bytes or presentation metadata.

**Done when:**

- Old conversation rows and old clients remain readable.
- A round trip through API, database, SSE, reload, and resume preserves metadata
  and frame references.
- Raw base64 never appears in conversation rows, search text, logs, or SSE payloads.
- Cross-session access to a frame returns the same authorization failure as other
  session-scoped files.
- Computer-use frames do not appear in the normal Files panel or user attachment
  picker.
- Retention defaults and cleanup behavior are documented and covered by tests.
- Reconstructing model input from a computer-use result produces only the original
  text result, not preview metadata or bytes.

### Step 2: Generic native-harness adapters

#### 2A. Claude adapter

**Likely files:**

- `omnigent/claude_native_bridge.py`
- Claude-native bridge/forwarder tests and recorded fixtures

**Work:**

1. Extract and validate eligible image blocks before
   `_strip_inline_image_data` replaces them.
2. Upload frame bytes through the bounded session-frame path.
3. Preserve the existing sanitized textual output plus attachment references.
4. Add presentation metadata only when the captured fixture-backed classifier
   identifies the built-in computer-use tool.
5. Keep non-computer image tool results and ordinary Claude tools behaviorally
   compatible unless separately specified.
6. Surface missing permissions, disabled server, unsupported account/platform,
   and sandbox failures as actionable status without changing vendor settings.

**Done when:**

- Every recorded Claude fixture maps deterministically to the expected generic
  function-call/output contract.
- Text plus image results retain useful text and exactly one artifact copy per
  frame.
- Duplicate transcript reads/resume do not duplicate persisted frames or tool
  items.
- Native approvals and interruption still work when the Computer panel is closed,
  disconnected, or unsupported.
- Ordinary Claude tool-result tests remain green.

#### 2B. Codex adapter

**Likely files:**

- `omnigent/codex_native_forwarder.py`
- Codex app-server forwarder tests and recorded fixtures

**Work:**

1. Add a generic `mcpToolCall` builder from the verified app-server fixtures.
2. Forward started, progress when emitted, completed, and failed lifecycle
   information without inventing missing fields. Use a terminal interrupted-turn
   edge to settle still-running calls when Codex emits no item completion.
3. Serialize standard MCP text/resource content safely for generic tool history.
4. Decode eligible image blocks into bounded frame artifacts.
5. Classify a started call provisionally using bounded, allowlisted
   `node_repl/js` plus `@oai/sky` arguments (or future explicit identity fields),
   then use `result._meta["codex/toolSurface"].kind == "computerUse"` as the
   authoritative completion identity. Retain generic MCP presentation for all
   other plugins and allow completion to correct the provisional classification.
6. Preserve error details in bounded display text and maintain current item dedup
   and response ordering.

**Done when:**

- Generic non-Computer Use MCP fixtures render as ordinary MCP/function tool calls.
- Computer Use fixtures receive normalized presentation metadata and artifact
  references.
- Started/optional-progress/completed/failed events preserve ordering across
  reconnect and resume without duplicate cards or frames; an interrupted turn
  cannot strand an in-progress card when no item completion arrives.
- Missing optional image/app-context fields degrade to a text-only tool result.
- Codex works only with a user-installed eligible plugin; no proprietary runtime is
  copied into Omnigent packages, builds, or tests.

### Step 3: Provider-neutral web UI

**Likely files:**

- `web/src/lib/conversationItems.ts`
- `web/src/lib/blocks.ts`
- `web/src/lib/itemsToBlocks.ts`
- `web/src/lib/blockStream.ts`
- `web/src/lib/events.ts`
- `web/src/lib/sse.ts`
- `web/src/components/ComputerUsePanel/*` (new)
- `web/src/components/SessionImage.tsx`
- `web/src/shell/railTabs.ts`
- `web/src/shell/WorkspacePanel.tsx`
- `web/src/shell/AppShell.tsx`
- colocated Vitest tests and `tests/e2e_ui/`

**Work:**

1. Extend history and live-stream parsing with optional presentation and attachment
   fields.
2. Derive a per-session computer-use view model from both hydrated history and live
   events: provider, app, action, status, latest valid frame, and error.
3. Add a Computer rail tab that becomes available after the first classified
   computer-use event.
4. Auto-open the rail and select Computer on the first running action, following
   the Browser tab's existing event-listener pattern. Do not repeatedly steal focus
   after the user changes tabs.
5. Render the latest frame through the session-authorized artifact path with useful
   loading, missing, expired, and error fallbacks.
6. Add a Stop action wired to the existing session interrupt behavior. The panel
   sends no mouse/keyboard input.
7. Make current app/action/status available to assistive technology and ensure the
   preview layout works at minimum rail width and in the mobile workspace surface.

**Done when:**

- The same synthetic event sequence produces the same panel state during live
  streaming and after full-page reload.
- The panel clearly distinguishes running, completed, interrupted, failed, and
  unavailable states.
- The first action auto-opens once; subsequent user tab choices are respected.
- Stop uses the existing interrupt path and the panel reflects the resulting
  terminal state.
- Missing, deleted, unsupported, or expired frames show a bounded fallback rather
  than a broken image or render crash.
- No provider-specific component is required to render Claude versus Codex.
- Colocated Vitest coverage and a synthetic Playwright happy path pass.

### Step 4: Capability and setup UX

**Work:**

1. Add a static computer-use capability declaration for supported native Claude
   and Codex harness specs.
2. Report dynamic runner eligibility separately: platform, harness version,
   server/plugin detected, enabled/disabled/unknown, required OS permissions, and
   sandbox outcome.
3. Present vendor-specific setup instructions and actionable failure reasons in a
   provider-neutral status surface.
4. Never represent static support as proof that the current machine/account can
   execute computer use.
5. Never silently enable a server/plugin, grant an app approval, change OS privacy
   settings, or set `sandbox.type: none`.

**Done when:**

- Unsupported platform, missing runtime, disabled capability, missing permission,
  and sandbox-blocked states are distinguishable in tests and UI.
- Setup copy links to official vendor documentation and does not expose internal
  paths or config.
- A capability probe is read-only and bounded; it cannot launch an interaction or
  mutate vendor state.
- Existing sessions without Computer Use retain their current UI and startup path.

### Step 5: Optional Electron picture-in-picture

This step begins only after the workspace panel is stable and maintainers agree
that the frame cadence is useful enough for a detached preview.

**Likely files:**

- `web/electron/src/main.js`
- `web/electron/src/preload.js`
- `web/electron/src/computerUsePreviewIpc.js` (new)
- `web/src/lib/nativeBridge.ts`
- Electron unit/integration tests

**Work:**

1. Add a single per-session always-on-top `BrowserWindow` that displays the same
   latest authorized artifact frame and status model as the rail panel.
2. Use context isolation, a narrow preload API, sender-origin validation, and no
   raw Node exposure.
3. Close or detach the window on session change, logout, server loss, or app quit.
4. Label the surface as latest-frame preview rather than live screen sharing.

**Done when:**

- PiP lifecycle and cleanup are deterministic across session switch, refresh,
  server restart, and app quit.
- Untrusted renderer origins cannot open, update, navigate, or read another
  session's preview.
- Only artifact identifiers and bounded presentation metadata cross IPC; raw
  base64 does not.
- The workspace panel remains the complete fallback when PiP is unavailable.

## Testing strategy

### Backend and adapters

- Recorded, sanitized Claude fixtures with text-only, image, mixed text/image,
  failure, interruption, and duplicate/resume cases.
- Recorded, sanitized Codex `mcpToolCall` fixtures for started, completed, failed,
  interrupted-without-item-completion, missing optional fields, and non-Computer
  Use MCP tools. Cover schema-defined progress with a synthetic fixture because
  the live Computer Use calls emitted none.
- Contract round-trip tests for presentation metadata and attachments.
- Assertions that persisted conversation JSON, search content, logs, and emitted
  SSE never contain frame base64.
- Artifact type/size/dimension bounds and malformed-data rejection.
- Session ownership and cross-session authorization tests.
- File-purpose filtering, retention, expiry fallback, and session cleanup tests.
- Prompt reconstruction tests proving metadata and attachments are not replayed.
- macOS manual capability matrix for native-wrapper sandbox, vendor command
  sandbox, vendor per-app approval, and OS Accessibility/Screen Recording state.

### Web

- Parsing tests in `conversationItems`, `itemsToBlocks`, `blockStream`, and `sse`.
- Reducer/view-model tests for ordering, dedup, reconnect, reload, provider switch,
  missing frames, and terminal states.
- `ComputerUsePanel` component tests for accessibility and responsive layout.
- `AppShell`/`WorkspacePanel` tests for tab availability, one-time auto-open, tab
  fallback, and Stop.
- Playwright happy path driven by synthetic server events and fixture artifacts.
  CI must not request macOS privacy permissions or control a real app.

### Electron

- IPC sender/origin validation.
- Window singleton, lifecycle, and cleanup.
- Cross-session artifact isolation.
- No arbitrary URL navigation and no raw IPC/Node exposure.

## Rollout and pull-request sequence

Keep changes reviewable and leave the tracking issue open until all agreed MVP
acceptance criteria are complete:

1. **Discovery evidence:** sanitized fixtures and a short design update confirming
   or revising this plan. No production behavior.
2. **Contract and artifacts:** optional conversation metadata, hidden frame
   artifacts, bounds, retention, and prompt isolation.
3. **Claude-native adapter:** fixture-backed extraction and classification.
4. **Codex-native adapter:** generic `mcpToolCall` forwarding plus Computer Use
   classification.
5. **Shared web UI:** Computer panel, auto-open, reload/reconnect, and Stop.
6. **Capability/setup UX:** eligibility and diagnostics.
7. **Electron PiP:** separate optional PR after panel feedback.

Each behavior-changing PR references the tracking issue using `Part of #...` until
the final MVP PR. UI PRs include the required screenshots/video, colocated Vitest
coverage, and Playwright happy path. All commits are DCO-signed.

## Security and privacy review checklist

- [ ] Only a user's installed vendor runtime performs computer control.
- [ ] No vendor binary/plugin is redistributed.
- [ ] No private API, bundle patch, or approval bypass is required.
- [ ] Frame type, dimensions, decoded bytes, and per-session retention are bounded.
- [ ] Frame bytes are absent from conversation rows, model prompts, logs, search,
      metrics, and IPC.
- [ ] Every artifact read is checked against the owning session and caller access.
- [ ] Shared-session behavior is explicit: users with session read access may see
      retained preview frames just as they see the session transcript.
- [ ] Normal session deletion and retention cleanup remove frame metadata and bytes.
- [ ] App names, action labels, error text, filenames, and MIME metadata are treated
      as untrusted input.
- [ ] The UI never implies that a missing preview means the native harness stopped.
- [ ] Sandbox changes, if any, are narrow and independently reviewed.
- [ ] Electron IPC is context-isolated, origin-validated, and session-scoped.

## Risks and mitigations

| Risk | Mitigation |
| --- | --- |
| Vendor event shapes change | Recorded fixtures, version-aware capability diagnostics, generic fallback cards |
| No public live-frame stream | Promise latest observable frame only; do not market live video |
| Screenshots contain sensitive information | Session ACLs, no indexing/telemetry, bounded retention, explicit sharing behavior |
| Image payload causes database/context growth | Artifact references only; base64 exclusion assertions at persistence and prompt boundaries |
| A permission or vendor-sandbox failure is misattributed | Report native-wrapper, vendor command-sandbox, per-app approval, and macOS permission state separately |
| Provider-specific logic leaks into UI | One normalized presentation contract and shared view model |
| Duplicate frames on resume/reconnect | Stable call/event identity plus content-aware/idempotent artifact persistence |
| Hidden frames clutter user files | Dedicated purpose and default list filtering |
| Proprietary runtime licensing | Integrate only with user-installed runtime; no vendoring, packaging, or fixture payloads containing proprietary code |

## Overall definition of done

The MVP is complete when all of the following are true:

- A supported Claude-native session and a supported Codex-native session can each
  perform a harmless vendor-controlled computer-use action while Omnigent displays
  provider, app/action state, lifecycle status, and the latest observable frame.
- The same state survives browser reload and SSE reconnect without duplicate tool
  cards or frame artifacts.
- Stop delegates through the existing Omnigent interrupt path; vendor approvals and
  safety controls remain authoritative.
- Ordinary non-computer tools and generic Codex MCP tools remain compatible.
- No raw frame base64 is stored in conversation history, sent through SSE/IPC, or
  replayed into model context.
- Frame artifacts are bounded, session-authorized, hidden from normal user files,
  retained according to a documented policy, and cleaned up with the session.
- Unsupported and misconfigured environments receive actionable diagnostics.
- Required backend, frontend, Playwright, and security tests pass.
- Public documentation accurately describes supported provider/platform
  combinations and calls the UI a latest-frame preview unless a real public stream
  is later available.

## Maintainer decisions requested

1. Is optional metadata on `function_call` / `function_call_output` preferred over a
   new display-only conversation item type?
2. Should hidden frames extend the existing `files` table with `purpose`, or use a
   separate generated-artifact record?
3. Is Claude-first the preferred adapter sequence, followed by generic Codex
   `mcpToolCall` support?
4. What default per-session frame-count/byte retention should the MVP use?
5. Should Electron PiP stay in the same tracking issue as a follow-up, or move to a
   separate issue after the rail panel ships?

## References

- [Claude Code computer use](https://code.claude.com/docs/en/computer-use)
- [OpenAI Computer Use](https://learn.chatgpt.com/docs/computer-use)
- [Omnigent contribution guide](../CONTRIBUTING.md)
- [Base64 image persistence issue](https://github.com/omnigent-ai/omnigent/issues/4310)
