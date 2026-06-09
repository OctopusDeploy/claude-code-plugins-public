# MCP tools and REST equivalents

This reference accompanies `SKILL.md` (the `octopus-onboarding` skill). It tells the agent which MCP tools to prefer for each onboarding step, which REST endpoints they map to, and how write-gating works in both modes.

Read this file once at the start of an onboarding session.

## Detect transport

The user either has the official Octopus MCP server connected, or they don't.

- **MCP connected** — your tool list includes tools like `list_spaces`, `list_projects`, `find_releases`, `read_resource`, `execute`. Server name is typically `octopus-deploy` or `octopus-deploy-mcp`. Resources under `octopus://...` are dereferenceable.
- **MCP not connected** — fall back to direct REST against `{serverUrl}` with `X-Octopus-ApiKey: <api-key>` in every request header (or `Authorization: Bearer <token>` if the user is using OAuth-style tokens).

Default when in doubt: probe for `list_spaces` in your available tools. If it's there, MCP is live. If it's not, REST it is.

## The rule

When MCP is connected, **every read flow described in this skill should go through the MCP tool first**. Drop to raw HTTP only when there's no equivalent tool — and even then, prefer the MCP `execute` tool over hand-rolled HTTP, so writes still flow through the gating layer.

When MCP is not connected, use the REST endpoints listed below. Same flow, just one transport instead of two.

## Tool ↔ REST cross reference

### Reads — core and discovery

| MCP tool | Equivalent REST | Notes |
|---|---|---|
| `list_spaces` | `GET /api/spaces/all` | Pick a space ID once at session start. |
| `get_current_user` | `GET /api/users/me` | Confirms credentials and surfaces permission scope. |
| `list_environments` | `GET /api/{sp}/environments/all` | |
| `list_projects` | `GET /api/{sp}/projects/all` (paged) | |
| `get_deployment_process` | `GET /api/{sp}/deploymentprocesses/{id}` (or `/projects/{id}/deploymentprocesses`) | Supports Config-as-Code projects via `gitRef`. |
| `get_variables` | `GET /api/{sp}/variables/{variableSetId}` | Returns project + library variable set variables. |
| `get_branches` | `GET /api/{sp}/projects/{id}/git/branches` | Version-controlled projects only. Min Octopus 2021.2. |

### Reads — find tools (unified ID-or-filter shape)

Each `find_*` tool accepts either an explicit ID (returns one entity) or filter params (returns a slim list with `resourceUri` for follow-up).

| MCP tool | Equivalent REST | Filters |
|---|---|---|
| `find_releases` | `GET /api/{sp}/projects/{id}/releases` (list) or `GET /api/{sp}/releases/{id}` (single) | `projectId`, optional `version` |
| `find_runbooks` | `GET /api/{sp}/runbooks` or `GET /api/{sp}/runbooks/{id}` | `projectId`, optional `name` |
| `find_tenants` | `GET /api/{sp}/tenants` or `GET /api/{sp}/tenants/{id}` | `name`, `tags`, `projectId` |
| `find_deployment_targets` | `GET /api/{sp}/machines` or `GET /api/{sp}/machines/{id}` | `name`, `roles`, `healthStatus`, `environmentId`, `commStyle`, `isDisabled` |
| `find_certificates` | `GET /api/{sp}/certificates` or `GET /api/{sp}/certificates/{id}` | `name`, `archived`, `tenantId` |
| `find_accounts` | `GET /api/{sp}/accounts` or `GET /api/{sp}/accounts/{id}` | `name`, `accountType` |
| `find_interruptions` | `GET /api/{sp}/interruptions` or `GET /api/{sp}/interruptions/{id}` | `assignedToMe`, `taskId`, `pendingOnly` |

### Tasks and logs

| MCP tool | Equivalent REST | Notes |
|---|---|---|
| `grep_task_log` | `GET /api/{sp}/tasks/{id}/details` then client-side grep | GNU-grep-style: `pattern`, `caseInsensitive`, `invertMatch`, `fixedString`, `beforeContext`, `afterContext`, `maxCount`. Activity logs are multi-megabyte; **always prefer `grep_task_log` over fetching the whole log**. |
| (no MCP equivalent — list deployments) | `GET /api/{sp}/deployments` with filters | Use `list_deployments` if available. |
| `get_deployment_from_url` | manual ID extraction | Pass a full Octopus deployment URL → returns task ID + `taskResourceUri` + grep hint. |
| `get_task_from_url` | manual ID extraction | Same shape for task URLs. |

### Kubernetes live status

| MCP tool | Equivalent REST | Notes |
|---|---|---|
| `get_kubernetes_live_status` | `GET /api/{sp}/projects/{id}/kubernetes/live-status?environmentId=...` | Requires Octopus 2025.3+. |

### Tenant variables

| MCP tool | Equivalent REST |
|---|---|
| `get_tenant_variables` | `GET /api/{sp}/tenants/{id}/variables[/projects/{projectId}]` |
| `get_missing_tenant_variables` | `GET /api/{sp}/tenants/variables-missing` |

### Catalog and capability resources (MCP only)

These have no clean REST equivalent — the MCP server publishes them as resources under `octopus://api/...`.

| Resource URI | What it returns |
|---|---|
| `octopus://api/llms.txt` | Markdown catalog of every Octopus REST endpoint, every step type, and the `Octopus.Action.*` property catalog (~350 KB). |
| `octopus://api/capabilities` | Runtime introspection: server version, enabled toolsets, available tools, feature flags. |

REST fallback for the catalog: `GET /api/experimental/llms.txt` (same content, but you have to fetch the whole thing). **Use `grep_llms_txt` with a pattern instead** when the MCP server is connected — it searches the catalog server-side and returns matching sections, saving most of the context budget.

### Resource URIs returned by reads

`find_*` and similar tools return slim summaries with a `resourceUri` field. To fetch the full body:

| URI shape | Returns |
|---|---|
| `octopus://spaces/{spaceName}/releases/{releaseId}` | Full release: notes, packages, build info, custom fields. |
| `octopus://spaces/{spaceName}/runbooks/{runbookId}` | Full runbook: runtime policies, steps. |
| `octopus://spaces/{spaceName}/tasks/{taskId}` | Lightweight task metadata: state, timing. |
| `octopus://spaces/{spaceName}/tasks/{taskId}/details` | Full `ServerTaskDetails`: ActivityLogs tree, Progress. |
| `octopus://spaces/{spaceName}/interruptions/{interruptionId}` | Full Form definition: control types, Markdown instructions, button options (Abort/Proceed), submitted Form.Values. |

Dereference with `read_resource` (or the standard `resources/read` primitive in resource-aware clients). `read_resource` is the universal bridge — if you see an `octopus://` URI in any tool's response and don't know what to do with it, call `read_resource` with that URI.

There is intentionally **no** `octopus://.../tasks/{id}/log` resource. Use `grep_task_log` instead — task logs can be multi-megabyte and inlining them as a resource would blow the context budget.

### Writes — gated through elicitation

Write tools require the MCP server to be running with `--no-read-only` (the default is read-only — keeps random read-only sessions from accidentally mutating state). Each write tool gates via `requireConfirmation`:

1. The user can set `OCTOPUS_SKIP_ELICITATION=true` to bypass (for automation / CI). Strict string equality.
2. If the client advertises elicitation capability, the SDK emits `elicitation/create` and the user gets a confirmation prompt.
3. Otherwise, the tool surfaces a `confirm: boolean` field in its own input schema; the agent must set `confirm: true` after explicit user agreement.

| MCP tool | Effect | REST equivalent |
|---|---|---|
| `create_release` | Additive — creates a new release record. | `POST /api/{sp}/releases` (or `POST /api/{sp}/releases/create/v1` for the by-name CI-friendly form). |
| `deploy_release` | **Destructive** — mutates live state. Supports untenanted, single-tenant, and tenant-tag forms. | `POST /api/{sp}/deployments`. |
| `run_runbook` | **Destructive** — runs operational automation. Supports prompted variables, guided failure, skip/include steps. | `POST /api/{sp}/runbookruns` (shape varies by runbook). |
| `execute` | Universal HTTP escape hatch. Send any verb + path + body to any Octopus REST endpoint. **Non-GET calls are write-gated** through the same elicitation flow. | The thing it's executing. |

When you need to mutate state and there's no dedicated MCP tool (e.g., creating a feed, registering a target), **use `execute` rather than dropping out to raw HTTP** — the elicitation gate still applies, and the user keeps a single audit surface.

### Toolset filtering

The MCP server organizes tools into toolsets. The user can enable a subset via `--toolsets core,releases,deployments` on launch. Available toolsets:

`core` (always on), `projects`, `deployments`, `releases`, `runbooks`, `tasks`, `tenants`, `kubernetes`, `machines`, `certificates`, `accounts`, `interruptions`, `context`.

If a tool you expect isn't in your tool list, the user's MCP server may have it filtered out — fall back to REST or `execute` for that one call rather than asking them to reconfigure.

## Announcing writes

Per the *Announcing progress to the user* convention in `SKILL.md`, every write call (MCP or REST) must be followed by a one-line confirmation. Reads stay silent. Use the `✓` form for mutations and `→` for output-only handoffs (snippets the user pastes / commands the user runs).

| Call | Announcement |
|---|---|
| `execute({ resource: "environments", body: {...} })` succeeds | `✓ Created environment '<Name>': <Id>` |
| `execute({ resource: "projects", body: {...} })` succeeds | `✓ Created project '<Name>': Projects-<id>` |
| `execute({ verb: "PUT", path: "...projects/{id}/deploymentprocesses" })` succeeds | `✓ Updated deployment process for '<project>' (<N> steps)` |
| `execute({ resource: "machines", body: {...} })` succeeds | `✓ Registered target '<Name>': Machines-<id>` |
| `create_release` succeeds | `✓ Created release <Version> for '<Project>': Releases-<id>` |
| `deploy_release` invoked but *output-only* (snippet handed to user) | `→ Deploy command ready (waiting for you to run it)` |
| `run_runbook` succeeds | `✓ Ran runbook '<Name>': RunbookRuns-<id>` |
| Pre-existing object detected (matched by name in §0.0 resume) | `✓ Reused existing <resource> '<Name>': <Id>` |
| 409 collision on retry of a known-name resource | `✓ Reused existing <resource> '<Name>': <Id>` (look up the ID first; don't surface the 409 as an error) |

Surface the destructive call's intent **before** the elicitation prompt fires, so the user doesn't see the confirmation dialog cold. Pattern: announce what you're about to do in one line, let the elicitation prompt the user, then after success print the `✓` line.

## Practical guidance

- **Reads first, writes last.** Most onboarding is reads (list spaces, find projects, get deployment process, fetch llms.txt sections). Stay in MCP for everything read-shaped.
- **Use `grep_llms_txt` for property lookups.** Don't read the whole catalog. Pattern: "what props does `Octopus.HelmChartUpgrade` take?" → `grep_llms_txt` with pattern `Octopus.HelmChartUpgrade` and follow the section.
- **Use `grep_task_log` for failure diagnosis.** Don't dereference the full task details resource for log search.
- **Use `execute` for writes that have no dedicated tool** — feed creation, target registration, variable sets, etc. Compose the body using shapes from `llms.txt`.
- **Confirmation behaviour you should expect.** Every destructive call (`deploy_release`, `run_runbook`, non-GET `execute`) will pause for user confirmation unless `OCTOPUS_SKIP_ELICITATION=true` is set. Treat the pause as the user's last chance to bail — surface what you're about to do before triggering the call, so the elicitation prompt isn't the first time they see the action.
- **REST fallback parity.** Every flow in `SKILL.md` works via raw REST too — the MCP layer is preferred but not required. If a user is testing in an environment where the MCP server isn't running, the same `POST /api/{sp}/...` call sequences from `SKILL.md` apply.
