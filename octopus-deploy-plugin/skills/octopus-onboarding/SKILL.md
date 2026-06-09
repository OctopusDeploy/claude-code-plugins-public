---
name: octopus-onboarding
description: Guides a user through their first Octopus Deploy setup — connecting a code repository, registering a deployment target (Kubernetes, Azure App Service, AWS ECS, Lambda, on-prem Tentacle, etc.), wiring up packages from their existing CI, and reaching their first real deployment. Prefers the Octopus MCP server's tools and resources when connected, and falls back to direct Octopus REST API calls when not. Trigger on "set up Octopus", "deploy my service", "connect my repo to Octopus", "first Octopus deployment", "Octopus project setup", "onboard to Octopus", "configure Octopus deployments", or any conversation where a user has Octopus credentials in hand and wants to stand up a working deployment from scratch — even if they don't use those exact phrases.
---

# Octopus onboarding

This skill walks an AI agent through a user's first end-to-end Octopus Deploy setup: connect a repo, register a target, wire up packages from CI, and reach a first real deploy. It assumes the user has an Octopus instance reachable and credentials in hand. It does **not** cover authoring Config-as-Code (`.ocl`) files — for that, use the sibling `writing-ocl` skill.

## Mental model

Three things to hold in your head at once:

1. **Octopus does not build artifacts. Something else does** — GitHub Actions, GitLab CI, Jenkins, Azure Pipelines, etc. The agent's job is to detect the existing CI, learn where it pushes its output, and connect Octopus to that, *without modifying the CI config*.
2. **Every onboarding has the same shared spine** — orient, connect cloud, register target — then diverges based on what the user actually wants to accomplish.
3. **Two steps are universally guidance-welcome** (registering a target, defining a deployment process). **Two steps are universally guidance-hostile** (CI integration, the first real deploy). Automate accordingly: the first kind, walk the user through; the second kind, hand them a snippet and step back.

Vocabulary is a known landmine — *tenant*, *space*, *channel*, *runbook*, *lifecycle* all confuse new users. Translate before you ask. See `references/vocabulary-and-deferrals.md`.

## Before you touch anything

### 0.0 Are you resuming a prior session?

If the user has run this skill against this space before, don't 409 on collisions. Before creating anything, check `list_projects`, `list_environments`, `find_deployment_targets`, and the feed list for objects with matching names; treat each pre-existing match as `✓ Reused existing …` (see *Announcing progress to the user* below). The PLAN summary (§0.8) must distinguish new resources from reused ones. Octopus state — not a local cache file — is the source of truth.

### 0.1 Establish two server URLs

You need both:

- **The URL *you* call** to talk to Octopus. Check (in order) `$OCTOPUS_URL`, `$OCTOPUS_CLI_SERVER`, `~/.octopus/config`, `.octopus/octopus.config`. If none, ask: *"What's your Octopus server URL?"*
- **The URL the *cluster or deployment target* will use** to reach Octopus. Usually the same, but not always:
  - Octopus runs on the user's laptop and a Kubernetes agent will run in **Docker Desktop** → agent URL is `http://host.docker.internal:<port>`.
  - Octopus on a LAN box, cluster is remote → agent URL is the LAN hostname/IP, not `localhost`.
  - Self-hosted Octopus behind a corporate firewall + a cloud cluster → the agent URL must be publicly reachable. Flag this *before* starting the agent install.
  - Octopus Cloud → same URL for both.

Record both. Call them `serverUrl` (yours) and `agentReachableUrl` (the target's).

### 0.2 Detect transport — MCP or direct REST?

The user may have the official Octopus MCP server connected (tools named `list_spaces`, `find_releases`, `create_release`, `deploy_release`, `run_runbook`, `grep_llms_txt`, `read_resource`, `execute`, etc.; resources under `octopus://...`). If they do, **prefer it** — every read in this skill has a tighter, validated MCP equivalent, and writes are gated through the elicitation flow rather than raw HTTP.

If the MCP server isn't connected, fall back to direct REST with `X-Octopus-ApiKey: <key>` in the header. The user must have an API key (Profile → My API Keys in the Octopus UI).

Detection heuristic: if you see an `octopus-deploy-mcp` (or similarly named) server in your tool list with `list_spaces` exposed, it's connected. Otherwise, REST.

The full MCP-tool-to-REST-endpoint cross reference, including which tools require `--no-read-only` and which gate via elicitation, is in `references/mcp-and-api.md`. Read that file once at the start of an onboarding session.

### 0.3 Load the API spec

Octopus exposes a markdown catalog of every REST endpoint, every step type, and the full `Octopus.Action.*` property catalog. Treat this as the source of truth — **never invent property keys; always look them up**.

- **MCP path**: dereference `octopus://api/llms.txt` via `read_resource` (or `resources/read`). For targeted lookups (e.g., "what properties does `Octopus.HelmChartUpgrade` take?"), use `grep_llms_txt` with a pattern — the catalog is ~350 KB and a full read wastes context.
- **REST fallback**: `GET {serverUrl}/api/experimental/llms.txt` with the API key header.

Also useful via MCP only: `octopus://api/capabilities` reports the running Octopus version, enabled toolsets, and feature flags. Check it once if you suspect a version mismatch (e.g., `get_kubernetes_live_status` requires Octopus 2025.3+).

### 0.4 Pick a space

A *space* is a tenancy boundary inside an Octopus install — call it a "workspace" when talking to the user. If they only have one, use it silently and don't mention spaces at all.

- **MCP**: `list_spaces` → pick the `Id` (`Spaces-1`).
- **REST**: `GET /api/spaces/all`.

If the user has several, ask only by display name (*"Which workspace — Default, Payments, or Data Platform?"*). Never ask the user to type a space ID.

Use the `/api/{spaceId}/...` prefix in all space-scoped REST calls. Three endpoints take `SpaceId` in the body instead of the prefix: `POST /api/tasks`, `POST /api/users/invitations`, `POST /api/migrations/*`. When in doubt, check the spec's "Prefixes (pick one)" marker.

### 0.5 Ask the one routing question

> "What are you trying to accomplish here — get a single service deployed quickly, set up a pipeline with approvals before production, or stand up shared deployment infrastructure that several teams or customers will use?"

Three branches, picked by the user's own words:

| The user said… | Default posture |
|---|---|
| "Just get my service deployed" / "I want to deploy this app" | Optimize for time-to-first-deploy. Accept sensible defaults. Skip multi-tenancy, channels, process templates unless asked. → **§3A** |
| "I want a pipeline with environments and approvals" / "We promote through Dev → Staging → Prod" | Add approval gates, multi-environment lifecycle, rollback. Assume the result will be handed to other developers afterward. → **§3B** |
| "We're setting up a platform for several teams" / "I'm building shared deployment infrastructure" | Start by modelling organizational structure (tenants, tag sets, project groups, Git-backed config). Don't deploy anything until the model is right. → **§3C** |

The branches share the same shared spine (§1) and the same build-output handoff (§2). Only the post-spine sequence differs.

### 0.6 SCAN — read the repo silently

If a local repo is open, detect what you can without asking. Use Glob/Grep (or the equivalent file-listing tool) up to 3 levels deep. Don't ask the user what's in their own repo if a signal file is sitting there.

**Signal files to look for** (one pass, then stop — don't re-scan per section):

| Stack | Signal files |
|---|---|
| Containers | `Dockerfile`, `docker-compose.yml`, `compose.yaml` |
| Node.js / TypeScript | `package.json`, `pnpm-workspace.yaml`, `turbo.json` |
| .NET | `*.csproj`, `*.sln`, `*.fsproj`, `*.vbproj` |
| Java / JVM | `pom.xml`, `build.gradle`, `build.gradle.kts`, `settings.gradle` |
| Python | `requirements.txt`, `pyproject.toml`, `Pipfile`, `setup.py` |
| Go | `go.mod` |
| Ruby | `Gemfile` |
| Rust | `Cargo.toml` |
| PHP | `composer.json` |
| Kubernetes | `Chart.yaml`, `kustomization.yaml`, `**/k8s/*.yaml`, `**/manifests/*.yaml` |
| Serverless | `serverless.yml`, `template.yaml` (SAM), `samconfig.toml` |
| Terraform / OpenTofu | `*.tf`, `terraform.tfvars`, `*.tfvars.json` |
| Static sites | `next.config.*`, `nuxt.config.*`, `gatsby-config.*`, `astro.config.*`, `vite.config.*` |
| CI | `.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`, `azure-pipelines.yml`, `.circleci/config.yml`, `bitbucket-pipelines.yml`, `buildkite.yml`, `cloudbuild.yaml` |

Always exclude: `node_modules/`, `.git/`, `vendor/`, `bin/`, `obj/`, `target/`, `dist/`, `build/`, `out/`, `.next/`.

**Build a stack model with three dimensions:**

1. **Language / framework** — one or more (e.g., `Node.js (Express)`, `.NET 8`, `Python (FastAPI)`, `Go`, `Java (Spring Boot)`).
2. **Deployment artifact** — what gets shipped (`Container`, `Helm chart`, `JAR`, `NuGet`, `Lambda zip`, `Static site`, `Raw scripts`, `Terraform`).
3. **CI system** — detected from the CI signal files above, or `none detected`.

Also infer: **project name** from `package.json`, `*.csproj`, `Cargo.toml`, `pyproject.toml`, or repo folder name. Offer it as the Octopus project name rather than asking. **Target hints** — Kubernetes manifests, ECS task definitions, Azure App Service config, Lambda templates.

**Print a one-line summary** in this exact format:

```
Stack detected: <language(s)> · <deployment artifact> · CI: <system>
```

Examples:
- `Stack detected: .NET 8 · Container · CI: Azure Pipelines`
- `Stack detected: Python (FastAPI) · Lambda (SAM) · CI: GitHub Actions`
- `Stack detected: Go · Helm chart · CI: none detected`
- `Stack detected: Node.js · Container + docker-compose · CI: GitHub Actions`

Present the detected stack as a **statement**, not a question: *"Looks like a Helm chart built by GitHub Actions and pushed to GHCR, deploying to Kubernetes — correct?"* The user confirms or corrects.

Full CI detection table (push step → registry coordinates → Octopus `FeedType`) is in `references/build-output-and-feeds.md`.

If any of the ambiguity triggers in §0.7 fire, ask **before** confirming the stack. Otherwise proceed to §0.8.

### 0.7 CLARIFY — when to ask before guessing

Default posture is silent detection (§0.6). But when the signals genuinely conflict or are absent, asking once is much cheaper than guessing wrong. **Ask one question at a time**, A/B/C format where possible.

| Trigger | Ask |
|---|---|
| `Dockerfile` **and** `docker-compose.yml`/`compose.yaml` both present | "Which are you planning to deploy with — the standalone container image, or the compose stack?" |
| Multiple `Dockerfile`s at different paths | "I see N services with Dockerfiles (`<path1>`, `<path2>`, …). Which one should I configure first? We can add the others after." |
| Kubernetes manifests/`Chart.yaml` **and** `docker-compose.yml` both present | "Is this deploying to Kubernetes, or running as docker-compose?" |
| No language signal files detected at all | "I couldn't detect a language from the repo. What is this app — language and framework?" |
| Terraform present with no deployment artifact (no container, Helm chart, Lambda zip, etc.) | "Is Octopus deploying the Terraform itself (infra-as-code), or something this Terraform provisions?" |
| Multiple top-level `*.sln` / monorepo with multiple deployables | "Multiple projects detected (`<list>`). Which one are we onboarding first?" |
| `serverless.yml` **and** `Dockerfile` both present | "Deploying as Lambda (Serverless Framework) or as a container?" |
| CI detected but pushes to multiple registries | "Your CI pushes to `<registries>`. Which is the one Octopus should pull from?" |
| Multiple CI systems present (e.g., GitHub Actions **and** Azure Pipelines configs both committed) | "I see configs for `<systems>`. Which one is the active build path?" |

If none of these fire, **don't ask** — confirm the stack as a statement and continue. Asking unnecessary questions is one of the failure modes called out in §5 and §9.

### 0.8 PLAN — confirm before any write

Before any Octopus mutation, print a plain-English plan of every resource the agent will create *or reuse*. Wait for an explicit `yes` / `go` from the user. If the user objects, revise and re-print — don't half-execute a plan.

Order the plan from outer to inner: **space → environments → lifecycle → project → deployment target(s) → cloud account (if needed) → feed → deployment process steps → variables → release-creation source (CI snippet or manual)**.

Worked example for a Helm-on-Kubernetes onboarding:

```
Plan for "checkout-api" onboarding:
  • Workspace: Default (Spaces-1, existing)
  • Environments to create: Development, Staging, Production
  • Lifecycle: "checkout-api Lifecycle" (Dev auto → Staging manual → Prod manual + 1 approval)
  • Project: "checkout-api"
  • Cloud account: AWS account "checkout-prod-oidc" (OIDC role, new)
  • Deployment target: Kubernetes agent in namespace "checkout-prod" (new)
  • Feed: GitHub Container Registry (new, scoped read-only PAT)
  • Deployment process: 1 step — Helm Chart Upgrade ("checkout/checkout-api")
  • Variables: 2 placeholders (Image.Tag, Replicas)
  • Release creation: from your existing GitHub Actions workflow (snippet provided; you paste, we don't commit)

Confirm to proceed (yes / no / change something)?
```

Two rules for the plan:

1. **Mark resumed objects explicitly.** If §0.0 detected pre-existing matches, say so: `• Project: "checkout-api" (existing, will reuse)` vs. `• Project: "checkout-api" (new)`. Never silently mutate something that already existed.
2. **The plan is the gate.** Don't speed-run past it because the stack was unambiguous. The PLAN is also the user's last chance to swap branch (3A vs. 3B vs. 3C) before writes start.

### Announcing progress to the user

Three behavioural conventions, applied from this point forward across every stage:

- **Stage banner.** When entering a major section, announce it once: `**Stage 0 — Prerequisites**`, `**Stage 1 — Shared spine: Orient / Connect / Register**`, `**Stage 2 — Build output handoff**`, `**Stage 3A — Quick deploy**` (or 3B/3C, whichever the user picked). One banner per section, not one per HTTP call.
- **Action confirmation.** After each successful Octopus mutation, print `✓ <verb> <resource>: <Id>`. Examples:
  - `✓ Created environment 'Development': Environments-3`
  - `✓ Registered Kubernetes agent target 'web-cluster': Machines-12`
  - `✓ Updated deployment process for 'checkout-api' (4 steps)`
  - `✓ Reused existing environment 'Production': Environments-1`
  - `✓ Created release 1.4.0 for 'checkout-api': Releases-87`
  - `→ Deploy command ready (waiting for you to run it)` for output-only steps that don't auto-fire.
- **No silent writes.** Every `POST` / `PUT` / `execute` / `create_release` / `run_runbook` write call must be followed by a confirmation line. Reads (`GET`, `list_*`, `find_*`, `read_resource`) stay silent.

This convention layers *on top of* the existing posture in §5 — it doesn't replace the conversational tone of §1 Step A's "Orient" sentence, and it doesn't make `register-target` any less guided. It just makes the agent's actions legible to the user.

---

## §1 The shared spine (every onboarding)

Three steps. Always in this order. Don't skip or reorder.

### Step A — Orient

One sentence: *"I'll set up **{project name}** to deploy **{package shape}** from **{where CI pushes it}** to **{target}** in the **{space name}** workspace."*

If the user has no space, project, or environment yet, create a Default space, create Dev/Staging/Prod environments, and a Default lifecycle — silently. No product tour. No hello-world template. No sample project.

### Step B — Connect to the cloud or infrastructure

This is the first high-anxiety step. **Show the permission surface before asking for credentials.** Over-scoped credentials are the #1 early-stage blocker — senior users will stop the onboarding entirely rather than paste admin keys.

Account types (`POST /api/{sp}/accounts`, body: `CreateAccountCommand`): AWS, Azure, GCP, SSH, username/password, token, generic OIDC. Pick from the detected target — AWS ECS/EKS/Lambda/S3 → `AmazonWebServicesAccount` (or OIDC role); Azure App Service / AKS → `AzureServicePrincipal` or `AzureOidcAccount`; GCP / GKE → `GoogleCloudAccount`; on-prem → no account (register the target directly).

For each cloud, show a paste-ready minimal-scope policy *before* asking for credentials. Offer OIDC wherever the target supports it — it sidesteps long-lived credentials entirely. Full per-cloud policies and the OIDC patterns are in **`references/cloud-credentials.md`**.

### Step C — Register the deployment target

`POST /api/{sp}/machines` (body: `CreateDeploymentTargetCommand`). The `Endpoint` object is polymorphic on `CommunicationStyle`.

> **For Kubernetes, default to the Kubernetes agent.** `CommunicationStyle: "KubernetesTentacle"` — the Helm-installed polling agent. It wins on every operational axis: no long-lived service-account token; no `SkipTlsVerification` hack; outbound polling only (works for Docker Desktop, NAT'd clusters, VPN'd clusters identically); identity scoped to the install namespace. Use it unless the user has an explicit reason otherwise. The full agent install flow — reachability prerequisite, pre-create the target, Helm command, polling for `Healthy` — is in **`references/kubernetes-agent-install.md`**.

| Target | `CommunicationStyle` | Key fields |
|---|---|---|
| **Kubernetes — agent (default)** | `KubernetesTentacle` | `KubernetesAgentDetails.HelmReleaseName`, `KubernetesAgentDetails.KubernetesNamespace`, `TentacleEndpointConfiguration.CommunicationMode: "Polling"` |
| Kubernetes — direct API (legacy) | `Kubernetes` | `ClusterUrl`, `Authentication` sub-object, `DefaultWorkerPoolId` — see `references/kubernetes-agent-install.md` for the auth matrix |
| Windows/Linux Tentacle (listening) | `TentaclePassive` | `Uri` (`https://host:10933/`), `Thumbprint` |
| Windows/Linux Tentacle (polling) | `TentacleActive` | `Uri` (`poll://...`), `Thumbprint` |
| SSH target | `Ssh` | `Host`, `Port`, `Fingerprint`, `AccountId`, `DotNetCorePlatform` |
| Offline drop (air-gapped) | `OfflineDrop` | `Destination.DropFolderPath`, `ApplicationsDirectory` |
| Azure Web App | `AzureWebApp` | `AccountId`, `ResourceGroupName`, `WebAppName`, `WebAppSlotName`? |
| AWS ECS cluster | `EcsTarget` | `ClusterName`, `Region`, `AccountId` |
| Cloud region (serverless / buckets / Lambda) | `CloudRegion` | `DefaultWorkerPoolId` — no network endpoint, used for grouping |

For on-prem Tentacles, surface the prerequisite checklist *before* the install command runs (ports, service account, TLS, change-request lead time). For cloud targets, use `GET /api/{sp}/machines/discover?host=...&port=...&type=...` to pre-fill connection details where it applies.

**Verify health — never use `GET /machines/{id}/connection`** (it returns a trivial summary, not a real check). Queue a Health task instead:

```
POST /api/tasks                       ← not space-prefixed in the URL
{
  "Name": "Health",
  "Description": "Health check for Machines-123",
  "Arguments": { "MachineIds": ["Machines-123"], "Timeout": "00:05:00" },
  "SpaceId": "Spaces-1"               ← space goes in the body for this endpoint
}
→ poll GET /api/Spaces-1/tasks/{taskId} → State: Queued → Executing → Success | Failed
```

When `Success`, `GET /api/{sp}/machines/{id}` shows `HealthStatus: "Healthy"`. On failure, fetch the task details and map the error to remediation (port unreachable → firewall; TLS mismatch → cert thumbprint; 401 → credential scope; `Unable to retrieve authentication token` on a feed → registry creds). Patterns in `references/error-recovery.md`.

### Step C.1 — Worker pools (only when needed)

| Target | Needs a worker pool? |
|---|---|
| Kubernetes — agent (`KubernetesTentacle`) | **No.** Agent targets are self-executing. |
| Kubernetes — direct API (`Kubernetes`) | **Yes.** Set `DefaultWorkerPoolId` on the endpoint. |
| Tentacle / SSH target | **No.** Step runs on the Tentacle itself. |
| Cloud region / serverless / bucket steps | **Yes.** Set `DefaultWorkerPoolId` on the endpoint. |
| Offline drop | **No.** |

A fresh Octopus install may show `GET /api/{sp}/workers` empty — that's fine; the server runs a built-in worker. Don't create a worker pool reflexively.

---

## §2 Connect the build output

Octopus deploys what some other system built. Before defining a deployment process, you need to know **what builds the artifact**, **where it lands**, and **how Octopus gets told about it**. This applies to every onboarding regardless of which §3 branch you're on.

Three handoff models. Default to the one that needs the fewest CI edits:

- **Model A — External feed (pull). Preferred.** Octopus points at the registry the CI already pushes to. Zero CI changes. Use whenever CI publishes to a registry the agent can give Octopus credentials for.
- **Model B — Push to Octopus's built-in feed.** CI uploads packages directly to Octopus. One CI change (an upload step). Use when CI doesn't push durably or the user wants Octopus as the single source of truth.
- **Model C — CI-triggered release.** CI builds, pushes, then calls `POST /api/{sp}/releases/create/v1` to create a release (and optionally deploy). Use when the user wants CI to drive the whole release cadence.

The full detection table (CI system → push step → registry coordinates → Octopus `FeedType`), version-detection rules, registry credential requirements (Docker feeds **always** need credentials, even for public registries — see `references/build-output-and-feeds.md` §"registry creds"), and build-information attachment are all in **`references/build-output-and-feeds.md`**.

**The cardinal rule** for this section, repeated everywhere: *don't modify the user's CI config and don't commit to their repo.* If they need a CI snippet, output it; let them paste.

---

## §3 Branches by user intent

After the spine and §2 are done, peel off into the branch the user picked in §0.5. The technical detail for each branch — full call sequences, deployment-process step-type properties, and the worked example "deploy to two tenants via Kubernetes" — is in **`references/deployment-process-steps.md`**.

### 3A. "Get my service deployed quickly"

**Stage 3A — Quick deploy.** Goal: first real deploy in under ten steps. Skip everything that isn't load-bearing.

1. **Create project** — `POST /api/{sp}/projects`, name inferred from the repo. Default lifecycle, default project group. `TenantedDeploymentMode: "Untenanted"` unless the scenario says otherwise. Announce: `✓ Created project '<name>': Projects-<id>`.
2. **Feed** — already done in §2, unless using Model B or C. If new feed was registered there, it should already have produced `✓ Created feed '<name>': Feeds-<id>`.
3. **Configure deployment process** — `PUT /api/{sp}/projects/{projectId}/deploymentprocesses`. Pick an `ActionType` from `references/deployment-process-steps.md` based on the target. **For Kubernetes, prefer manifest-based steps** (`Octopus.HelmChartUpgrade`, `Octopus.KubernetesDeployRawYaml`, `Octopus.Kubernetes.Kustomize`) over the legacy container builder — keeps the user's manifests in the repo, where they belong. Announce: `✓ Updated deployment process for '<project>' (<N> steps: <step names>)`.
4. **Set variables** — `PUT /api/{sp}/variables/{variableSetId}`. Only what the deployment process actually references. Don't pre-create "standard sets" — they register as noise. Announce: `✓ Set variables for '<project>' (<N> placeholders)`.
5. **Create release + output deploy command** — `POST /api/{sp}/releases` then *output* (don't auto-run) `POST /api/{sp}/deployments`. Let the user fire the first deploy with their own eyes on the logs. Announce: `✓ Created release <Version> for '<project>': Releases-<id>` followed by `→ Deploy command ready (waiting for you to run it)`.

Skip: tenants, channels, runbooks, process templates, dashboards. Add when needed.

Once the first deploy succeeds, offer the ephemeral preview environments capability — see §3D below.

### 3B. "Set up a pipeline with approvals"

**Stage 3B — Pipeline with approvals.** Adds three things over 3A for a trustworthy multi-environment pipeline before any developer uses it:

1. **Create environments** — explicit Dev / Staging / Prod via `POST /api/{sp}/environments`. `AllowDynamicInfrastructure: false` on Prod. Announce one line per environment: `✓ Created environment 'Development': Environments-<id>`, etc.
2. **Create lifecycle** (call it "promotion path" or "pipeline stages" to the user) — `POST /api/{sp}/lifecycles`. Phase 1 (Dev) automatic on release; Phase 2 (Staging) manual promotion; Phase 3 (Prod) manual + `MinimumEnvironmentsBeforePromotion`. Announce: `✓ Created lifecycle '<name>': Lifecycles-<id> (3 phases)`.
3. **Add manual approval step** — add an `Octopus.Manual` step scoped to Prod via `Environments: [Prod-env-id]` and `ResponsibleTeamIds`. Create the team first with `POST /api/{sp}/teams` if needed. Announce the team creation (if any) and the step addition: `✓ Added manual approval step (scoped to Production, ResponsibleTeam: <name>)`.

Then complete the 3A steps (project, deployment process, variables, release+output deploy) using the lifecycle from step 2. Apply the same announcement convention.

CI handoff (§2) often defaults to Model C here — the user wants CI to orchestrate. Output the snippet; do not modify their CI file.

Rollback: confirm the lifecycle can see prior releases. Explain rollback via `POST /api/{sp}/deployments` to a prior release ID, or an `Octopus.DeployRelease` step with `DeploymentCondition: IfNewer`. Don't pre-configure it — explain the mechanics on request.

Once the first deploy succeeds, offer the ephemeral preview environments capability — see §3D below.

### 3C. "Stand up shared infrastructure for multiple teams or customers"

**Stage 3C — Shared platform.** Longer setup, intentionally. Users on this path tolerate a longer onboarding if it means the model holds up when many teams start using it. Each numbered step ends with the matching announcement.

1. **Model tenancy first.** Tag sets (`POST /api/{sp}/tagsets`) for each dimension; tenants (`POST /api/{sp}/tenants`) with `TenantTags`; set `TenantedDeploymentMode` on projects. Announce: `✓ Created tag set '<name>': TagSets-<id> (<N> tags)` for each, then `✓ Created tenant '<name>': Tenants-<id>` for each tenant.
2. **Git-backed configuration.** `POST /api/{sp}/projects/{id}/git/convert`. Process templates in Platform Hub: `POST /api/platformhub/{gitRef}/processtemplates`. Announce: `✓ Converted project '<name>' to Git-backed config (branch: <gitRef>)`.
3. **Project groups + scoped user roles.** `POST /api/{sp}/projectgroups`, `POST /api/userroles`, `POST /api/{sp}/scopeduserroles`. Announce one line per project group / user role / scoped role.
4. **Worker pools.** `POST /api/{sp}/workerpools` — dynamic for per-cloud execution, static for on-prem runners. Announce: `✓ Created worker pool '<name>': WorkerPools-<id>`.
5. **Shared feeds.** Create once with `POST /api/{sp}/feeds`; reference from every project's deployment process. Announce: `✓ Created shared feed '<name>': Feeds-<id>`.
6. **Process templates** instead of step-by-step processes. Share to consumer spaces via `POST /api/platformhub/{gitRef}/processtemplates/{slug}/share`. Announce: `✓ Published process template '<slug>' (v<version>)` and `✓ Shared template to space '<consumer-space>'` for each share.
7. **Compliance policies** (`POST /api/platformhub/{gitRef}/policies`) for audit trails, deploy freezes, manual gates. Announce: `✓ Created compliance policy '<name>'`.

Only after all of that, stand up a reference project the 3A way, wired through the shared template + tag model.

Once that reference project's first deploy succeeds, the ephemeral preview environments capability (§3D) is also available — though most 3C users handle PR previews per-tenant via the tag model first.

### 3D. Optional — Ephemeral preview environments for pull requests

**Stage 3D — Ephemeral previews.** *Offered at the end of any branch (3A/3B/3C), only after the first real deploy has succeeded.* Not a separate branch; an additive capability.

Ask one question: *"Want to wire up ephemeral preview environments for pull requests? This adds a Preview environment in Octopus, a teardown runbook, and a GitHub Actions PR workflow snippet you'd paste in (we don't commit to your repo)."*

If yes, follow `references/ephemeral-environments.md` end-to-end. In short, it creates:

- A `Preview` environment with `AllowDynamicInfrastructure: true`.
- A `Teardown Preview Environment` runbook (script-step skeleton — user fills in cluster-specific teardown).
- A `Preview.Url` variable placeholder.
- *Output* (not committed) a `.github/workflows/octopus-pr-preview.yml` snippet using `OctopusDeploy/create-release-action@v3` and `OctopusDeploy/run-runbook-action@v3`. GitLab CI, Azure Pipelines, and CircleCI equivalents are also in the reference.

Apply the same `✓` action-confirmation convention. The hard rule from §2 / §9 still holds: **never commit anything to the user's repo** — emit the YAML, let them paste it.

If no, stop cleanly. The user can come back to it later by re-running the skill.

---

## §5 When to guide vs. when to get out of the way

| Step | Posture | Why |
|---|---|---|
| Orient / account creation | **Quiet by default**, guided when the user signals unfamiliarity. | Users who skip past explanations treat wizards as obstacles. |
| IAM / service principal / OIDC | **Guided — always.** Show the policy *before* asking for credentials. | Over-scoped credentials are the top early-stage blocker. |
| Register deployment target | **Guided — always**, even for users who skip explanations. | Target-side config is where most setup failures happen. |
| On-prem agent install | **Guided + prerequisite checklist up front.** | Firewall and change-request lead time dominate actual wall-clock time. |
| Detect CI and pick handoff model | **Silent detection + confirm**; never edit the CI config. | CI integration is the most-resented automation surface. |
| Register feed | **Guided, quick** — confirm credentials scope. | Part of the register-target pattern. |
| Define deployment process | **Guided with stack-specific starters and dry-runs** (`GET /releases/{id}/deployments/preview/{envId}`). | Translation from the user's mental model to Octopus step properties is the deepest confidence valley. |
| Set variables | **Silent.** Only what the deployment process references. | Pre-created "standard sets" register as noise. |
| Release creation from CI | **Manual. Output snippets.** | Same rule as CI integration. |
| First real deploy | **Manual. Do not auto-trigger.** | Users need to see the first deploy work with their own eyes. |
| Rollback configuration | **Guided on request only.** Explain, don't pre-configure. | Users want to understand rollback *mechanics*; they don't want it set up unprompted. |

---

## §7 High-anxiety step playbooks

Brief inline. Detail in references.

**Cloud credentials** — generate the minimal-scope policy first; show it; offer OIDC by default; keys only on request. After `POST /accounts`, call `GET /accounts/{id}/usages` to demonstrate the scope is empty (nothing yet trusts this account). Per-cloud policies in `references/cloud-credentials.md`.

**On-prem Tentacle install** — output the prerequisite block *first*: ports (10933 inbound listening / 10943 outbound polling), service account (Windows: Local System or named account with log-on-as-service; Linux: non-root in `tentacle` group), TLS (Tentacle generates its own cert, Octopus verifies by thumbprint), and the change-request lead time reminder. Ask: *"Is all of that already in place?"* If no, stop. Only then output `Tentacle.exe register-with` / `./configure-tentacle.sh`.

**Registry credentials for the feed** — Octopus's Docker-feed handshake **does not support anonymous pull**. Empty username / password fails registration with `Unable to retrieve authentication token`, even for public GHCR / public Docker Hub / public ECR. Always supply some credentials. Per-registry minimal-scope tokens in `references/build-output-and-feeds.md`.

**First real deploy** — never call `POST /api/{sp}/deployments` automatically. Use:
1. `GET /api/{sp}/releases/{id}/deployments/template` — what *will* happen.
2. `GET /api/{sp}/releases/{id}/deployments/preview/{envId}[/{tenantId}]` — which steps run / skip.
3. Output the exact deploy command + UI deep link. Let the user fire it.

**Production deploy** — if the target environment has "prod" / "production" in its name or slug:
1. Confirm: *"This will deploy to Production. Continue?"*
2. If no `Octopus.Manual` step is scoped to that env, offer to add one.
3. `GET /api/deploymentfreezes` — if a freeze is active, report before proceeding. Don't offer to override without an explicit ask.

---

## §9 What to defer / not do

Common user frustrations. Don't do any of these unless the user explicitly asks:

- Create a "Hello World" sample project.
- Run a guided product tour.
- **Auto-generate or modify the user's CI workflow file.** Output snippets, never edit.
- **Write a new CI config from scratch.** If none exists, see `references/build-output-and-feeds.md` (§"no CI yet").
- **Commit anything to the user's repo.**
- Create default variables, tag sets, or channels "for completeness."
- Auto-trigger the first deploy.
- Ask the user to type any Octopus ID (`Spaces-1`, `Projects-2`) by hand.
- Use *tenant* / *space* / *channel* / *runbook* / *lifecycle* without grounding them in the user's domain first. See `references/vocabulary-and-deferrals.md`.

Exception: **ephemeral preview environments are explicitly offered**, not deferred — but only after a successful first real deploy on §3A/§3B/§3C. See §3D and `references/ephemeral-environments.md`.

---

## Pointer table — which reference file to read for what

| If you're handling… | Read first |
|---|---|
| Detecting MCP vs REST, picking the right tool, write gating | `references/mcp-and-api.md` |
| Cloud credentials and OIDC patterns (AWS / Azure / GCP) | `references/cloud-credentials.md` |
| Installing the Kubernetes agent (or legacy direct-API auth) | `references/kubernetes-agent-install.md` |
| Detecting the CI system, mapping to feed types, registry creds | `references/build-output-and-feeds.md` |
| Picking an `ActionType` and its property catalog (incl. worked example) | `references/deployment-process-steps.md` |
| Vocabulary translation and what to defer | `references/vocabulary-and-deferrals.md` |
| API errors and the `PUT`-is-replace rule | `references/error-recovery.md` |
| Ephemeral preview environments for PRs (Preview env + teardown runbook + CI snippet) | `references/ephemeral-environments.md` |

For full command bodies and the step-type catalog, **always** consult `octopus://api/llms.txt` (via MCP `read_resource` or `grep_llms_txt`) or the REST equivalent `GET /api/experimental/llms.txt`. Don't guess.
