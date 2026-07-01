---
description: Create a Jira task from a Figma comment link. Usage: /ftj:add <figma comment link> [EPIC-KEY or epic name]
argument-hint: <figma comment link> [EPIC-KEY]
---

Create a Jira task from the following input using the **create flow** in the
`figma-to-jira` skill:

$ARGUMENTS

Parse the input as defined by the skill:
- Find the Figma link (required). Read the referenced comment automatically from
  Figma using my stored token. If the link is missing, or the comment cannot be
  read, ask for a screenshot, the text, or a corrected link, then stop.
- Detect an optional epic key matching `[A-Z][A-Z0-9]+-\d+` (e.g. TRACKER-142) to
  attach directly, or an epic name to look up. If neither is given, recommend one.
- Any extra text I paste supplements or overrides the fetched comment.

Then write a concise, outcome-focused task (title, compact description as a user
story, 2–4 acceptance criteria, Figma link as source), show a preview for my
approval, and create the issue on approval — exactly per the figma-to-jira skill.
If `./.figma-to-jira/config.json` is missing, tell me to run /ftj:setup first.

Keep replies terse (caveman-brief): a short status, the preview, the result link
— no narration of steps or tools. Follow the efficiency rules in the skill.
