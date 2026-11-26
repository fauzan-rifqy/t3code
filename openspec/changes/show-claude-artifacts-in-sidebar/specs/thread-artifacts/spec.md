# Spec Delta

## Purpose

Keeps the claude.ai Artifacts that an agent publishes during a thread attached to that thread, so users can find and reopen them from the sidebar without scrolling through the chat.

## ADDED Requirements

### Requirement: Published artifacts are recorded on their thread

When an agent's `Artifact` tool call publishes a page and returns a claude.ai artifact URL (`claude.ai/artifact/<id>` or `claude.ai/code/artifact/<uuid>`), the system SHALL record that artifact on the thread that ran the call. The record SHALL hold the URL, a title and the publish time.

#### Scenario: Successful publish

- **WHEN** a Claude Code turn completes an `Artifact` publish whose result contains `https://claude.ai/code/artifact/abc`
- **THEN** the thread's artifact list contains an entry with that URL

#### Scenario: Failed or non-publish call

- **WHEN** an `Artifact` call fails, or is a `read`, `list`, `open`, `pin`, `unpin` or `quickstart` action
- **THEN** the thread's artifact list does not change

#### Scenario: URL in other tool output

- **WHEN** a claude.ai artifact URL appears in a tool call other than `Artifact`, or in assistant text
- **THEN** the thread's artifact list does not change

### Requirement: Artifact titles are readable

Each recorded artifact SHALL have a title. The title SHALL be the publish call's `title` input when one is given. Otherwise it SHALL be the published file's base name without its extension. If neither exists, it SHALL be the URL.

#### Scenario: Title from input

- **WHEN** an artifact is published with `title: "Error rate dashboard"`
- **THEN** its entry's title is "Error rate dashboard"

#### Scenario: Title from file name

- **WHEN** an artifact is published from `/tmp/scratch/release-notes.html` with no `title` input
- **THEN** its entry's title is "release-notes"

### Requirement: Republishing updates rather than duplicates

A publish to an artifact URL that is already recorded on the thread SHALL update that entry's title and publish time. It SHALL NOT add a second entry.

#### Scenario: Agent republishes an artifact

- **WHEN** an artifact already recorded on the thread is published again to the same URL
- **THEN** the thread still has one entry for that URL, showing the newer title and publish time

### Requirement: Deleted artifacts are removed

When an `Artifact` delete action for a recorded URL succeeds, the system SHALL remove that entry from the thread.

#### Scenario: Agent deletes an artifact

- **WHEN** the agent successfully deletes an artifact recorded on the thread
- **THEN** the thread's artifact list no longer contains it

### Requirement: Artifact list is bounded

A thread SHALL keep at most 20 artifact entries. Adding an entry beyond that limit SHALL drop the entry with the oldest publish time.

#### Scenario: Twenty-first artifact

- **WHEN** a thread holding 20 artifacts records a new one
- **THEN** the thread holds 20 artifacts, the new one included and the oldest dropped

### Requirement: Sidebar thread row shows artifacts

On web and desktop, a sidebar thread row with one or more artifacts SHALL show an artifact badge with the count. The badge's hover card SHALL list each artifact by title, newest first. Rows with no artifacts SHALL show no badge.

#### Scenario: Thread with artifacts

- **WHEN** a thread has two recorded artifacts
- **THEN** its sidebar row shows an artifact badge reading 2, and hovering it lists both titles

#### Scenario: Thread without artifacts

- **WHEN** a thread has no recorded artifacts
- **THEN** its sidebar row shows no artifact badge

#### Scenario: Live update

- **WHEN** the agent publishes an artifact while the user watches the sidebar
- **THEN** the badge appears or its count increases without a reload

### Requirement: User can open an artifact

On web and desktop, activating an artifact entry SHALL open its URL the same way other links in a thread open. When the user has chosen to open links in the app and the thread's environment supports the browser panel, it SHALL open as a tab in the thread's side browser panel. Otherwise it SHALL open in the system browser.

#### Scenario: Open in the side panel

- **WHEN** the user has chosen to open links in the app and clicks an artifact title in the sidebar hover card
- **THEN** the artifact opens as a tab in that thread's browser panel, beside the chat

#### Scenario: Open in the system browser

- **WHEN** the user has chosen to open links in the system browser, or the browser panel is unavailable for the thread's environment
- **THEN** the artifact URL opens in the system browser

#### Scenario: Modifier click

- **WHEN** the user Cmd-clicks or Ctrl-clicks an artifact title
- **THEN** the artifact URL opens in the system browser, whatever the link setting is

#### Scenario: Panel cannot open

- **WHEN** opening the artifact in the browser panel fails
- **THEN** the artifact URL opens in the system browser instead

### Requirement: User can remove an artifact from a thread

Each artifact entry SHALL offer a remove action. Removing an entry SHALL delete it from the thread on every connected client. It SHALL NOT delete the artifact on claude.ai. A later publish to the same URL SHALL add it back.

#### Scenario: Remove then republish

- **WHEN** the user removes an artifact from a thread, and the agent later republishes to that URL
- **THEN** the entry disappears from the thread's sidebar row after the removal and reappears after the republish

### Requirement: Mobile shows artifacts

The mobile thread list SHALL show the artifact count on rows that have artifacts. The mobile app SHALL let the user open each artifact in the system browser.

#### Scenario: Mobile row

- **WHEN** a thread with one artifact appears in the mobile thread list
- **THEN** its row shows an artifact count of 1, and tapping the artifact opens its URL

### Requirement: Compatible with older peers

The artifact list SHALL be optional on the wire and in stored thread data. Clients and servers without artifact support SHALL keep decoding threads that carry it, and threads without it SHALL decode as having no artifacts.

#### Scenario: Older server

- **WHEN** a new client connects to a server that never sends artifact data
- **THEN** threads load normally and show no artifact badge
