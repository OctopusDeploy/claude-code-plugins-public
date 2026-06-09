# Report layouts and framework mappings

This file backs the `octopus-audit-report` skill. It covers five report scenarios and a mapping of Octopus event types to common compliance frameworks. Treat it as a structural reference, not a fill-in-the-blank template — adapt the section names and table shapes to the question the user actually asked.

Formatting reminders that apply to every report:

- Markdown intended for browser rendering
- Sentence case for every heading
- Project, environment, and entity names in bold
- Event IDs, category names, and document IDs in backticks
- Raw payload fragments and log lines in code blocks
- Fold long event tables behind `<details>` after they run past about fifty rows

## Activity overview

The default shape when a user asks for "an audit" or "a report" without narrowing the question. Lead with the scope and the headline numbers, then drill down.

Open with:

- Space name and ID
- Window in ISO 8601 (`from` / `to`) and the equivalent calendar duration
- Generation timestamp
- Event count, with the pre-suppression count called out if `includeInternalEvents: false` filtered anything

Break the events down by group:

| Group | Count | Share |
|-------|-------|-------|
| `Created` | n | n% |
| `Modified` | n | n% |
| `Deleted` | n | n% |
| `Deployment` | n | n% |
| `Interruption` | n | n% |
| Other | n | n% |

Add a "Most active principals" section listing the top users by event count, with the categories each one is responsible for. Tag service-account rows (`IsService: true`) and API-key rows (`IdentityEstablishedWith: "ApiKey"`) explicitly — auditors usually want them separated from interactive sessions.

Add a "Most touched documents" section showing which entities saw the most events, grouped by document type.

Close with a chronological event table (Occurred → Category → User → Related documents → Message). Place this inside a `<details>` block for larger reports.

## Per-principal report

For an investigation focused on one or more named users. Build one block per principal.

Each block opens with the principal's name, user ID, and a note on whether the row is a service account. Follow with these sub-sections:

- **Authentication** — `LoginSucceeded` and `LoginFailed` counts, with originating IPs and user agents listed
- **Deployments** — `Deployment` group event totals, broken down by project and environment
- **Releases** — `ReleaseCreated`, `ReleasePublished`
- **Variables** — variable lifecycle events, with the variable set or project that owns each one
- **Identity changes** — events touching `Users-`, `Teams-`, `UserRoles-`, or `ScopedUserRoles-` documents

Finish each block with a chronological listing of the principal's events, then a short risk-flags section calling out anything unusual:

- New IP ranges or user agents
- Activity outside the principal's usual working hours
- Self-administered permission changes
- High-volume sequences originating from a single API key

## Investigation report

For suspected incidents or "what just happened" questions.

Lead with an authentication-signals sub-section. Group `LoginFailed` events by user, IP, and user agent. Call out any sequence of failures followed within a short window by a `LoginSucceeded` for the same principal — that's the credential-probing pattern. Also list logins from IP addresses not seen earlier in the window.

Follow with a credential-lifecycle sub-section. For every `ApiKeyCreated`, `ApiKeyRegenerated`, and `ApiKeyDeleted`, note who created it, when, and whether the principal carried out activity in the next twenty-four hours.

Add a permissions sub-section listing events on `Users-`, `Teams-`, `UserRoles-`, and `ScopedUserRoles-` documents. Highlight changes that increased access, especially access touching production environments.

Add an anomalies sub-section covering off-hours activity (define the window explicitly so the auditor knows what "off-hours" meant), new user agents, service accounts acting outside their usual category set, and bulk deletes.

Close with a single chronological timeline that merges every event touched by the sections above.

## Configuration history

For change-window evidence and post-change review.

Open with a counts table grouped by document type:

| Document type | `Created` | `Modified` | `Deleted` |
|---------------|-----------|-----------|-----------|
| `Projects-` | n | n | n |
| `Variables-` | n | n | n |
| `Releases-` | n | n | n |
| `Deployments-` | n | n | n |
| `Environments-` | n | n | n |

For the changes that matter — typically variable updates, deployment-process edits, and channel or lifecycle changes — re-fetch each event by `eventId` with `excludeDifference: false` and quote the relevant slice of `ChangeDetails.Differences` in a code block. Keep the quoted slice tight; the full document context can run long.

Follow with a deployments-in-window table covering every `DeploymentSucceeded`, `DeploymentFailed`, and `DeploymentQueued`, with project, environment, release, and outcome columns.

If interruptions are in scope, list every `Interruption` group event with a pointer to the associated task. For the full approval payload, follow up with `find_interruptions`.

## Framework mappings

These mappings are starting points, not certifications. An auditor decides what evidence satisfies a control — this skill's job is to surface the events most commonly used as evidence.

### SOC 2

- **CC6 (Logical and physical access)** — `LoginSucceeded`, `LoginFailed`, user lifecycle events, API key lifecycle events, every event on `Teams-`, `UserRoles-`, and `ScopedUserRoles-` documents
- **CC7 (System operations)** — `Deployment` group events, `RunbookRunSucceeded`, `RunbookRunFailed`, `MachineHealthy`, `MachineUnhealthy`
- **CC8 (Change management)** — `Modified` group events on `Projects-`, `DeploymentProcesses-`, and `Variables-`; `ReleaseCreated`, `ReleasePublished`; every `Interruption` group event covering approvals

### GDPR

- **Article 30 — Records of processing** — the full event stream for the Space; events on `Tenants-` documents identify which tenants processed data
- **Article 32 — Security of processing** — the same set listed under SOC 2 CC6, plus the `Deployment` group and machine health events
- **Article 33 — Breach notification** — build the incident timeline from `LoginFailed`, `ApiKeyCreated`, permission-change events, and `Deleted` group events spanning the suspected window

### HIPAA

- **§164.308(a)(3) Workforce security** — user and team lifecycle events; role assignments
- **§164.308(a)(4) Information access management** — permission-change events; `Interruption` group events covering access approvals
- **§164.308(a)(5)(ii)(C) Log-in monitoring** — `LoginSucceeded` and `LoginFailed`, with IP and user-agent fields
- **§164.312(b) Audit controls** — the entire output of `find_events` is itself the evidence
- **§164.312(d) Person or entity authentication** — login events plus the `IdentityEstablishedWith` field on every event payload
