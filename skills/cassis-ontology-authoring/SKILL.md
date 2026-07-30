---
name: cassis-ontology-authoring
description: |
  Extend and enrich a Cassis ontology that lives in a git repository. Turns the context the
  user provides (schema dumps or DDLs, dbt YAML, query history or dashboard SQL, internal
  docs, agent skills, meeting notes) into ontology files: tables, domains, metrics, joins,
  and context_md docs. Validates every change with cassis-cli and delivers it as a pull
  request. Use when asked to "add a schema to the ontology", "add this to Cassis", "extend
  the ontology", "fold this doc into the ontology", "mine our query history into metrics",
  or "enrich the context" in a repo containing a cassis/ tree.
argument-hint: "[path to the ontology repo, defaults to current directory]"
---

# Cassis ontology authoring

This skill extends an **existing** Cassis ontology in a git repo. It is not for initial project setup (see [docs.getcassis.com/setup](https://docs.getcassis.com/)). The human companion guide, with the same playbooks as copy-paste prompts for agents that can't install skills, is the [README in this skill's directory](https://github.com/GetCassis/skills/tree/main/skills/cassis-ontology-authoring).

## Doctrine order

`cassis/AGENTS.md` (written by `cassis ontology fmt`) is the modeling doctrine: where rules belong, how to write descriptions, metric conventions, and the extension-pass process itself ("Extension passes: structure before content"). **Read it in full before the first edit.** If it conflicts with this file, AGENTS.md wins.

## Preflight (once per session)

1. Locate the ontology tree (default `cassis/`, or the repo's configured base path). If absent, stop: this repo isn't a Cassis ontology checkout. Suggest `cassis ontology pull` or point to the git setup docs.
2. `cassis version` reports 1.1.0 or newer. The subcommand is `cassis version`, not `cassis --version` (there is no such flag). If the version is older: `pip install -U cassis-cli` (Python 3.10+). Older versions predate the Markdown domain format and are rejected on upload.
3. `CASSIS_API_KEY` is set. If not, ask the user (Organization settings → API keys). The project id comes from `CASSIS_PROJECT_ID`, or from the project file in the ontology tree on repos new enough to carry one. A legacy tree (`_project.yml`) holds no project id at all, so `CASSIS_PROJECT_ID` is required there for anything that talks to a project (`ontology test`, `eval run`, `upload`). `ontology check` and `ontology fmt` need only the key.
4. If the tree still holds legacy YAML domains (`_project.yml`, `_domain.yml`), run `cassis ontology fmt` once: it migrates the repo to the Markdown domain layout (domains become `README.md` files) and refreshes `AGENTS.md`. Commit that migration as its **own commit** so the content diff stays readable. It ships in the same PR as the content work; a separate PR is not worth the round trip.
5. Run `cassis ontology fmt` once: it writes or refreshes `cassis/AGENTS.md` to the current doctrine version. If it changed anything beyond AGENTS.md, commit that normalization separately. Then read AGENTS.md.
6. Create a working branch, and verify you are on it: `git rev-parse --abbrev-ref HEAD`. Never work on the default branch: merging to it publishes. Being handed a repo that is already off the default branch does not excuse skipping the check.

If the Cassis MCP is connected, use `get_project_status` to confirm the project and `get_source_schema` when adding tables. If it isn't connected, proceed with the CLI alone and ask the user for anything you'd otherwise read from the MCP.

## Route by input

| The user hands you | Playbook |
|---|---|
| A schema dump, information_schema export, or DDLs | 1. New schema |
| dbt YAML, column docs, catalog exports | 2. Fold in prose |
| Query history, dashboard or Metabase SQL | 3. Mine queries |
| Internal docs, runbooks, agent skills, notes | 2. Fold in prose |
| An issue from the Cassis queue (`list_issues`) | 4. Work the issue queue |
| A verified answer or definition from an expert | Eval capture (below) |

Run every pass by AGENTS.md's extension-pass process: scope, then structure, then fill bottom-up. On top of that doctrine, this skill adds the orchestration: work one domain per pass, sequence domains by query-history volume and user demand, and tell the user the proposed order before starting.

## The structure gate (a hard stop)

Before any content exists, propose the structure and **stop**. This is a full halt, not a heading you write above the work.

1. Post the proposed domain tree: each domain path, its one-line purpose, and which tables land under it. Name anything you plan to leave out.
2. End your turn there. Output nothing else.
3. Wait for the user to answer. Their reply is the approval, and it routinely changes the tree; a tree you approved yourself has tested nothing.

Do **not** create table files, domain files, joins, or metrics in the same turn as the tree. If you catch yourself writing "I have everything I need, now I'll create the files", you have skipped this gate: stop and post the tree instead.

The gate applies even when the request looks unambiguous, even when the structure mirrors an existing domain, and even when you are running unattended. If nobody answers, you stop; you do not proceed on your own approval.

## Playbook 1: new schema

Write in this order. It is not a list of concerns, it is the sequence: **(1) confirm visibility, (2) table files, (3) joins, (4) metrics, (5) domain README bodies last, (6) unknowns to the TODO list throughout.**

1. Confirm the tables are visible to Cassis: check `get_source_schema` (MCP) or ask the user. Tables in the ontology but absent from the source schema break query generation. If missing: connected warehouses sync automatically; metadata-only projects need the user to send their Cassis contact an information_schema dump first. Do not proceed for invisible tables. If the MCP is not connected, this is a question for the user, not something to assume: a schema dump proves the columns, never that Cassis can see the table.
2. Create `tables/<schema>/<table>.yml` per table, following AGENTS.md conventions. State only what the schema proves: grain, types, keys, nullability. No invented semantics.
   **A hedged guess is still a guess.** The no-invention rule is not limited to enum values. Do not write a storage format ("likely `'M:SS.mmm'`"), a cause of nullability ("may be non-numeric for cars that never started"), a physical event ("when the driver entered the pit lane"), or a scope qualifier ("ranks among non-finishers") that the source does not state. "Likely", "probably" and "appears to be" do not make a claim safe: the agent reads the sentence, not the hedge. Write what the dump proves, and put the rest in the TODO list.
   **Never invent a contrast with a column you did not read.** When a new column shares a name with one on an existing table (`position_order`, `status_id`, `lap`), the tempting sentence is "this one means X, unlike the other one". You cannot know that from a dump. Open the sibling's description: if it applies, reuse its wording; if you cannot confirm the relationship, say the relationship is unverified and add it to the TODO list. A confidently drawn false distinction is worse than silence, because it survives review as a documented decision.
   Encode an enum value list only when it is verified (a user-run `SELECT DISTINCT`, query history, or docs quoting the stored values). Never approximate one or write "etc.": a wrong example primes the agent harder than no example.
3. Add the joins the new tables need, hub-and-spoke (see Playbook 3). A shared column name is not a join.
4. Metrics are **not** derivable from a schema dump: a dump proves a column exists, never which aggregation the business calls by a name. Do not invent them. Do name the obvious candidates for the user, as a question, in the unknowns review: "these tables would usually carry metrics X and Y, do you want them, and how do you define them?" Silently shipping zero metrics is as unhelpful as guessing at them.
5. The domain files were created frontmatter-only at the structure step. Write each domain README body **last**, after every table file in that domain exists, mapping the tables and how they relate. A README written first is a guess about files you have not written yet.
   The README body carries **only** what AGENTS.md §4 "Never restate the structured fields" leaves it: cross-table routing, and join or filter recipes no structured field can express. Tables, columns, metrics and joins are already surfaced to the agent with their descriptions, units, grain and conditions. So no `## Tables` list, no re-pasted formula, no types, units, cast gotchas or format notes: those belong in the column description and nowhere else. Test each sentence: if it would still be true pasted into one table's or column's `description`, it belongs there, not here.
   The generated `<!-- cassis:nav:begin -->` block at the bottom of each README is not an exception to this and not a model for it. `fmt` regenerates it and strips it before the agent ever sees the file, so it is GitHub navigation for humans. Leave it alone and do not imitate it in the body.
6. Collect every open business question (ambiguous column, unknown enum values, unclear grain) into a TODO list **as you go**, written down, not held in your head: a `TODO-ontology.md` at the repo root, or an equivalent scratch file. It gets reviewed with the user before delivery (see "Review unknowns, then deliver") and folded into the PR body. Two bullets in a closing summary is not a TODO list. An explicitly flagged unknown is recoverable; a guessed definition poisons answers silently.

## Playbook 2: fold in prose

Route each piece of content to where AGENTS.md says it belongs:

- Durable business rules, routing logic, vocabulary → the right domain's `README.md` body (create topic subdomains with their own README for deep subjects rather than bloating one doc)
- Column-level facts (value lists, magic values, gotchas) → column descriptions
- Metric definitions → `metrics/<name>.yml`
- Alternate names → synonyms

If the source contradicts the existing ontology, flag it to the user with both versions. Never silently overwrite. Discard content that is transient (deadlines, people, one-off analyses): the ontology holds durable meaning only.

## Playbook 3: mine queries

From the queries, extract:

- Recurring metrics: name, grain, SQL → `metrics/*.yml`
- Join paths not yet in `joins.yml` (hub-and-spoke for keys shared by many tables; never a join between every pair of tables that share a column name)
- Gotchas: filters everyone applies, date guards, magic values → column descriptions or the domain `README.md`

Where two queries disagree on a definition, include both with the difference spelled out, per AGENTS.md.

## Playbook 4: work the issue queue

Cassis detects recurring problems from real conversations and failing evals. `list_issues` is the queue, `get_issue` adds the occurrences and the fix proposal, `get_issue_evidence` the raw evidence behind one occurrence. You have the same material the app's own assistant works from, so the value you add is not cleverness, it is running the loop properly.

1. **Read the whole issue before editing anything.** `get_issue`, then `get_issue_evidence` on the occurrences that actually decide the question. Occurrences are evidence, not a specification: a resolution a user stated in-thread is a claim, and gets verified like any other expert assertion before you encode it.
2. **Route on the proposal, and respect what it is telling you.**
   - `kind=apply_resolution`, `status=ready`: `mutations` are concrete ontology operations. They are app-shaped (`create_metric`, `update_column`), so translate them into the repo's equivalent edit rather than transcribing them; the file layout is yours to get right.
   - `kind=reference_tables`, `status=flagged`: this is an extension pass, not a patch. `get_source_schema` for the named tables, then Playbook 1 in full, structure gate included. Do not smuggle new tables in as a bugfix.
   - `kind=manual`, or no proposal at all: read `triage_reason`. This is usually undefined business meaning, which is exactly where a guess does the most damage. It routes to the user, not to a clever inference.
3. **Verify on your local files, never by re-asking.** `ask_question` answers from the *published* head; your fix is sitting uncommitted in the repo, so re-asking shows no change and reads as "the fix failed". Probe with `cassis ontology test -q "<the question from the occurrence>"`, which runs against the checkout.
4. **Lock it against regression before you ship it.** `cassis eval add-case` with the occurrence's question and the now-correct SQL. An issue fixed without a gold case is an issue that comes back, and the eval is what proves the fix to whoever reviews the PR.
5. **Reference the issue in the PR body** (id and link). The PR is where a reviewer reconstructs why this change exists.
6. **Close it after the merge publishes, not after the commit.** `update_issue_status` → resolved. Until the merge syncs, the fix is not live and the issue is still true. Closing early makes the queue lie.
7. **One issue per PR** wherever the fixes are independent. A bundled PR is hard to review and impossible to revert one piece of.

## Validate after every batch

```bash
cassis ontology fmt && cassis ontology check
```

Fix until green. Then probe with at least one real question the change should now answer:

```bash
cassis ontology test -q "<question>"
```

Show the user the question and the outcome, in the final summary as well as inline. If the answer is wrong, fix the context, not the question.

**When the probe fails for infrastructure reasons** (504, timeout, transport error, no project id), the probe did not pass: it did not run. Say so plainly to the user and carry it into the delivery summary and the PR body as an unverified change. Do not promote `check` to "the gate that matters" and move on. `check` validates that the files parse and import; only the probe shows the agent can answer with them. Retry once, then report the failure and let the user decide whether to ship unverified.

## Eval capture

Whenever the user (or an expert they consulted) confirms an answer or a definition, propose locking it in:

```bash
cassis eval add-case -q "<the confirmed question>" --gold-sql "<the correct SQL>"
```

Both flags are required; there is no interactive prompt.

Before opening the PR, if the project has an eval suite:

```bash
cassis eval run
```

A regression here blocks the PR until resolved or explicitly waived by the user.

## The unknowns gate (the second hard stop)

Before you commit, post the TODO list in the conversation and **stop**, exactly as at the structure gate. Same halt, same reason.

Most unknowns are discovered while filling, not while scoping, so the user has never seen them. Anything you flagged after the structure gate is new information to them. Shipping it straight into a PR body means the one person who can answer these reads them for the first time in a review, next to a diff that already assumes the answers.

The list is a **file first**, `TODO-ontology.md` at the repo root, committed with the change. Posting the table in chat is how you run the gate, not where the list lives: a description that says "unverified, see TODO" against no TODO file is a dangling pointer, and chat is gone by the time someone reads the diff. Write the file, then post its contents.

Post the list, say which items block nothing and which would change the modeling, and wait. Every answer gets folded in as one more batch, then re-run validation. If the user says to move on or cannot answer, keep the items as they are and carry them into the PR body. "I raised two of these earlier" does not discharge the gate for the four you found afterwards.

## Deliver

1. The unknowns gate above is cleared.
2. Commit on the working branch with a message summarizing what was added and from which source.
3. Open a PR: `gh pr create` (or the repo's equivalent). Body: domains and tables touched, metrics added, the remaining flagged unknowns, probe outcome, eval results. **The commit is not the deliverable, the PR is.** If the push or the PR fails (no remote, no credentials, protected branch), report the exact failure and hand the user the command to finish it. Never end a pass with "done" when the work is sitting in a local commit.
4. The `cassis / ontology validation` check runs on the PR. The user reviews and merges; merging publishes.

Never commit to the default branch. Never edit Cassis-managed branches (created by app publishes). Never delete ontology files unless the user explicitly asks: imports are full-replace, so a deleted file deletes the object.

## References

- File format: https://docs.getcassis.com/file-format/
- Git workflow: https://docs.getcassis.com/git/
- CLI: https://docs.getcassis.com/cli/
- Complete worked example: https://github.com/GetCassis/cassis-ontology-examples/tree/main/examples/stallora/cassis
