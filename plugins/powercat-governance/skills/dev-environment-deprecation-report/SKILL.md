---
name: dev-environment-deprecation-report
description: |
  Builds a "Developer Environments Deprecation Report" (HTML) for any Power Platform tenant: days left until
  unmanaged developer environments are suspended, which ones are unmanaged, the apps, flows and agents at risk,
  environments created on behalf of other makers, and makers over the one-per-maker limit. Uses the Power
  Platform for Admins connectors and a bundled generator script. Use when the user asks for a "developer
  environment report", "unmanaged developer environments", "dev environment deprecation", "created on behalf
  of developer environments", "one PDE per maker" or "days left until developer environment suspension".
  Do NOT use for production, sandbox, default or Teams environment reports, or general
  inventory/usage questions — use ask-power-platform-advisor instead.
metadata:
  category: analysis
  icon: CalendarClock
---

# Developer Environments Deprecation Report

## Overview
Microsoft is retiring **unmanaged developer environments** and the Power Apps Developer Plan's unmanaged
model. This skill collects the tenant's developer environments and their apps / flows / agents through the
Power Platform admin connectors, then runs the bundled `scripts/build_report.py` so every run produces the
same report layout. It is generic: every tenant ID, connection ID, user and environment comes from the
admin's own tenant at run time — never hard-code any of them.

## When to Use
- An admin wants to know which developer environments will be suspended and what they will lose
- "How many days until developer environments are suspended?" / "Which dev envs are unmanaged?"
- "Who created developer environments on behalf of other makers?" / "Who is over one PDE per maker?"

## When NOT to Use
- Production, sandbox, default or Teams environments — this report covers **developer environments only**
- General inventory, usage, DLP or capacity questions — use the `ask-power-platform-advisor*` skills
- Changing environments (enabling Managed Environments, deleting, moving solutions) — this skill only reports
- A user without Power Platform admin rights — the admin connectors will return nothing useful; say so

## Key dates (confirm every run)
Source: Microsoft Learn, "Important changes (deprecations) coming in Power Platform" —
https://learn.microsoft.com/en-us/power-platform/important-changes-coming#deprecation-of-unmanaged-developer-environments-and-the-power-apps-developer-plan

| Milestone | Default | Script flag |
|---|---|---|
| Creation of new unmanaged developer environments ends | 2026-11-15 | `--create-cutoff` |
| Existing unmanaged developer environments suspended | 2026-12-15 | `--suspend-date` |
| Suspended environments deleted | 2027-01-15 | `--delete-date` |

Re-fetch that page with `web_fetch` at the start of every run. If any date changed, pass the new date via
the flag and tell the user. If the page can't be reached, use the defaults and say they were not re-confirmed.
The page states that from the creation cutoff each user can create only **one managed developer environment**
and existing managed developer environments are kept even when a user has more than one. It does **not** say
whether converting an extra unmanaged developer environment to managed counts against that limit — say so
whenever you advise conversion for a maker who is over the limit.

## Workflow

Full call shapes, paging and the tool-script skeleton: [references/data-collection.md](references/data-collection.md).
Write all intermediate files to `working/devenv/`.

### Step 1 — Confirm dates
`web_fetch` the Learn page above; note the three dates.

### Step 2 — Connections
`powerplatform-ListConnections` → pick a usable connection for `shared_powerplatformforadmins`
(Power Platform for Admins) and `shared_powerplatformadminv2` (Power Platform for Admins V2). If either is
missing, `powerplatform-CreateConnection` and follow its `next_step`; use `FindConnectors` /
`HydrateConnector` to confirm connector names and operation parameters if a call is rejected.

### Step 3 — Environments
`powerplatform-InvokeConnectorOperation` on `shared_powerplatformforadmins`, operation `Get-AdminEnvironment`,
params `{"api-version":"2020-10-01","$top":250}`; follow `nextLink` with `$skiptoken` until done. Save all
environments (unfiltered is fine — the script keeps `environmentSku == "Developer"`) to `working/devenv/envs.json`.

### Step 4 — Resolve people
`createdBy.displayName` and `usedBy.displayName` come back empty. Collect the distinct `createdBy.id` /
`usedBy.id` values (skip `SYSTEM`) and resolve them with `me_profile-GetMultipleUsersDetails` (fallback:
`graph-QueryGraph` `/users?$filter=id in (...)`). Save `{"<id>": "<display name>"}` to `working/devenv/users.json`.

### Step 5 — Resource counts
`InvokeConnectorOperation` on `shared_powerplatformadminv2`, operation `QueryResources`
(`api-version 2024-10-01`, table `PowerPlatformResources`), **one call per type**: apps = canvasapps,
modeldrivenapps, codeapps, apps; flows = cloudflows, agentflows, m365agentflows; agents =
copilotstudio/agents. If `totalRecords` exceeds the rows returned (1000 cap), split by
`properties.createdAt` and re-query. Run all of these in **one `core-RunScript` tool script**. Save all
`data` rows to `working/devenv/resources.json` and note the UTC time of the scan.

Gotchas: the `response` field is a JSON **string** (`json.loads` it); when `truncated` is true read the newest
`grounding/responses/InvokeConnectorOperation_*.json` created during the call.

### Step 6 — Build the report
```bash
python <skill-dir>/scripts/build_report.py \
  --envs working/devenv/envs.json --resources working/devenv/resources.json \
  --users working/devenv/users.json --today <YYYY-MM-DD> --scan-time <scan UTC ISO time> \
  [--create-cutoff …] [--suspend-date …] [--delete-date …] \
  --out working/Developer-Environments-Report.html --summary-json working/devenv/summary.json
```
`<skill-dir>` is this skill's folder (e.g. `/mnt/user-config/skills/dev-environment-deprecation-report`).
Never hand-edit the HTML; change inputs or flags and re-run.

### Step 7 — Validate and publish
```bash
python /opt/workspace-config/.github/skills/html/scripts/validate.py working/Developer-Environments-Report.html
```
Fix any error (re-run the script), then publish:
`host-CopyArtifact(surface="output", source="working/Developer-Environments-Report.html", destination="Developer-Environments-Report.html")`
and check the returned `copied` count is 1. Retry the same call on "source not visible yet".

## Report contents (what the script produces)
1. Red countdown banner — days to the suspension date (recomputed live on open via `data-target`), three
   milestone tiles, and "N of your M developer environments are unmanaged and will lose access to X apps,
   Y flows and Z agents unless converted to Managed Environments or their solutions are moved."
2. KPI tiles — developer envs, unmanaged (count, %), managed, resources at risk, created on behalf of others
   (count, %), makers over the 1-per-maker limit (+ environments beyond it).
3. Action table — unmanaged developer envs by resources at risk: created by, created on behalf of ("On behalf"
   badge), apps, flows, agents, dates, priority (High ≥40, Medium 1–39, Low 0), "Created after scan" badge.
4. Managed-vs-unmanaged donut and the "What to do before the suspension date" checklist.
5. One-PDE-per-maker table (Within limit / Over limit · kept as is / Over limit · has unmanaged) and
   "Who provisions on behalf of others" bar chart.
6. Managed developer environments table.
7. Footer: managed = `governanceConfiguration.protectionLevel == "Standard"`; counts from the resource
   inventory; envs created after the scan show 0; countdown refreshes on open.

## Output format (chat answer)
Read `working/devenv/summary.json` (never recount by hand) and answer:
1. **Lead with the countdown:** "**N days** until unmanaged developer environments are suspended (date)."
2. One sentence: N of M developer envs unmanaged; X apps / Y flows / Z agents at risk.
3. Short table of unmanaged developer envs: Environment | Created by | On behalf of | Apps | Flows | Agents | Priority.
4. On-behalf count (and top provisioners), makers over the limit (and how many have unmanaged envs).
5. Next steps (enable Managed Environments — premium licence needed; or move/export solutions; environment
   routing; one env per maker; notify the "on behalf of" users), plus the report link.

## Guardrails
- Read-only: never enable/disable Managed Environments, delete, move or modify environments, and never send
  emails or Teams messages about the findings unless the user explicitly asks in a separate request.
- No hard-coded tenant data: connection IDs, environment IDs and user names come from this run only.
- Report developer environments only — no production, sandbox, default or Teams content.
- Don't invent counts: if a query fails or stays capped after splitting, say which type is incomplete.
- Treat user names and environment details as internal data — keep them in the report and chat answer only.
- If the Learn page dates differ from the defaults, use the Learn dates and call out the change.
