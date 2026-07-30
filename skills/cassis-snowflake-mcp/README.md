# Cassis + Snowflake MCP integration

MCP integration to use Cassis with Snowflake when Cassis isn't natively connected to your warehouse. Cassis grounds the question against your project's ontology and generates the SQL; Snowflake executes it.

## What this is

The skill is a thin behavior guide. The heavy lifting splits between three pieces:

- **Cassis MCP**: grounds your question against the project's ontology and responds with SQL, a plan with assumptions to approve, or an ontology gap to clarify.
- **Snowflake MCP**: executes the SQL. Read-only by default.
- **`cassis-snowflake-mcp` skill**: wires the two together. Enforces approvals, handles auto-LIMIT, routes execution errors back to Cassis for in-session correction, and keeps Snowflake result values from leaking back into Cassis on follow-ups.

Cassis returns fully qualified table names (`"DB"."SCHEMA"."TABLE"`), so the Snowflake side only needs the warehouse and credentials locked in.

## Where it runs

This is an MCP workflow, so it runs in any client that supports the Model Context Protocol.

## When to use it

Reach for this workflow when your Cassis project is configured to **generate SQL only**, without a direct warehouse connection on Cassis's side.

In that setup, Cassis grounds the question against your ontology and returns one of three things: SQL ready to execute, a plan with assumptions for you to approve, or an ontology gap that needs a definition before it can answer. It doesn't run the SQL itself. This skill plugs in the Snowflake MCP as the executor, so you get end-to-end plain-language access to your data without giving Cassis warehouse credentials.

If your Cassis project is connected directly to the warehouse (Cassis handles both generation and execution), you don't need this skill. Use Cassis on its own.

## How a question flows

1. First-run check: the skill verifies both MCPs respond and a `project_id` is set. If multiple Cassis projects are accessible, it prompts you to pick one.
2. Calls `ask_question` with `project_id` and the question.
3. Branches on the response status:
   - **`answered` with SQL**: runs the SQL on Snowflake. Shows result, SQL, true row count.
   - **`answered` with gap text**: surfaces Cassis's clarification verbatim, waits for you to define the missing concept, then resends on the same `chat_id`.
   - **`needs_execution` with plan**: renders intent, strategy, objects used, and assumptions. Assumptions get emphasized, because they're where misreads usually hide. Asks for approval before resending with `execute_pending_plan: true`.
   - **`error`**: surfaces and stops. No silent retries.
4. Auto-LIMIT: for row-returning SELECTs without `LIMIT`, `GROUP BY`, or aggregates, the skill wraps the query with `LIMIT 10` for preview and runs a separate `COUNT(*)` for the true total. CSV save offered for the full result.
5. Error recovery: if Snowflake rejects the SQL, the skill sends the error back to Cassis on the same `chat_id` and runs the corrected query, rather than hand-patching. Only the error text and the SQL go back, never result rows, so Cassis can fix the SQL in-session and flag an ontology gap to enrich. A minimal local patch is a last resort, only on a concrete signal and with explicit approval.

## Setup

### 1. Connect the Cassis MCP

Cassis MCP is a remote MCP hosted by Cassis at `https://app.getcassis.com/mcp/`. Auth is OAuth 2.1, the browser handles login, no manual token. One connection covers every project you have access to.

Full reference: [docs.getcassis.com/mcp](https://docs.getcassis.com/mcp.html). MCP access is currently limited to Cassis design partners.

### 2. Connect the Snowflake MCP

The Snowflake MCP is [snowflake-labs-mcp](https://github.com/Snowflake-Labs/mcp), self-hosted via `uvx`. Four sub-steps.

**a. Snowflake auth (JWT key-pair).** The connector supports every Snowflake auth method (username/password, externalbrowser SSO, OAuth, MFA, key-pair). We recommend JWT key-pair: no browser prompts on every MCP call, and it works on accounts without a SAML IdP.

```bash
mkdir -p ~/.snowflake
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out ~/.snowflake/rsa_key.p8 -nocrypt
chmod 0600 ~/.snowflake/rsa_key.p8
openssl rsa -in ~/.snowflake/rsa_key.p8 -pubout
```

The private key stays on your machine and is what the MCP uses to sign auth tokens; anyone with read access to that file can authenticate as your Snowflake user. The `chmod 0600` locks it to you. The public key (the next command's output) is what gets registered in Snowflake.

Copy the public key (the base64 block, joined on one line) and run in Snowsight:

```sql
ALTER USER <your-user> SET RSA_PUBLIC_KEY='<paste-the-key>';
```

**b. Connection profile.** Add to `~/.snowflake/connections.toml` (create the file if missing, then `chmod 0600`):

```toml
[your-connection-name]
account = "<your-account-id>"
user = "<your-user>"
authenticator = "SNOWFLAKE_JWT"
private_key_file = "~/.snowflake/rsa_key.p8"
warehouse = "<your-warehouse>"
database = "<your-database>"
schema = "<your-schema>"
role = "<your-role>"
```

For `role`, use a dedicated read-only Snowflake role (e.g. `USAGE` on the warehouse + `SELECT` on the relevant schemas). The YAML config in step c blocks write statements at the MCP layer, but that's client-side enforcement; a read-only role at the Snowflake level is the durable safety net if anyone bypasses the MCP with the same key.

Test the connection before going further:

```bash
uvx --from snowflake-cli snow connection test --connection <your-connection-name>
```

**c. Service config.** A small YAML telling the MCP which statement types to allow. Save as `snowflake-mcp-config.yaml` anywhere convenient:

```yaml
agent_services: []
search_services: []
analyst_services: []
other_services:
  object_manager: true
  query_manager: true
  semantic_manager: false
sql_statement_permissions:
  - Select: true
  - Describe: true
  - Show: true
  - Use: true
  - Explain: true
  - Insert: false
  - Update: false
  - Delete: false
  - Merge: false
  - Truncate: false
  - Drop: false
  - Alter: false
  - Create: false
  - Grant: false
  - Revoke: false
```

Keep writes off unless a separate workflow explicitly needs them.

**d. MCP client entry.** Add an entry pointing at `uvx snowflake-labs-mcp` with your service config path and connection name. JSON form (most clients use this; translate the same fields if your client's config is TOML or another format):

```json
{
  "mcpServers": {
    "snowflake-<name>": {
      "command": "uvx",
      "args": [
        "snowflake-labs-mcp",
        "--service-config-file",
        "/absolute/path/to/snowflake-mcp-config.yaml",
        "--connection-name",
        "<your-connection-name>"
      ]
    }
  }
}
```

Restart the client and approve the new MCP when prompted. Smoke test with a `SELECT CURRENT_DATABASE()` call through the Snowflake MCP.

Full Snowflake MCP reference: [github.com/Snowflake-Labs/mcp](https://github.com/Snowflake-Labs/mcp).

### 3. Install the skill

For Claude Code, install it from the Cassis marketplace:

```
/plugin marketplace add GetCassis/skills
/plugin install cassis-snowflake-mcp@cassis
```

For clients without native skill loading, paste the contents of [`SKILL.md`](SKILL.md) into your custom instructions, system prompt, or agent instructions field, prefixed with:

> Follow the rules in the guide below for any Cassis question in this session.

The skill triggers on phrases like "ask cassis", "query with cassis", or "cassis question".

## Data protection

The skill is designed so Snowflake result values don't flow back into Cassis on follow-ups. Refinement requests ("now by month", "filter to last week") send only your text plus the `chat_id` to Cassis. The skill won't paraphrase Snowflake output or inject result values into the next prompt.

Execution errors are the exception, by design: when a query fails, the error text and the failing SQL are sent back to Cassis so it can correct the SQL in-session and surface ontology gaps. Errors are metadata (names, types, syntax), never result values. A local patch stays a last resort, and the skill surfaces a verbatim warning before running one, since patches bypass the ontology and the durable fix is always an ontology update in the Cassis app.

These are design intentions, not deterministic guardrails. The model running the skill can still drift. We recommend running Cassis tool calls in **manual accept mode** in your client, so you get a second checkpoint before any text leaves your machine.

## Feedback

Issues and improvements: [github.com/GetCassis/skills/issues](https://github.com/GetCassis/skills/issues).
