---
name: octopus-audit-report
description: Use this skill when a user asks to produce audit reports or compliance evidence from Octopus Deploy event data through the Octopus MCP server. It covers questions about who deployed, who edited which variables, when a release shipped, why a permission changed, and what activity happened in a Space over a given window. Trigger on phrasing such as audit report, audit trail, change log, who changed what, who deployed, compliance evidence, SOC 2, GDPR, HIPAA, security investigation, access review, user activity, or runbook history.
---

## Mental model

Every action in Octopus Server — human or automated — emits an event. Events sit alongside the resources they touch, scoped to the Space that owns the resource, and they're permanent: the MCP server can read and filter them but never rewrite them. An audit report is a slice of that stream, narrowed by time, scope, principal, or document, then reshaped into something an auditor or incident responder can act on.

Three MCP tools live around the audit log. Reach for them in this order:

- `find_events` is the audit log itself. One tool covers search, single-event lookup, and four discovery modes that enumerate categories, groups, document types, and user agents. Start here for every audit task.
- `grep_task_log` and `find_interruptions` cover task execution traces and approval activity. Pull them in after `find_events` tells you which specific task or interruption matters.

## Approach

Work outside-in: pin the scope, learn the vocabulary, page through the stream, then re-fetch detail for the entries that need it.

1. **Pin the scope.** Audit data is per-Space. If the user didn't name one, call `list_spaces` and ask. Cross-Space audits need one pass per Space — there's no instance-wide search.
2. **Learn the vocabulary.** Category and group names vary by Octopus version, so don't hard-code them. Hit the discovery modes first:
   - `find_events(mode: "listCategories")` lists every category the server recognizes, such as `DeploymentSucceeded` or `VariableUpdated`.
   - `find_events(mode: "listGroups")` lists the coarser bands (`Created`, `Modified`, `Deleted`, `Deployment`, `Interruption`, ...) and the categories each one contains.
   - `find_events(mode: "listDocumentTypes")` returns the entity prefixes (`Projects-`, `Releases-`, ...).
   - `find_events(mode: "listAgents")` returns the user-agent strings the audit subsystem has seen.

   Metadata modes ignore `spaceName`; they're server-wide.
3. **Run the search.** A search call sets `spaceName`, a window (`from` inclusive, `to` exclusive, both ISO 8601), any filters from the next section, `excludeDifference: true` for the bulk pass, and `includeInternalEvents: false` to drop machine-housekeeping noise. Default `take` is 30; raise it to 100 for reports.
4. **Page to the end.** Keep looping while `items.length === take` or `skip + items.length < totalResults`, bumping `skip` by `take` each iteration. Stopping early is the most common cause of an audit trail that looks complete but isn't.
5. **Re-fetch detail where it matters.** `ChangeDetails` (the before-and-after diff) is the heaviest field per event, so the bulk pass should strip it. When the report needs a specific diff — usually for `VariableUpdated`, deployment-process edits, or permission changes — call `find_events` again with that single `eventId` and `excludeDifference: false`.
6. **Shape the output.** Pick the layout from [`references/report-templates.md`](references/report-templates.md) that matches the user's question: activity overview, per-principal investigation, incident timeline, configuration history, or framework-mapped compliance evidence.

## Choosing filters

Use the coarsest filter that still answers the question — short queries are easier to read and easier to defend.

**By activity class — `eventGroups`.** Five groups carry most audit work:

| Group | What it covers |
|-------|---------------|
| `Created` | New resources of every kind |
| `Modified` | Edits to existing resources |
| `Deleted` | Resource removals |
| `Deployment` | Deployment lifecycle: queued, started, succeeded, failed |
| `Interruption` | Manual interventions and approvals |

**By specific event — `eventCategories`.** Use when the user asks about a particular kind of activity:

| Category | Typical use |
|----------|-------------|
| `DeploymentSucceeded`, `DeploymentFailed`, `DeploymentQueued` | Deployment outcomes |
| `ReleaseCreated`, `ReleasePublished` | Release lifecycle |
| `VariableCreated`, `VariableUpdated`, `VariableDeleted` | Variable changes |
| `RunbookRunSucceeded`, `RunbookRunFailed` | Runbook execution |
| `UserCreated`, `UserUpdated`, `UserDeleted` | User management |
| `LoginSucceeded`, `LoginFailed` | Authentication |
| `ApiKeyCreated`, `ApiKeyRegenerated`, `ApiKeyDeleted` | API key lifecycle |

This isn't the full set — confirm against `mode: "listCategories"` on the target server before relying on a name.

**By document kind — `documentTypes`.** Accepts entity prefixes like `["Projects-", "Variables-"]`.

**By specific document — `regarding` or `regardingAny`.** `regarding` requires the event to reference every ID listed (AND). `regardingAny` matches when the event references any one of the IDs (OR). Reach for `regardingAny` to correlate a set of releases or deployments; `regarding` when you need an event that links a particular pairing — say, a specific release deployed to a specific environment.

**By principal — `users`, `tenants`, `tags`.** Tags take canonical form like `release-rings/beta`. A `users` filter returns every event for that user ID regardless of authentication method; separate interactive sessions from automation by inspecting `IsService` and `IdentityEstablishedWith` on the returned events.

## Reading the results

Each event payload carries:

- `Occurred` — UTC timestamp; sort by this to build timelines
- `Category` — the specific event kind
- `UserId`, `Username`, `IsService`, `IdentityEstablishedWith` — who triggered it and how
- `IpAddress`, `UserAgent` — where it came from
- `RelatedDocumentIds` — the resources the event touches; group by these to track one document's change history
- `Message` — human-readable summary, usually safe to quote verbatim
- `ChangeDetails` — before-and-after diff, populated only when `excludeDifference` is false

Patterns worth surfacing in the report itself:

- A run of `LoginFailed` events ending in `LoginSucceeded` for one principal — credential probing
- `ApiKeyCreated` followed by a burst of activity under that principal — fresh credential being exercised
- `VariableUpdated` events touching production-scoped variables outside a deployment window
- Role or team membership changes that grant production access
- Activity originating from a user agent or IP range not seen earlier in the window

## Examples

**Default Space, last 30 days.** Resolve the Space, run `mode: "listGroups"` to confirm the group set, then paginate a search over a 30-day window with `excludeDifference: true` and `includeInternalEvents: false`. Render with the activity-overview layout.

**One user, last week, Production Space.** Map the email or display name to a user ID (try the Octopus API's user list if `find_events` doesn't return a match on `Username`). Search with `users=["Users-NN"]` and render with the per-principal layout. Note whether the user's activity also includes API-key events that originated from them.

**Pipeline and variable activity in one project this month.** Resolve the project ID (for example `Projects-42`). Set `regarding=["Projects-42"]` and `eventCategories` to the deployment and variable categories you care about. Render with the configuration-history layout, fetching diffs on demand for the variable changes worth showing.

**Failed logins and API key creation, last week.** Two narrow queries: one with `eventCategories=["LoginFailed"]`, one with `eventCategories=["ApiKeyCreated", "ApiKeyRegenerated"]`. Cross-reference by user and IP and render with the investigation layout.

**SOC 2 evidence for Q4, Production Space.** Use the framework mapping in [`references/report-templates.md`](references/report-templates.md) to identify the event groups and categories that back each control, run one query per control, and stitch the output into a quarter-scoped compliance report.

## Symptoms and fixes

| What you see | Usual cause | What to do |
|--------------|-------------|------------|
| Empty result set | Window too narrow, or wrong Space | Widen `from`/`to`; cross-check `spaceName` with `list_spaces` |
| 403 from the tool | Caller lacks `EventView` on the Space | Ask for `EventView` on the target Space, then retry |
| A user's activity is missing | ID format mismatch, or activity carried by an API key | Try username, email, and display-name variants; inspect events with `IsService: true` or `IdentityEstablishedWith: "ApiKey"` |
| Trail looks short | Pagination stopped too early | Loop on `skip` until the returned page is shorter than `take` |
| Empty `ChangeDetails` | Bulk pass set `excludeDifference: true` | Re-fetch the single event by `eventId` with `excludeDifference: false` |

## Performance pitfalls

- `ChangeDetails` is by far the heaviest field on an event payload. Strip it on the bulk pass and re-fetch single events for the diffs you actually need to show.
- `includeInternalEvents: false` removes machine-housekeeping events (`MachineAdded`, `MachineDeleted`, `MachineDeploymentRelatedPropertyWasUpdated`) that otherwise drown out human activity on instances with many polling targets.
- Every claim in the final report needs to map back to a real event in the log. Don't infer activity that isn't there.

## Tone and style

Follow the Octopus Deploy writing guide. Use active voice, plain English, US spelling, the Oxford comma, and contractions like "don't", "can't", "you'll". Prefer "lets" over "allows", "use" over "leverage", "after" over "once", "while" over "whilst", "deactivate" over "disable". Capitalize Octopus Server, Octopus Cloud, Tentacle, Spaces (the feature), and PowerShell. Lowercase deployment target, environment, worker, and worker pool. Spell it on-premises, ID (not Id), Redeploy, and Rerun. Use sentence case for headings; bold project, environment, and entity names; use numerals for 2 and above and spell out zero and one; avoid exclamation marks.

## Asking for clarification

Don't guess at the Space or the time window — if the user named neither, ask. For an investigation, ask whether the user already has a candidate principal or document in mind, or whether the report needs to surface anomalies cold; the shape of the search differs in each case.
