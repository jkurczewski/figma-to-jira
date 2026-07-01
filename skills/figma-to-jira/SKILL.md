---
name: figma-to-jira
description: Use when creating a Jira task from a Figma comment. Reads the comment automatically from a Figma link (or accepts pasted text/screenshot) and creates a concise, outcome-focused Jira issue in a preconfigured project, recommending or attaching an epic. Triggered by the /figma-to-jira:add and /figma-to-jira:setup commands.
---

# Figma → Jira

Turn a Figma comment into a concise, outcome-focused Jira task. This skill backs
the `/figma-to-jira:setup` and `/figma-to-jira:add` commands.

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
  Scopes: File content read-only + Comments read-only. It is captured during
  setup and stored locally (never committed — see Configuration).

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
  "figmaToken": "figd_..."
}
```

`figmaToken` is a secret. Setup writes `./.figma-to-jira/.gitignore` containing
`*` so the whole folder (token included) is never committed. Never print the
token in output. If this file is missing when running `/figma-to-jira:add`, tell
the user to run `/figma-to-jira:setup` first.

## Setup flow (`/figma-to-jira:setup`)

1. Confirm the Atlassian connector is available (discover a Jira tool via
   ToolSearch). If not, give connection instructions and stop.
2. Fetch the accessible Atlassian site(s) to get `cloudId`. If more than one,
   ask which site.
3. Fetch visible Jira projects for that site. Present them with
   `AskUserQuestion` and let the user pick the target project.
4. Fetch the project's available issue types. Set `defaultIssueType` to `Story`
   if present, otherwise `Task`.
5. Ask the user for their Figma personal access token (scopes: File content
   read-only, Comments read-only). Point them at Figma → Settings → Security →
   Personal access tokens.
6. Write `./.figma-to-jira/config.json` (cloudId, projectKey, projectName,
   defaultIssueType, `labels: ["figma"]`, figmaToken) and write
   `./.figma-to-jira/.gitignore` with a single line `*`.
7. Confirm: "Setup done — tasks will go to <projectName> (<projectKey>)."

## Create flow (`/figma-to-jira:add <args>`)

1. Load `./.figma-to-jira/config.json`. If missing, or `figmaToken` is absent →
   direct to `/figma-to-jira:setup`.
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
   - Call the Figma REST API:
     `GET https://api.figma.com/v1/files/<fileKey>/comments` with header
     `X-Figma-Token: <figmaToken>` (run via a Bash `curl`; never echo the token).
   - Find the comment whose `id` equals the fragment id; else the one whose
     `client_meta.node_id` matches the `node-id`. Use its `message` (and author)
     as the source content.
   - **On `403`/token expired:** tell the user their Figma token expired and to
     re-run `/figma-to-jira:setup` with a fresh token; stop.
   - **If the comment cannot be found / no link:** fall back — ask the user for a
     screenshot, the text, or a corrected link; do not invent content.
4. **Epic selection** (when needed):
   - Query epics via JQL:
     `project = <projectKey> AND issuetype = Epic [AND summary ~ "<name>*"] ORDER BY updated DESC`.
     (`issuetype = Epic` works even on localized Jira instances.)
   - For a name lookup, confirm the best match. With no epic given, propose the
     1–2 best matches with `AskUserQuestion` plus a "No epic" option.
5. **Write the task** following the principles below.
6. **Preview** the task (title, description, acceptance criteria, epic, issue
   type) and ask for approval before creating.
7. On approval, create the issue via the Atlassian create-issue tool (project =
   projectKey, issuetype = defaultIssueType, labels from config, and the chosen
   epic as `parent` — the `parent` field links a Story to an Epic in both
   company- and team-managed projects). Return the created issue's URL.

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

Input: `/figma-to-jira:add https://figma.com/design/KEY?node-id=16628-53616#1764710155 TRACKER-142`

The plugin reads comment `1764710155` from the Figma file, drafts a `Story` in
the configured project linked to epic `TRACKER-142`, with a compact outcome-focused
description, 2–4 acceptance criteria, and the Figma link as source — then shows a
preview for approval before creating.
