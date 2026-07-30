---
name: cassis-snowflake-mcp
description: Use this skill to query data with Cassis (which grounds the question against the user's project ontology and returns SQL, a plan to approve, or an ontology gap to clarify) and Snowflake (which executes the SQL). Triggers when the user says "ask cassis", "query with cassis", "cassis question", or asks any natural-language data question while the Cassis MCP and a Snowflake MCP are both connected. Handles all three Cassis response paths: direct SQL, plan approval, and ontology-gap clarification.
---

# Cassis + Snowflake MCP integration

This skill orchestrates the Cassis MCP (which grounds the question against the project ontology) and a Snowflake MCP (which executes the resulting SQL) so the user can ask plain-language questions and get grounded answers.

## Scope

This skill assumes the user's Cassis project is set up to **generate SQL only**, with no direct database connection on Cassis's side. The skill always runs the generated SQL on the separately connected Snowflake MCP.

## Required MCPs

- Cassis MCP, authenticated to the user's Cassis org. Org auth is handled at the MCP layer, so the skill works across any project the user has access to.
- A Snowflake MCP connected to the warehouse holding the data referenced by the chosen Cassis project. Cassis returns fully-qualified table names (`"DB"."SCHEMA"."TABLE"`), so the right database and schema are picked automatically. The user only needs to confirm the warehouse context.

## First-run check (run once per session)

Before the first question, verify:

1. Cassis MCP responds to `ping`.
2. At least one Snowflake MCP is reachable (`list_objects` with `object_type=database` succeeds).
3. A Cassis project_id has been provided. If multiple projects are accessible and none was specified, call `list_projects` and prompt the user to pick.

If any check fails, surface a clear, actionable error:

- Cassis MCP unreachable: "Reconnect the Cassis MCP from your client. OAuth re-auth may be required."
- Snowflake MCP unreachable: "Reconnect the Snowflake MCP for your warehouse."
- No project specified: list the available projects with name and id, ask the user to pick one.

## The workflow

Call `ask_question` with `project_id` and `question` (no `chat_id` on the first call).

Branch on `status`:

### Path A. `status: answered`, `sql` present

Happy path. Run the SQL on Snowflake. Show the result (see Result display) and the generated SQL.

### Path B. `status: answered`, no SQL, gap text in `answer`

Cassis flagged that the question references a concept that isn't in the ontology. Frame the response so the user can see immediately that the next move is theirs:

1. A short header line: **"Cassis is asking you to clarify a concept in your question."**
2. The `answer` text surfaced verbatim (it lists the undefined concepts and proposes definitions).
3. A one-line closing instruction: **"Reply with your definitions and I'll continue."**

Then wait. When the user replies, call `ask_question` again with the same `chat_id` and the clarification as the new `question`. This usually transitions to Path C.

### Path C. `status: needs_execution`, plan present

Render the plan in a readable format:

- **Intent**: `plan.intent`
- **Strategy**: `plan.strategy` as a numbered list
- **Objects used**: `plan.objects_used`
- **Assumptions**: `plan.assumptions`. Emphasize these. They're the defaults Cassis picked and the most likely place to spot a misread of the question.

Ask for explicit approval. When the user approves, call `ask_question` again with the same `chat_id` and `execute_pending_plan: true`. The follow-up returns Path A; handle it the same way.

### Path D. `status: error`

Surface the error and stop. Do not retry silently.

## Snowflake execution rules

- Run the SQL exactly as Cassis returned it. The SQL is Cassis's grounded output; the skill never silently edits it.
- **Auto-LIMIT** (row-returning SELECTs only). If the top-level SELECT has no `LIMIT`, no `GROUP BY`, and no aggregate function (`COUNT`, `SUM`, `AVG`, etc.), wrap the query with `LIMIT 10` for the preview run. Issue a separate `SELECT COUNT(*) FROM (<original SQL>)` to get the true total. Display the 10-row preview, the true total, and offer to save the full result as CSV. Skip this for aggregate or `GROUP BY` queries; their row count is already bounded.
  - Preview note above the result: "Preview of 10 rows (of total M). Ask for the full CSV or a higher row cap if needed."
- If Snowflake errors:
  - Surface the error text to the user clearly.
  - Send the error back to Cassis to fix, in the same chat. Call `ask_question` again with the same `chat_id` and the error, for example: "The SQL you generated failed on Snowflake with this error: <error text>. Please correct it." Cassis re-grounds against the ontology and returns corrected SQL, a plan, or an ontology-gap clarification. Handle the response with Paths A to D as usual, and run the corrected SQL under the same approval and Auto-LIMIT rules.
  - This is the durable path: Cassis owns the fix in-session, so a recurring error becomes an ontology enrichment rather than a one-off local patch.
  - Only the error text and the failing SQL go back to Cassis, never result rows or values (see Data protection). Errors are metadata: table and column names, types, syntax.
  - A local patch is a last resort, only if Cassis cannot resolve the error and the fix derives from a concrete signal (Snowflake "did you mean X?", a `describe_object` match, or an obvious casing or quoting issue). Keep it minimal (one column or one quote, never a restructure), show it alongside the original, prepend this warning verbatim, and get explicit approval before running it:
    > Warning: this is a local workaround. The Cassis ontology is the durable fix for this kind of mismatch. Running this patched query bypasses the ontology, so the same error will hit the next person who asks a related question. Report the issue to your Cassis admins so they can update the ontology.

## Result display

- Always show: the generated SQL, plan assumptions (if any), and the true row count (not just the preview size).
- Up to 20 rows: render as a markdown table inline.
- More than 20 rows: show the first 10 rows + total count, and offer to save the full result set as CSV.
- One-line summary above the table when the data is small and the shape is obvious (e.g. "5 traffic sources, total revenue from $X to $Y"). This is the only place the skill paraphrases the data, and it goes only to the user, never back to Cassis.

### CSV save path

When the user accepts the CSV save offer, write the file to disk and report the path back:

- Look for an output convention in the workspace's `CLAUDE.md` (or equivalent session instruction). If one exists, write under a `cassis-queries/` subfolder inside it.
- If no output convention is set anywhere, ask the user where to save before writing. Do not invent a default.
- Filename: `cassis-{project-slug}-{chat-id-short}-{YYYYMMDD-HHmm}.csv`.

## Data protection (hard rule)

**Result values from Snowflake never flow back to Cassis automatically.**

When the user sends a refinement follow-up ("now by month", "filter to last week"), the next `ask_question` call carries only the user's text plus the `chat_id`. The skill does not paraphrase, summarize, paste, or otherwise inject any Snowflake result values into the prompt.

Execution errors are the one thing the skill sends back on purpose. When a query fails, the error text and the failing SQL go to Cassis on the same `chat_id` so it can correct the SQL in-session and, where relevant, surface an ontology gap to enrich. Errors are metadata (table and column names, types, syntax), not data. Never include result rows or values in that message.

If the user types a value into their own follow-up ("the result was $1.2M, our definition is..."), that's their call. Manual accept mode (recommended in setup docs) gives them a second chance to confirm before the call goes out.

After the first answer in a session, show this once:

> Refinement follow-ups send only your text to Cassis, never the result values from Snowflake. Query errors are sent back so Cassis can fix the SQL in-session.

## Ontology gap notification

When the user works through Path B (clarification) and reaches a final answer, append once per session, after the result:

> Note: your question referenced a concept that isn't yet defined in your project's ontology. Cassis has flagged the gap. Your Cassis admins can review it in the Cassis app and accept a suggested enrichment to the ontology, so future questions on the same concept get answered without redefining it.

Do not repeat this message after every answer in the session.

## Chat continuity

- Preserve `chat_id` across turns within a single user session so refinements thread correctly.
- Do not persist `chat_id` across sessions. A fresh session starts a fresh Cassis chat.

## Approval points (with manual accept mode)

Per question, the skill typically asks for 2 to 4 approvals:

1. The initial `ask_question` call.
2. If the plan path triggers: the `execute_pending_plan` call.
3. The Snowflake preview query.
4. If a separate `COUNT(*)` is needed for the true total on a row-returning SELECT: a second Snowflake call. Avoid this when the user only wants the preview.

Avoid chatty patterns that would create more than this. Batch where possible.
