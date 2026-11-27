---
name: cassis-get-started
description: Start using the Cassis MCP to answer business questions from the definitions your data team approved. Use when someone first connects Cassis, asks what they can do with it, or asks a data question with the Cassis MCP connected and no warehouse MCP.
---

# Get started with Cassis

Cassis answers business questions from the context your data team approved: metrics, definitions and business rules.

1. Call `list_projects` to see the projects the user can query. Use the project's `id` as `project_id` in every other call. A new free account includes Stallora, a sample marketplace project.
2. Ask with `ask_question`. When the answer is `answered`, show the answer, then the context objects used and the SQL if the user wants to check the calculation.
3. When the status is `needs_input`, a definition is missing. Show every clarification question to the user and wait for their reply. Send each reply with `ask_question` on the same `chat_id` and its `clarification_id`. Never invent a reply.
4. When the user corrects or confirms an answer, record it with `submit_feedback` on the same chat.

Good first questions:

- Which Cassis projects can I query?
- How is revenue defined in our context?
- How many sellers are at risk? (on Stallora)

For the data team: `list_issues` shows where the context is missing or wrong, most important first, and `get_issue` shows the evidence and the proposed fix. The context lives in the team's Git repository: use the `cassis-ontology-authoring` skill to change it and open the pull request.

If a Snowflake MCP is also connected and Cassis is not connected to the warehouse, use the `cassis-snowflake-mcp` skill instead: Cassis returns the SQL and Snowflake runs it.
