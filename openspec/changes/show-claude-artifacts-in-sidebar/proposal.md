# Proposal

## Why

When Claude Code publishes a claude.ai Artifact, such as a report, dashboard or diagram, its link appears only inside the tool-call row in the chat timeline. That row is collapsed, its output is compacted on the wire, and it scrolls away as the turn continues. After the turn ends, finding the page means scrolling back through the thread. Threads already show linked pull requests and issues on their sidebar rows, so artifacts belong in the same place.

## What Changes

- The server detects successful Claude Code `Artifact` publishes from the tool result. Each one is recorded as a link on its thread, holding the URL, a title and the publish time. Republishing to the same URL updates the existing entry and does not add a duplicate. An `Artifact` delete action removes the entry.
- Threads carry an optional, bounded list of artifact links. It sits on the full thread and on the sidebar thread shell, and it is persisted with the thread like linked issues. Older servers and clients still decode.
- The web and desktop sidebar thread row shows an artifact badge with a count. Its hover card lists each artifact by title. Clicking an entry opens the artifact, and each entry has a way to remove it from the thread.
- Mobile thread rows show the same artifact count and open the artifact from the thread.
- New user-facing strings are translated.

## Capabilities

### New Capabilities

- `thread-artifacts`: claude.ai Artifacts an agent publishes are recorded on their thread and shown, opened and removed from the thread's sidebar row.

### Modified Capabilities

None. No specs exist yet under `openspec/specs/`.

## Impact

- **Contracts** (`packages/contracts`): a new `ThreadArtifactLink` schema, an optional `artifacts` field on `OrchestrationV2AppThread` and `OrchestrationV2ThreadShell`, link and unlink commands, and a sync event.
- **Server** (`apps/server/src/orchestration-v2`): an artifact link service, a reactor on `turn-item.updated` that detects publishes, Orchestrator command handling, and the projection and shell copy in `ProjectionStore.ts`.
- **Client runtime** (`packages/client-runtime/src/state/models.ts`): the shell exposes `artifacts`, defaulting to `[]`.
- **Web** (`apps/web/src/components/sidebar/SidebarThreadRow.tsx` and nearby components): the badge, the hover-card list, and the open and remove actions.
- **Mobile** (`apps/mobile/src/features/threads`): the row count and the open action.
- **i18n**: new strings in `apps/{web,mobile}/src/i18n/translations`.
- **Performance**: the shell gains one small, capped array. The wire budget test in `apps/server/integration/transferBudgetV2.integration.test.ts` needs an update.
- **Providers**: only the Claude Code adapter produces artifacts today. Other providers are unaffected and show nothing.
