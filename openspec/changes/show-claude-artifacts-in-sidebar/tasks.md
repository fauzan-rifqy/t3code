# Tasks

## 1. Contracts

- [ ] 1.1 Add `ThreadArtifactLink` in `packages/contracts/src/threadArtifact.ts` and export it. Verify with a schema test that decodes a valid link and rejects an empty URL or title.
- [ ] 1.2 Add the optional `artifacts` array to `OrchestrationV2AppThread` and `OrchestrationV2ThreadShell`. Add the `thread.artifact.link` and `thread.artifact.unlink` commands and the `thread.artifact-synced` event. Verify that `vp run typecheck` passes and that a decode test shows a thread without `artifacts` still decodes.

## 2. Server: commands and projection

- [ ] 2.1 Handle link (upsert by canonical URL, newest first, capped at 20) and unlink in `Orchestrator.ts`, and map both to `thread.artifact-synced`. Verify with Orchestrator tests covering insert, republish update, the 21st-entry eviction and unlink.
- [ ] 2.2 Reduce the event in `ProjectionStore.ts`, persist it in `payload_json`, and copy `artifacts` onto the shell in `threadShellFromProjection` and the incremental shell path. Verify with a ProjectionStore test that the shell reflects link and unlink after a reload.
- [ ] 2.3 Add a `ThreadArtifactService` with `link` and `unlink` (`Effect.fn("ThreadArtifactService.link")` and so on), and expose the unlink RPC under `CLIENT_GUARDED_RPC_SCOPES`. Verify with a service test and the RPC guard test.

## 3. Server: publish detection

- [ ] 3.1 Add a pure parser that takes an `Artifact` `dynamic_tool` item and returns `{ kind: "publish", url, title } | { kind: "delete", url } | null`, implementing the title fallback, URL canonicalization and the ignored actions from the spec. Verify with unit tests built from recorded Claude `Artifact` tool results (publish, republish, delete, read, failed call, URL in a non-Artifact tool).
- [ ] 3.2 Add `ThreadArtifactReactor` on `turn-item.updated`, ordered per thread, that calls the service only for completed items. Wire it into the server layer next to `PullRequestSyncReactor`. Verify with a reactor test that drains events and asserts the thread's artifacts.
- [ ] 3.3 Add a Claude replay fixture with an `Artifact` publish (`orchestration-v2/testkit/ProviderReplayFixtures.ts`). Verify that the replay harness test ends with one artifact on the thread.

## 4. Wire budget

- [ ] 4.1 Add an artifacts case to `apps/server/integration/transferBudgetV2.integration.test.ts`. Verify that `vp run test:perf:v2-wire` passes within budget.

## 5. Client runtime

- [ ] 5.1 Add `artifacts` to the shell model in `packages/client-runtime/src/state/models.ts`, defaulting to `[]` in `presentThreadShell`, and add an `unlinkThreadArtifact` command with `createEnvironmentRpcCommand`. Verify with a models test for the default and a command test.

## 6. Web and desktop sidebar

- [ ] 6.1 Add a `ThreadArtifactRows` component (title, relative publish time, open, remove) and an `artifactBadge` in `SidebarThreadRow.tsx` using `ThreadLinkBadge`, with remove gated by `permissionAtom`. Open with `api.shell.openExternal` on desktop and `window.open` on web. Verify with a browser test (`await using screen = await render(...)`) that covers the badge count, the hover list, opening and removing.
- [ ] 6.2 Add web translation strings (badge label with a plural count, "Remove from thread", "Artifacts") and run `vp run i18n:generate`. Verify that `vp run i18n:check` passes.

## 7. Mobile

- [ ] 7.1 Show the artifact count on mobile thread rows (`features/threads/thread-list-v2-items.tsx`), and add an Artifacts list in thread detail that opens with `Linking.openURL` and supports remove. Verify with component tests for the count and the list.
- [ ] 7.2 Add mobile translation strings. Verify that `vp run i18n:check` passes.

## 8. Docs

- [ ] 8.1 Add a short note to `docs/user/` explaining that artifacts Claude publishes appear on the thread's sidebar row, open in the system browser, and that removing one does not delete it on claude.ai. Verify that the page reads correctly in the user-docs voice.

## 9. Integration

- [ ] 9.1 Run `vp run check:changed`. Verify that it passes.
- [ ] 9.2 Run an integrated pass with `test-t3-app` on web, using a real Claude Code (Haiku) turn that publishes an artifact. Check that the badge appears live, the hover list and open work, and removing it syncs to a second connected client. Repeat on mobile with `test-t3-mobile`.

## Workflow follow-up

- Archive the change after the PR is reviewed and merged.
