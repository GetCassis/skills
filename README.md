# Cassis skills

Agent skills for working with [Cassis](https://getcassis.com), context maintenance for analytics agents. Keep governed context in Git, test changes with evals, review them through pull requests, and serve the approved version over MCP.

## The skills

| Skill | What it does |
|---|---|
| [`cassis-ontology-authoring`](skills/cassis-ontology-authoring/) | Extends an existing Cassis ontology in a git repository. Hand it a schema dump, dbt YAML, query history or an internal doc, and it writes the ontology files, validates them with `cassis-cli`, reviews the open questions with you, and opens the pull request. Comes with a [guide for humans](skills/cassis-ontology-authoring/README.md) carrying the same playbooks as copy-paste prompts, for agents that can't install skills. |
| [`cassis-snowflake-mcp`](skills/cassis-snowflake-mcp/) | Runs Cassis and a Snowflake MCP together when Cassis isn't natively connected to your warehouse. Cassis grounds the question against your ontology and returns SQL, Snowflake executes it. |

## Install

In Claude Code, add the marketplace once, then install what you need:

```
/plugin marketplace add GetCassis/skills
/plugin install cassis-ontology-authoring@cassis
/plugin install cassis-snowflake-mcp@cassis
```

To install a single skill by hand instead:

```bash
git clone https://github.com/GetCassis/skills.git /tmp/cassis-skills
cp -r /tmp/cassis-skills/skills/cassis-ontology-authoring ~/.claude/skills/
```

For clients without native skill loading (Dust, claude.ai, a custom agent), paste the contents of the skill's `SKILL.md` into your custom instructions or system prompt. The ontology authoring skill also ships its playbooks as standalone prompts in [its README](skills/cassis-ontology-authoring/README.md).

## Elsewhere

| What | Where |
|---|---|
| Product documentation, file format, CLI, MCP reference | [docs.getcassis.com](https://docs.getcassis.com/) |
| Two complete, copyable ontology trees | [GetCassis/cassis-ontology-examples](https://github.com/GetCassis/cassis-ontology-examples) |
| Audit a dbt project for what an agent will get wrong today | [GetCassis/dbt-agent-readiness](https://github.com/GetCassis/dbt-agent-readiness) |
| Modeling doctrine (how to write an ontology the agent reads well) | `cassis/AGENTS.md` in your own checkout, written by `cassis ontology fmt` |

## Feedback

Issues and improvements: [github.com/GetCassis/skills/issues](https://github.com/GetCassis/skills/issues).

MIT licensed.
