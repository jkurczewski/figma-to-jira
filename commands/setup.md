---
description: One-time per-project setup — pick the target Jira project and store your Figma token for figma-to-jira.
---

Run the **setup flow** from the `figma-to-jira` skill:

1. Verify the Atlassian connector is connected (discover a Jira tool via
   ToolSearch). If not, tell me how to connect it and stop.
2. Get the accessible Atlassian site(s) and cloudId (ask if more than one).
3. List my visible Jira projects and ask which one to target.
4. Detect the project's issue types; set the default to Story (fallback Task).
5. Ask me for my Figma personal access token (scopes: File content read-only,
   Comments read-only) so the plugin can read comments from Figma links.
6. Write `./.figma-to-jira/config.json`, make sure the folder is gitignored so
   the token is never committed, and confirm where tasks will go.

Follow the setup flow and configuration schema exactly as defined in the
figma-to-jira skill.
