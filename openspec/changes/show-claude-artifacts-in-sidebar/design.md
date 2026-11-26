# Design

## Context

See `proposal.md` for the motivation and `specs/thread-artifacts/spec.md` for the requirements.

Threads already carry two lists of linked items, pull requests and issues. Both follow the same path:

- **Contracts:** an optional array on `OrchestrationV2AppThread` and `OrchestrationV2ThreadShell` (`packages/contracts/src/orchestrationV2.ts:511-515`, `:2421-2425`).
- **Commands and events:** link and unlink commands (`thread.issue.link/unlink`) and a `thread.issue-synced` event that carries the thread.
- **Server:** the Orchestrator applies the commands (`Orchestrator.ts:5386`). `ProjectionStore.ts` persists the list in the thread's `payload_json` and copies it onto the shell (`:1960-1967`, `:2281-2288`).
- **Client runtime:** `presentThreadShell` defaults the list to `[]` (`packages/client-runtime/src/state/models.ts:317-320`).
- **Web sidebar:** `SidebarThreadRow.tsx:1165-1200` renders a `ThreadLinkBadge` whose hover card lists the rows.

PR links are also detected automatically. `PullRequestSyncReactor.ts:844-873` listens to `turn-item.updated`, reads PR URLs from finished command output, and dispatches `thread.pull-request.link`.

Claude Code's `Artifact` tool is not specially classified. It becomes a `dynamic_tool` turn item that carries `toolName`, `input` and the full `output` on the server (`ClaudeAdapterV2.ts:7718-7731`). `WireProjection.ts:190-198` compacts that output before it reaches clients, so clients cannot recover the URL themselves.

## Goals / Non-Goals

**Goals:**

- Reuse the linked-issues path end to end, so artifacts add a third list, not a new mechanism.
- Detect publishes on the server, from the full tool result, without changing any provider adapter.

**Non-Goals:**

- A dedicated artifact viewer or iframe embed. Artifacts open in the existing browser panel, as Decision 4 describes.
- A global, cross-thread artifacts view, a command palette entry, or an MCP tool for linking artifacts by hand.
- Fetching titles or thumbnails from claude.ai.

## Decisions

### 1. Detect publishes in a reactor, not in the Claude adapter

Add `ThreadArtifactReactor`, modeled on `PullRequestSyncReactor`. It handles `turn-item.updated` events whose item is a `dynamic_tool` named `Artifact` with a completed status. It parses the input's `action` (a missing action means `publish`) and the claude.ai artifact URL in the output text. It then calls `ThreadArtifactService.link` or `.unlink`, which dispatch the commands.

- **Why:** the adapter stays a translation layer ("complexity at the adapter boundary" means wire quirks, not product reactions). Event handling is ordered per thread, like PR linking. Any future provider that emits an `Artifact` dynamic tool works with no further changes.
- **Alternative considered:** extract the URL in `buildToolCallArtifacts` and add a new turn item field. That changes a hot adapter path and the wire schema for every tool call, so it was rejected.
- **Alternative considered:** parse URLs on the client. The client only sees compacted output, so this is not possible.

The URL pattern is `https://claude.ai/(code/)?artifact/[A-Za-z0-9-]+`. It is canonicalized by stripping the query string and hash, and the canonical form is the dedupe key. For a delete, the URL comes from the input's `url`.

### 2. Contract shape

`packages/contracts/src/threadArtifact.ts`:

```ts
ThreadArtifactLink = Schema.Struct({
  url: TrimmedNonEmptyString, // canonical claude.ai artifact URL, the identity
  title: TrimmedNonEmptyString,
  publishedAt: IsoDateTime,
});
```

- `artifacts: Schema.optional(Schema.Array(ThreadArtifactLink))` goes on both the thread and the shell, with the same "older servers still decode" comment that `issues` has.
- Commands: `thread.artifact.link` (upsert by URL) and `thread.artifact.unlink`.
- Event: `thread.artifact-synced`, with the same thread payload as `thread.issue-synced`.
- The Orchestrator owns the upsert, the newest-first ordering and the 20-entry cap, so every write path gets them.
- **Why no `source` field:** issues need one because both users and agents link them. Here only the agent links and the user only removes, so the field would carry no information. Add it if manual linking ever arrives.

### 3. Sidebar UI reuses `ThreadLinkBadge`

`SidebarThreadRow` gets an `artifactBadge` next to `issueBadge`. It renders `ThreadLinkBadge` with an artifact icon and the count, and its hover card holds a new `ThreadArtifactRows` component. Each row shows the title and the published time, opens on click, and has a remove (×) button that sends `thread.artifact.unlink` through `createEnvironmentRpcCommand`.

The remove action is gated with `permissionAtom` and declared in `CLIENT_GUARDED_RPC_SCOPES`, matching issue unlinking. The design adds no animation.

### 4. Open through `useOpenLink`, so artifacts can show in the side browser panel

On web and desktop, rows open artifacts with `useOpenLink(threadRef)` (`apps/web/src/browser/useOpenLink.ts:23`), the same path PR links and chat links use. `resolveLinkTarget` (`browserLinkTarget.ts:25`) makes the decision:

- **Side panel:** the artifact opens as a tab in the thread's browser panel, through `openUrlInPreview`, when the user's `browserLinkTarget` setting is `"app"` and `canOpenLinksInApp` holds (the preview scope, and the panel being available for the environment).
- **System browser:** used when the setting is `"system"`, on Cmd-click, Ctrl-click or middle-click, when the panel is unavailable, and as the fallback if opening the panel fails (`api.shell.openExternal`).

**Signing in.** Artifacts are private by default, so the panel's first visit can show claude.ai's sign-in page.

- **Desktop:** the panel runs with persistent browser profiles. The user signs in once, or imports cookies from another browser (`docs/user/browser-import.md`), and the session sticks.
- **Web and remote environments:** the tab runs in the environment's server browser and streams to the client. Sign-in works the same way but lives on that machine.

**Why:** artifacts show beside the chat with no new viewer code, and they behave exactly like every other link in T3 Code, including the user's existing preference.

**Alternatives considered:**

- _Always open in the system browser._ This avoids the first sign-in, but it ignores a preference the user already set and loses the side-by-side view. Rejected.
- _Always open in the panel, whatever the setting._ Special-casing one link type is surprising. Rejected.
- _Embed the artifact page in an iframe in the thread._ claude.ai controls whether its pages can be framed, and this would need new viewer code. Out of scope.

### 5. Mobile

The thread list row (`apps/mobile/src/features/threads/thread-list-v2-items.tsx`) shows the artifact count next to the PR indicator. Thread detail adds a compact "Artifacts" list that opens with `Linking.openURL` and offers remove through the same RPC.

### 6. Providers

The Claude Code adapter works through Decision 1. Codex, Cursor, Grok, OpenCode, Antigravity, Kiro, Pi, Muse Code and fx have no claude.ai Artifact tool, so they are explicitly not supported. They need no code changes, and their rows simply show no badge.

## Risks / Trade-offs

- **[Risk]** The `Artifact` tool's input and output shape could change in future Claude Code releases. → Matching relies only on the tool name, the `action` and `url` input fields, and a URL regex over the output text. Unit tests use recorded tool results, and any mismatch fails silently to "no badge", never to an error.
- **[Risk]** Adding a shell field grows the wire payload. → The list is capped at 20 small entries, is omitted when empty, and `transferBudgetV2.integration.test.ts` gets an artifact case. Run `vp run test:perf:v2-wire`.
- **[Risk]** Retained events must decode after upgrade or downgrade. → The new fields are optional, and the new event is ignored by older consumers, as `thread.issue-synced` was.
- **[Trade-off]** The first artifact opened in the browser panel can land on claude.ai's sign-in page. → The user signs in once per browser profile, or imports cookies on desktop. Cmd-click or Ctrl-click still opens the system browser, which is already signed in.
- **[Trade-off]** Titles come only from the tool input or the file name, which can be terse. This was accepted to avoid fetching claude.ai from the server.

## Migration Plan

No data migration. Existing threads have no `artifacts` field and decode as empty. A rollback leaves the stored field unread on older servers.
