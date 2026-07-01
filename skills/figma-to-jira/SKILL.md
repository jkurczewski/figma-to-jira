---
name: figma-to-jira
description: Use when creating a Jira task from a Figma comment. Reads the comment automatically from a Figma link (or accepts pasted text/screenshot) and creates a concise, outcome-focused Jira issue in a preconfigured project, recommending or attaching an epic. Triggered by the /ftj:add and /ftj:setup commands.
---

# Figma → Jira

Turn a Figma comment into a concise, outcome-focused Jira task. This skill backs
the `/ftj:setup` and `/ftj:add` commands.

## Prerequisites

- The user must have the **Atlassian connector** connected in Claude. If Jira
  tools are unavailable, stop and tell them to connect it
  (Settings → Connectors → Atlassian), then retry.
- Discover Atlassian tools at runtime with `ToolSearch`, e.g.
  `ToolSearch("atlassian jira create issue")`,
  `ToolSearch("atlassian jira search projects")`,
  `ToolSearch("atlassian jira search issues jql")`.
  Never assume fixed tool names — use whatever the search returns. Expected
  candidates (hints only): getAccessibleAtlassianResources / getVisibleJiraProjects,
  getJiraProjectIssueTypesMetadata, searchJiraIssuesUsingJql, createJiraIssue,
  getJiraIssue.
- A **Figma personal access token** (per user) is required to read comments.
  Scopes: File content read-only + Comments read-only — add **Comments: write**
  if the user enables auto-reply (see below). It is captured during setup and
  stored locally (never committed — see Configuration).

## Efficiency (keep token use minimal)

- **Be terse — caveman-brief.** Speak to the user in as few words as possible:
  a one-line status, the preview, the result link. No preamble, no recaps, no
  narrating steps or tools. Prefer showing over explaining.
- **Never pull large payloads into context** (Jira and Figma responses are huge):
  - Figma comments: `curl -s` the response to a file, then extract only the
    target with `jq` (by comment id, else `client_meta.node_id`). Never print or
    read the whole dump.
  - Jira JQL: request minimal `fields` (e.g. `["summary"]` or `["labels"]`),
    small `maxResults`, `responseContentFormat: "markdown"`. If the result is
    still large it is saved to a file — read only what you need with `jq`.
  - `getVisibleJiraProjects`: pass `expandIssueTypes: false`.
- Reuse fetched data within a run; don't re-fetch. Never echo the Figma token.

## Configuration

Per-project config at `./.figma-to-jira/config.json` (relative to the current
working project):

```json
{
  "cloudId": "abc-123",
  "projectKey": "MOB",
  "projectName": "Mobile App",
  "defaultIssueType": "Story",
  "labels": ["figma"],
  "figmaToken": "figd_...",
  "confirmBeforeCreate": true,
  "autoReplyOnCreate": true,
  "autoReplyMessage": "Thanks! We've created a task for this — we'll get back to you soon once it's prioritized."
}
```

`confirmBeforeCreate` controls whether the plugin shows a preview and waits for
approval before creating the issue (`true`), or creates it immediately after
generating it and then reports what it made (`false`).

`autoReplyOnCreate` controls whether, after creating an issue, the plugin posts a
short confirmation reply on the source Figma comment. The reply is
`autoReplyMessage` with the created issue key appended in parentheses, e.g.
"Thanks! …once it's prioritized. (SHELL-518)" — the key only, not a full Jira
URL. It is posted under the user's own Figma token (i.e. in their name), so the
person adding the task owns it. `autoReplyMessage` is the base text (a sensible
default is shown).

`figmaToken` is a secret. Setup writes `./.figma-to-jira/.gitignore` containing
`*` so the whole folder (token included) is never committed. Never print the
token in output. If this file is missing when running `/ftj:add`, tell
the user to run `/ftj:setup` first.

## Setup flow (`/ftj:setup`)

1. Confirm the Atlassian connector is available (discover a Jira tool via
   ToolSearch). If not, give connection instructions and stop.
2. Fetch the accessible Atlassian site(s) to get `cloudId`. If more than one,
   ask which site.
3. Fetch visible Jira projects for that site (`expandIssueTypes: false` to keep
   the payload small). Present them with `AskUserQuestion` and let the user pick.
4. Fetch the project's available issue types. Set `defaultIssueType` to `Story`
   if present, otherwise `Task`.
5. Ask the user for their Figma personal access token. Point them at Figma →
   Settings → Security → Personal access tokens. Scopes: File content read-only,
   Comments read-only (plus Comments: write if they enable auto-reply next).
6. Ask (with `AskUserQuestion`) whether to **confirm before creating**: show a
   preview and wait for approval (`confirmBeforeCreate: true`), or create the
   issue immediately after generating it (`confirmBeforeCreate: false`).
7. Ask (with `AskUserQuestion`) whether to **auto-post a confirmation reply** on
   the Figma comment after a task is created. Explain it posts under their own
   token (in their name), so they own it, and that it requires the token's
   Comments: write scope. Set `autoReplyOnCreate` accordingly and default
   `autoReplyMessage` to "Thanks! We've created a task for this — we'll get back
   to you soon once it's prioritized." (offer to customise).
8. Write `./.figma-to-jira/config.json` (cloudId, projectKey, projectName,
   defaultIssueType, `labels: ["figma"]`, figmaToken, confirmBeforeCreate,
   autoReplyOnCreate, autoReplyMessage) and write `./.figma-to-jira/.gitignore`
   with a single line `*`.
9. Confirm: "Setup done — tasks will go to <projectName> (<projectKey>)."

## Create flow (`/ftj:add <args>`)

1. Load `./.figma-to-jira/config.json`. If missing, or `figmaToken` is absent →
   direct to `/ftj:setup`.
2. **Parse the arguments** (free-form text after the command):
   - **Figma link:** find a `figma.com` URL. REQUIRED. Extract the file key
     (`/design/<fileKey>` or `/file/<fileKey>`), the `node-id`, and the comment
     id (the `#<id>` fragment or a `?comment=<id>` / `#<id>` at the end).
   - **Epic:** if a key matching `\b[A-Z][A-Z0-9]+-\d+\b` (e.g. `TRACKER-142`) is
     present, attach directly (validate it is an Epic in the project). If an epic
     *name* is given instead (e.g. "DACH"), look it up (step 4) and confirm. If
     neither is given, recommend one (step 4).
   - **Extra text / screenshot:** anything else supplements or overrides the
     fetched comment.
3. **Read the comment from Figma** (this is the default — do not ask the user to
   paste it):
   - `curl -s` the Figma REST API to a file (do not read the full dump):
     `GET https://api.figma.com/v1/files/<fileKey>/comments` with header
     `X-Figma-Token: <figmaToken>` (never echo the token).
   - Extract only the target with `jq`: the comment whose `id` equals the
     fragment id; else the one whose `client_meta.node_id` matches the `node-id`.
     Use its `message` (and author) as the source content.
   - **On `403`/token expired:** tell the user their Figma token expired and to
     re-run `/ftj:setup` with a fresh token; stop.
   - **If the comment cannot be found / no link:** fall back — ask the user for a
     screenshot, the text, or a corrected link; do not invent content.
4. **Epic selection** (when needed) — use minimal `fields`, small `maxResults`,
   markdown format; if a result is large, `jq` the saved file, don't read it all:
   - Query epics via JQL (fields `["summary"]`):
     `project = <projectKey> AND issuetype = Epic [AND summary ~ "<name>*"] ORDER BY updated DESC`.
     (`issuetype = Epic` works even on localized Jira instances.)
   - For a name lookup, confirm the best match. With no epic given, propose the
     1–2 best matches with `AskUserQuestion` plus a "No epic" option.
   - **Inherit the epic's label(s):** once an epic key is known, query its child
     issues via JQL `parent = <EPIC-KEY>` with fields `["labels"]` (up to ~50),
     then tally label frequency with `jq` on the saved file. Teams sort by label,
     so take the label(s) present on **at least half** of the children (the
     dominant epic label, e.g. `DACH`). Add them to the task's labels. If no epic,
     or no label reaches the threshold, inherit nothing.
5. **Write the task** following the principles below.
6. **Confirmation (per `confirmBeforeCreate`):** if `true` (default), show a
   preview (title, description, acceptance criteria, epic, issue type, final
   labels) and wait for approval before creating. If `false`, skip approval and
   proceed straight to creation, then report what was created.
7. Create the issue via the Atlassian create-issue tool (project =
   projectKey, issuetype = defaultIssueType, labels = config `labels` **plus the
   dominant epic label(s) from step 4**, deduped, and the chosen epic as `parent`
   — the `parent` field links a Story to an Epic in both company- and
   team-managed projects). Return the created issue's URL.
8. **Auto-reply (if `autoReplyOnCreate` is true and a source comment id is
   known):** post the reply to that comment via the Figma REST API —
   `POST https://api.figma.com/v1/files/<fileKey>/comments` with body
   `{"message": "<autoReplyMessage> (<ISSUE-KEY>)", "comment_id": "<commentId>"}`
   and header `X-Figma-Token: <figmaToken>` (run via Bash `curl`; never echo the
   token). Append the created issue key in parentheses (key only, not a full
   URL), e.g. `... prioritized. (SHELL-518)`. On `403` (token lacks Comments:
   write), tell the user the task was created but the reply needs a token with
   write scope; do not fail the task.

## Task-writing principles

- **Outcome over solution.** Describe what should be true when the work is done,
  not how to implement it. Do not prescribe a solution — even if the comment
  suggests one, frame it as the decision/outcome to reach.
- **Compact.** A tight note beats a wall of text. No filler.
- **Structure:**
  - **Title** — short, outcome-oriented (not a restatement of the raw comment).
  - **Description** — a simple user story / the expected end result.
  - **Acceptance criteria** — 2–4 observable, checkable points.
  - **Source** — the Figma comment link, always included.
- **Ask, don't guess.** Missing content or an unreadable link → request it.

## Example

Input: `/ftj:add https://figma.com/design/KEY?node-id=16628-53616#1764710155 TRACKER-142`

The plugin reads comment `1764710155` from the Figma file, drafts a `Story` in
the configured project linked to epic `TRACKER-142`, with a compact outcome-focused
description, 2–4 acceptance criteria, and the Figma link as source — then shows a
preview for approval before creating.
