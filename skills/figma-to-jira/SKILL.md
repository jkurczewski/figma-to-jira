---
name: figma-to-jira
description: Use when creating a Jira task from a Figma comment. Turns pasted comment text and/or a screenshot plus a Figma link into a concise, outcome-focused Jira issue in a preconfigured project, recommending or attaching an epic. Triggered by the /ftj and /ftj-setup commands.
---

# Figma → Jira

Convert a Figma comment into a concise, outcome-focused Jira task. This skill
backs the `/ftj-setup` and `/ftj` commands.

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
  getJiraProjectIssueTypesMetadata, searchJiraIssuesUsingJql, createJiraIssue.

## Configuration

Per-project config at `./.figma-to-jira/config.json` (relative to the current
working project):

```json
{
  "cloudId": "abc-123",
  "projectKey": "MOB",
  "projectName": "Mobile App",
  "defaultIssueType": "Story",
  "labels": ["figma"]
}
```

If this file is missing when running `/ftj`, tell the user to run `/ftj-setup`
first.

## Setup flow (`/ftj-setup`)

1. Confirm the Atlassian connector is available (discover a Jira tool via
   ToolSearch). If not, give connection instructions and stop.
2. Fetch the accessible Atlassian site(s) to get `cloudId`. If more than one,
   ask which site.
3. Fetch visible Jira projects for that site. Present them with
   `AskUserQuestion` and let the user pick the target project.
4. Fetch the project's available issue types. Set `defaultIssueType` to `Story`
   if present, otherwise `Task`.
5. Write `./.figma-to-jira/config.json` with cloudId, projectKey, projectName,
   defaultIssueType, and an optional `labels: ["figma"]`.
6. Confirm: "Setup done — tasks will go to <projectName> (<projectKey>)."

## Create flow (`/ftj <args>`)

1. Load `./.figma-to-jira/config.json`. If missing → direct to `/ftj-setup`.
2. **Parse the arguments** (free-form text after `/ftj`):
   - **Figma link:** find a `figma.com` URL. It is REQUIRED and should point to
     a specific comment. If there is no link, or it clearly does not point to a
     comment, politely ask the user for a screenshot, the comment text, or a
     corrected link — then stop. Do not force task creation.
   - **Epic key:** match `\b[A-Z][A-Z0-9]+-\d+\b` (e.g. `SHELL-141`). If found,
     treat it as an explicit epic — validate it is an Epic in the configured
     project and attach the new task to it, skipping recommendation.
   - **Content:** everything else is the comment content — plus any pasted
     screenshot image, which you read visually. If no usable content is present,
     ask for the text or a screenshot; do not invent it.
3. **Epic selection** (only if no explicit epic key):
   - Query epics in the project via JQL:
     `project = <projectKey> AND issuetype = Epic ORDER BY updated DESC`.
   - Semantically match the content to the epics. Propose the 1–2 best with
     `AskUserQuestion`, plus a "No epic" option. Use the user's choice.
4. **Write the task** following the principles below.
5. **Preview** the task (title, description, acceptance criteria, epic, issue
   type) and ask for approval before creating.
6. On approval, create the issue via the Atlassian create-issue tool (project =
   projectKey, issuetype = defaultIssueType, labels from config, parent/epic
   link as chosen). Return the created issue's URL.

## Task-writing principles

- **Outcome over solution.** Describe what should be true when the work is done,
  not how to implement it. Do not prescribe a solution.
- **Compact.** A tight note beats a wall of text. No filler.
- **Structure:**
  - **Title** — short, outcome-oriented (not a restatement of the raw comment).
  - **Description** — a simple user story / the expected end result.
  - **Acceptance criteria** — 2–4 observable, checkable points.
  - **Source** — the Figma comment link, always included.
- **Ask, don't guess.** Missing content or a bad link → request it.

## Example

Input: `/ftj https://figma.com/file/…?comment=123 SHELL-141 The empty state on
the dashboard looks broken — no illustration and the copy overflows on mobile.`

Result: a `Story` in the configured project, linked to epic `SHELL-141`, titled
e.g. "Dashboard empty state renders correctly on mobile", with a compact
description of the expected outcome, 2–4 acceptance criteria, and the Figma link
as source.
