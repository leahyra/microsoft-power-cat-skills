# Developer Environments Deprecation Report

Produce a **Developer Environments Deprecation Report** (self-contained HTML) for any Power Platform tenant: how many days are left until unmanaged developer environments are suspended, which developer environments are unmanaged, the apps, flows and agents at risk in each one, environments created on behalf of other makers, and makers over the one-developer-environment-per-maker limit.

> **Preview:** This skill is in preview. These features are available before official release for customers to provide feedback.

## Why this skill

Microsoft is retiring **unmanaged developer environments** and the Power Apps Developer Plan's unmanaged model. Admins need to know, before the suspension date, which developer environments are affected, what they contain, and who owns them. This skill turns that into a single, repeatable report in one prompt, with the same layout on every run.

| Milestone | Default date |
|---|---|
| Creation of new unmanaged developer environments ends | 2026-11-15 |
| Existing unmanaged developer environments suspended | 2026-12-15 |
| Suspended environments deleted | 2027-01-15 |

The skill re-checks these dates against [Important changes (deprecations) coming in Power Platform](https://learn.microsoft.com/en-us/power-platform/important-changes-coming#deprecation-of-unmanaged-developer-environments-and-the-power-apps-developer-plan) on every run and uses the published dates if they have changed.

## Prerequisites

- **Microsoft Copilot Cowork** with the **Power Platform Admin (Preview)** plugin installed and connected. See [Power Platform Admin in Cowork (Preview)](https://learn.microsoft.com/en-us/power-platform/admin/power-platform-admin-cowork).
  - A Microsoft 365 admin (tenant admin or Copilot administrator) installs the plugin in the Microsoft 365 admin center under **Agents > Tools > Registry**.
  - The user connects it in Cowork under **Customize > Installed > Power Platform Admin (Preview) > Connect**, using an **OAuth** connection to **Power Platform for Admins V2**.
- The connected account must hold **Global Administrator**, **Power Platform Administrator** or **Dynamics 365 Administrator**. Installing the plugin does not grant data access.
- A **Power Platform for Admins** (V1) connection. The skill offers to create it if it is missing.

## Installation

### Copilot Cowork (recommended)

Download the `dev-environment-deprecation-report` folder and add `SKILL.md` to Cowork as a personal skill.

### From the marketplace

```
/plugin marketplace add microsoft/power-cat-skills
/plugin install powercat-governance@power-cat-skills
```

> The data-collection steps use the Power Platform admin connectors exposed by Cowork. In hosts that do not expose these connectors, the skill can't collect tenant data.

## Usage

Invoke directly with `/dev-environment-deprecation-report`, or use any of these phrases:

- *Build a developer environment deprecation report*
- *How many days until developer environments are suspended?*
- *Which developer environments are unmanaged?*
- *Who created developer environments on behalf of other makers?*
- *Which makers are over one developer environment per maker?*

## What it does

1. **Confirms the dates** from Microsoft Learn and flags any change.
2. **Checks connections** to Power Platform for Admins and Power Platform for Admins V2.
3. **Lists every environment** in the tenant and keeps the developer environments.
4. **Resolves creators and owners** to display names through Microsoft Graph.
5. **Counts resources per environment** (canvas, model-driven and code apps; cloud, agent and Microsoft 365 agent flows; Copilot Studio agents), splitting queries that exceed the 1,000-row cap.
6. **Builds the HTML report**, validates it and saves it as `Developer-Environments-Report.html`.
7. **Answers in chat** with the countdown, the at-risk environments, and next steps.

### Report contents

| Section | What it shows |
|---|---|
| Countdown banner | Days to suspension (recomputed live when the report is opened), three milestone tiles, and a one-line impact statement |
| KPI tiles | Developer environments, unmanaged (count and %), managed, resources at risk, created on behalf of others, makers over the limit |
| Action table | Unmanaged developer environments ranked by resources at risk, with creator, "on behalf of" badge, app/flow/agent counts and priority (High ≥ 40, Medium 1–39, Low 0) |
| Managed vs. unmanaged | Donut chart and a "What to do before the suspension date" checklist |
| One per maker | Makers within or over the limit, and who provisions environments on behalf of others |
| Managed developer environments | Environments already protected |

## Guardrails

- **Read-only.** The skill never enables or disables Managed Environments, deletes, moves or changes environments, and never sends emails or Teams messages about the findings unless you ask in a separate request.
- **No hard-coded tenant data.** Connection IDs, environment IDs and user names come from the current run only.
- **Developer environments only.** Production, sandbox, default and Teams environments are out of scope; use the Power Platform Admin (Preview) advisor for those questions.
- **No invented numbers.** If a query fails or stays capped, the report says which resource type is incomplete.
- **Internal data stays internal.** User names and environment details appear only in the report and the chat answer.

## Related skills

- [`create-pp-dev-env`](../create-pp-dev-env) — provision a **managed** developer environment for a maker, the replacement path for unmanaged environments flagged by this report.

## Folder structure

```
dev-environment-deprecation-report/
├── SKILL.md      # Skill definition and workflow
└── README.md     # This file
```

## Support

If you face issues with:

- **Using this skill:** Report your issue here: https://github.com/microsoft/power-cat-skills/issues. (Microsoft Support won't help with issues related to this skill, but they will help with related, underlying platform and feature issues.)
- **The Power Platform Admin (Preview) plugin or core Power Platform features:** Use your standard channel to contact Microsoft Support.

## License

See the [LICENSE](../../../../LICENSE) file for license information.
