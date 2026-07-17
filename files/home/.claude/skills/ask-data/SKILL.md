---
name: ask-data
argument-hint: "[natural language data question]"
description: |
  Answer any Docker data question by querying Snowflake live and grounding the answer
  in the data team's documented table schemas, join patterns, and real Q&A from the
  #help-data Slack channel. Use this skill whenever someone asks about product usage,
  metrics, retention, Hub, Desktop, Scout, DHI, AI/MCP, Build Cloud, Salesforce, cohort
  analysis, data modeling, debugging data issues, or wants to run a Snowflake query —
  even if they phrase it conversationally ("how many users...", "what's the trend...",
  "why is this number different...", "can you pull..."). Always invoke this skill rather
  than guessing or making up numbers.
---

# ask-data

You answer natural-language data questions by writing and executing SQL against Docker's
Snowflake data warehouse, then explaining the results in plain English. You ground every
answer in the data team's documented schemas, patterns, and real #help-data Q&A.

The user's question: "$ARGUMENTS"

---

## Step 1 — Load context

Before writing any SQL, read the relevant reference files. This is not optional — the
files contain critical table schemas, join patterns, and known pitfalls that are not
in your training data.

**Always read (every question):**
- `${HIVEMIND_ROOT}/team-workspaces/data_analysts/skills/snowflake/reference/_shared.md` — core entities,
  join patterns, internal user filtering, effective plan tier logic

**Read based on topic detected in the question:**

| Topic | File to read |
|-------|-------------|
| Hub pulls, pushes, orgs, seats, plan tiers, subscriptions | `${HIVEMIND_ROOT}/team-workspaces/data_analysts/skills/snowflake/reference/hub.md` |
| Docker Desktop, heartbeat, CLI, Gordon, Desktop instances | `${HIVEMIND_ROOT}/team-workspaces/data_analysts/skills/snowflake/reference/desktop.md` |
| Docker Scout, vulnerability scanning, enabled repos | `${HIVEMIND_ROOT}/team-workspaces/data_analysts/skills/snowflake/reference/scout.md` |
| DHI, Docker Hardened Images, image pulls (free tier) | `${HIVEMIND_ROOT}/team-workspaces/data_analysts/skills/snowflake/reference/dhi.md` |
| Docker AI Agent, MCP events, MCP catalog | `${HIVEMIND_ROOT}/team-workspaces/data_analysts/skills/snowflake/reference/ai.md` |
| Docker Build Cloud | `${HIVEMIND_ROOT}/team-workspaces/data_analysts/skills/snowflake/reference/cloud.md` |
| Cross-product adoption rollups | `${HIVEMIND_ROOT}/team-workspaces/data_analysts/skills/snowflake/reference/products.md` |
| Insights dashboard, customer-facing metrics | `${HIVEMIND_ROOT}/team-workspaces/data_analysts/skills/snowflake/reference/insights.md` |
| Salesforce, SFDC, opportunities, ARR, revenue | `${HIVEMIND_ROOT}/team-workspaces/data_analysts/skills/snowflake/reference/salesforce.md` |
| Retention, cohorts, churn, reactivation | `${HIVEMIND_ROOT}/team-workspaces/data_analysts/skills/analysis/cohort-retention/SKILL.md` |

---

## Step 2 — Clarify if needed

If the question is ambiguous, ask one focused clarifying question before writing SQL.
Examples of ambiguity:
- "users" — Hub users, Desktop users, or Scout users?
- "active" — any event? a specific action?
- "last month" — calendar month or trailing 30 days?
- "our customers" — include free tier? internal Docker users?

If the question is clear enough to proceed, don't ask — just go.

---

## Step 3 — Write and execute SQL

**Execution:**

Use the `mcp__claude_ai_snowflake-sql__sql_exec_tool` MCP tool to run SQL. If the tool
is unavailable, prompt the user to connect first with `/mcp` and select the
`snowflake-sql` connection. Always execute — don't just provide SQL for the user to run.

**Query conventions:**
- Always exclude the current incomplete period: `< DATE_TRUNC('{granularity}', CURRENT_DATE)`
- Always exclude internal Docker users unless explicitly asked (see `_shared.md` for pattern)
- Use `LIMIT 100` for exploratory queries
- Prefer CTEs over subqueries for readability
- Use `ILIKE` for case-insensitive string matching
- Fully qualify all table names: `DATABASE.SCHEMA.TABLE`
- For large tables (>100M rows), sample first: `FROM table SAMPLE SYSTEM(1)`
- Use `DESCRIBE TABLE` to explore unfamiliar tables before querying

**Safety — hard rules, never bypass:**
- NEVER run `DROP`, `TRUNCATE`, `DELETE`, `UPDATE`, `ALTER`, `MERGE`, or `INSERT`
- If a user asks for a destructive operation, refuse and explain why

**Schema discovery when needed:**
```sql
SHOW SCHEMAS IN DATABASE <db>;
SHOW TABLES IN SCHEMA <db>.<schema>;
SHOW TABLES LIKE '%pattern%' IN DATABASE <db>;
DESCRIBE TABLE <db>.<schema>.<table>;
```

---

## Step 4 — Summarize results

- Lead with the number or finding that directly answers the question
- Summarize large result sets — don't dump raw rows
- Flag any caveats (internal users couldn't be fully filtered, date range assumptions, known data gaps)
- Suggest a follow-up query if the results raise an obvious next question

---

## Step 5 — Write to Google Drive (if requested)

After summarizing results, offer to write a formatted report to Google Drive using `gws`:

```bash
# Create a Google Doc
gws docs create --title "Ask Data: [topic] - [date]" --body "[formatted markdown content]"

# Create a Google Sheet (for tabular data)
gws sheets create --title "Ask Data: [topic] - [date]"

# Upload a local file
gws drive upload <file>
```

Run `gws login` first if not yet authenticated (browser OAuth prompt).

---

## Known pitfalls (apply automatically)

1. **Internal user filtering** — always join to `workspaces.canonical.dim_user` and filter `is_docker_internal_email = FALSE`, or use the pattern in `_shared.md`. Don't rely on email domain alone.

2. **Incomplete current period** — always exclude the current in-progress day/week/month with `< DATE_TRUNC(...)`. Forgetting this makes current-period numbers look artificially low.

3. **Scout: API vs FE columns** — Scout user counts must use API-sourced columns (`num_scout_image_analysis_api_calls > 0`), not FE columns which are systematically undercounted. See `scout.md`.

4. **Snapshot/non-additive columns** — columns ending in `_eom`, `_eow`, `_eod` are point-in-time snapshots. Never `SUM` them across periods — use `MAX` or pick the most recent row.

5. **Desktop unit of analysis** — `desktop_instance_uuid` counts devices; `hub_user_uuid` counts people. Decide which is right before writing the query. See `desktop.md`.

6. **Plan tier classification** — use the `effective_plan_tier` CTE pattern with `fct_org_user_mapping + dim_org` for clean Free/Pro/Team/Business breakdown. Don't use `core_product_name` directly — it produces fragmented values.

7. **SCD2 temporal joins** — when joining slowly-changing dimension tables, use: `enabled_at < period_end AND (disabled_at IS NULL OR disabled_at >= period_start)`.

8. **Hub repo table selection** — use `mart.docker_entities.l3_dim_hub__repositories` for reporting. Avoid querying `prep` or `intermediate` layers directly.

9. **dbt layer usage** — prefer `mart.*` or `presentation.*` for analysis. Use `prep.*` or `intermediate.*` only when mart tables don't have what you need, and flag this in your response.

---

## Retention / cohort questions

If the question involves retention, churn, reactivation, or cohort analysis, read
`${HIVEMIND_ROOT}/team-workspaces/data_analysts/skills/analysis/cohort-retention/SKILL.md`
before writing SQL.

---

## dbt layer naming

| Prefix | Layer | When to use |
|--------|-------|-------------|
| `l1_` | Raw/staging | Avoid — use only if mart doesn't have it |
| `l2_` | Intermediate/cleaned | Avoid — use only if mart doesn't have it |
| `l3_` | Mart — broad business models | Default for reporting |
| `l4_` | Mart — BU-specific, terminal models | Use for team/product-specific reporting |
| `pres_` | Presentation | Use for dashboard-optimized queries |
