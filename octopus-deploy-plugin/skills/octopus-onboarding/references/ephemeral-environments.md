# Ephemeral preview environments for pull requests

Companion reference for `SKILL.md` (`octopus-onboarding`), §3D. Use this when the user has accepted the optional ephemeral-previews offer at the end of §3A / §3B / §3C.

This reference is **strictly post-first-deploy.** Don't run any of this until the user's first real deploy on the persistent environments (Dev / Staging / Prod) has succeeded. The whole point of preview environments is to mirror a working deployment process onto short-lived environments — there's nothing to mirror if the main process is broken.

## When to offer

Once, at the end of the onboarding, after the first real deploy succeeds. Ask one yes/no question:

> *"Want to wire up ephemeral preview environments for pull requests? I'll add a Preview environment in Octopus, a teardown runbook, a `Preview.Url` variable placeholder, and a GitHub Actions PR workflow snippet you'd paste in. Octopus side I do; the CI YAML I output for you to commit — we never write to your repo."*

If no: stop. The user can re-run the skill later.
If yes: proceed through the four creation steps, then the CI-side output. Apply the `✓` action-confirmation convention from `SKILL.md`.

## What gets created in Octopus

Four resources, in this order.

### 1. The Preview environment

```
POST /api/{sp}/environments      (or MCP `execute` with resource "environments")
{
  "Name": "Preview",
  "Slug": "preview",
  "Description": "Per-PR ephemeral environments. Created and torn down by GitHub Actions.",
  "AllowDynamicInfrastructure": true,
  "UseGuidedFailure": false,
  "SortOrder": 99
}
→ EnvironmentResource
```

`AllowDynamicInfrastructure: true` is the load-bearing flag — without it, runbooks can't create-and-delete short-lived deployment targets in this environment. `UseGuidedFailure: false` keeps PR deploys non-blocking on failure (a broken PR shouldn't pause a queue).

Announce: `✓ Created environment 'Preview': Environments-<id>`.

If the environment already exists from a prior run, `find_environments` / `GET /api/{sp}/environments?partialName=Preview` will surface it. Reuse it (`✓ Reused existing environment 'Preview': Environments-<id>`) rather than failing on 409.

### 2. Extend the project's lifecycle

Two options. Pick based on what 3A/3B set up.

**Option A — append Preview as an optional phase to the existing lifecycle.** Preferred when the user has a single project-specific lifecycle from §3A or §3B.

```
GET /api/{sp}/lifecycles/{lifecycleId}             ← fetch current
# Mutate the response: append to Phases array
PUT /api/{sp}/lifecycles/{lifecycleId}             ← whole-object replace (PUT is replace, not patch)
{
  ...existing fields preserved...,
  "Phases": [
    ...existing phases...,
    {
      "Name": "Preview",
      "OptionalDeploymentTargets": ["Environments-<previewId>"],
      "AutomaticDeploymentTargets": [],
      "MinimumEnvironmentsBeforePromotion": 0,
      "IsOptionalPhase": true
    }
  ]
}
```

The `PUT`-is-replace rule from `references/error-recovery.md` applies — fetch first, mutate, send the whole object back.

**Option B — create a dedicated `Preview Lifecycle` and use a separate project channel.** Use only if the user runs PR previews on a different cadence (different feed, different deployment process) and doesn't want them sharing the main project's lifecycle.

For Option B, also create a channel scoped to that lifecycle via `POST /api/{sp}/projects/{projectId}/channels`, with a `VersionRange` that matches the PR pattern (e.g., `pr-*`).

Announce: `✓ Added 'Preview' phase to lifecycle '<name>': Lifecycles-<id>` (Option A) or `✓ Created lifecycle 'Preview Lifecycle': Lifecycles-<id>` + `✓ Created channel 'PRs' for '<project>': Channels-<id>` (Option B).

### 3. The teardown runbook

```
POST /api/{sp}/runbooks
{
  "Name": "Teardown Preview Environment",
  "Description": "Cleans up resources after a PR preview deployment. Invoked by the PR-close CI job.",
  "ProjectId": "Projects-<id>",
  "EnvironmentScope": "Specified",
  "Environments": ["Environments-<previewId>"],
  "MultiTenancyMode": "Untenanted",
  "RunRetentionPolicy": { "QuantityToKeep": 100, "ShouldKeepForever": false }
}
→ RunbookResource
```

Then set the process:

```
GET /api/{sp}/runbookprocesses/{runbookProcessId}    ← fetch current (created empty)
PUT /api/{sp}/runbookprocesses/{runbookProcessId}
{
  ...preserved metadata...,
  "Steps": [{
    "Name": "Teardown",
    "Properties": { "Octopus.Action.TargetRoles": "" },
    "Condition": "Success",
    "StartTrigger": "StartAfterPrevious",
    "PackageRequirement": "LetOctopusDecide",
    "Actions": [{
      "Name": "Teardown",
      "ActionType": "Octopus.Script",
      "IsDisabled": false,
      "Environments": [],
      "ExcludedEnvironments": [],
      "Channels": [],
      "TenantTags": [],
      "Properties": {
        "Octopus.Action.Script.ScriptSource": "Inline",
        "Octopus.Action.Script.Syntax": "Bash",
        "Octopus.Action.Script.ScriptBody": "#!/usr/bin/env bash\nset -euo pipefail\n# TODO: replace with cluster-specific teardown.\n# Examples:\n#   helm uninstall \"checkout-pr-#{Octopus.Release.Number}\" -n \"pr-#{Octopus.Release.Number}\"\n#   kubectl delete namespace \"pr-#{Octopus.Release.Number}\"\n#   aws ecs update-service --cluster previews --service \"pr-#{Octopus.Release.Number}\" --desired-count 0\necho \"Teardown placeholder — fill this in.\"\n"
      }
    }]
  }]
}
```

Do not invent richer teardown logic. The user knows their cluster — they fill in the Helm uninstall, namespace delete, or ECS service stop. The skill leaves a `TODO` they can complete in the Octopus UI or via Config-as-Code (see the sibling `writing-ocl` skill).

Announce: `✓ Created teardown runbook 'Teardown Preview Environment': Runbooks-<id>`.

### 4. The Preview.Url variable

```
GET /api/{sp}/variables/{variableSetId}        ← project's variable set
PUT /api/{sp}/variables/{variableSetId}
{
  ...preserved fields...,
  "Variables": [
    ...existing...,
    {
      "Name": "Preview.Url",
      "Value": "",
      "Type": "String",
      "Description": "URL of the PR preview deployment. Set by the deployment process from the runbook's output variable, or by the cluster ingress controller.",
      "Scope": { "Environment": ["Environments-<previewId>"] },
      "IsSensitive": false
    }
  ]
}
```

Empty value is intentional. The deployment process — or a Helm post-install hook, or an ingress annotation — writes the real URL at deploy time. The placeholder exists so the deployment process can reference it without erroring.

Announce: `✓ Added variable 'Preview.Url' scoped to Preview environment`.

## CI-side output (never committed)

The agent **emits** the CI YAML to the chat. The user pastes it into their repo themselves. The "don't commit to the user's repo" rule from `SKILL.md` §2 and §9 still applies — preview environments don't get a carve-out.

### GitHub Actions (default)

```yaml
# .github/workflows/octopus-pr-preview.yml
# Deploy a PR preview when a pull request is opened/synced; tear it down when the PR closes.
name: PR preview

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

permissions:
  contents: read
  id-token: write    # only needed if you use OIDC to Octopus

jobs:
  deploy-preview:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Replace with your existing build/push job, or call it as a reusable workflow.
      # We assume the image / package is published by the time this runs.

      - name: Create + deploy preview release
        uses: OctopusDeploy/create-release-action@v3
        with:
          api_key: ${{ secrets.OCTOPUS_API_KEY }}
          server: ${{ secrets.OCTOPUS_URL }}
          space: ${{ secrets.OCTOPUS_SPACE }}
          project: <project-name>
          release_number: pr-${{ github.event.pull_request.number }}-${{ github.run_number }}
          deploy_to: Preview
          variables: |
            Image.Tag=pr-${{ github.event.pull_request.number }}

  teardown-preview:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: Run teardown runbook
        uses: OctopusDeploy/run-runbook-action@v3
        with:
          api_key: ${{ secrets.OCTOPUS_API_KEY }}
          server: ${{ secrets.OCTOPUS_URL }}
          space: ${{ secrets.OCTOPUS_SPACE }}
          project: <project-name>
          runbook: Teardown Preview Environment
          environments: Preview
          variables: |
            Octopus.Release.Number=pr-${{ github.event.pull_request.number }}-${{ github.run_number }}
```

Substitute `<project-name>` with the project's name from the PLAN. The release-number pattern `pr-<num>-<run>` keeps preview releases distinct from `main` releases and gives the teardown runbook something stable to key off.

Announce: `→ PR workflow snippet ready (paste into .github/workflows/octopus-pr-preview.yml; we don't commit)`.

### Other CI systems

Same Octopus side (the four resources above). Only the CI YAML changes.

**GitLab CI** — use `rules: - if: $CI_PIPELINE_SOURCE == "merge_request_event"` for deploy, and a separate job with `rules: - if: $CI_MERGE_REQUEST_EVENT_TYPE == "merge_request_event" && $CI_MERGE_REQUEST_STATE == "closed"` (or run via merge-request close webhook) for teardown. Use the official `octopusdeploy/octopus-cli-action` or a `curl` to `POST /api/{sp}/releases/create/v1`.

**Azure Pipelines** — `pr:` trigger handles open/sync; for close, use a `repository: branchwebhook` event or a separate scheduled cleanup runbook against stale PRs. The `OctopusDeploy.octopus-deploy-build-release-tasks` extension provides `OctopusCreateRelease@6` and `OctopusRunbook@6`.

**CircleCI** — `when: << pipeline.parameters.is-pr >>` filters; rely on a separate scheduled workflow for stale-PR cleanup, since CircleCI doesn't natively fire on PR-close.

**Bitbucket / Buildkite / Jenkins** — same shape: deploy on PR open/update, teardown on PR close. All use the `OctopusServer` REST API or the `octopus` CLI (`octopus release create`, `octopus runbook run`).

For any non-GitHub CI, the agent should output the CI-side snippet only if the user asks for that flavour; otherwise default to GitHub Actions, since it's the most common pattern.

## Cleanup expectations

`AllowDynamicInfrastructure: true` is what enables the create-then-delete pattern. A typical deploy-step inside the deployment process registers a short-lived deployment target (a namespace, an ECS service, a Lambda alias), scopes it to `Preview`, and the teardown runbook calls `DELETE /api/{sp}/machines/{id}` (or the Helm uninstall, or the namespace delete) to remove it. The Preview *environment itself* is never deleted — only the per-PR machines and resources inside it.

If the user wants stale-PR cleanup beyond the close-webhook flow (PR forgotten / draft never closed), suggest a scheduled runbook that lists machines scoped to `Preview` with names older than N days, and tears them down. Don't pre-build it — explain the mechanics on request.

## Common errors

- **409 on environment creation** — Preview already exists from a prior run. Look up the existing ID, reuse it, continue.
- **Lifecycle phase already includes Preview** — same fix as above; reuse, don't re-add.
- **Runbook process `PUT` returns "field X required"** — `PUT` is replace, not patch. `GET` the runbook process first, mutate, and `PUT` the whole object. See `references/error-recovery.md`.
- **CI job fails with "package not found"** — the build/push job didn't complete, or the version pattern in `release_number` doesn't match the feed. Confirm the feed query returns the PR-tagged artifact before the create-release-action runs.
- **Teardown runbook succeeds but resources remain** — the placeholder `TODO` script is still in place. Open the runbook in the Octopus UI and replace it with the cluster-specific teardown.

## What this reference deliberately doesn't do

- Doesn't auto-detect the user's PR workflow shape — too varied; surfaces the GitHub Actions default and lists alternatives.
- Doesn't build the teardown script for the user — they know their cluster better than we do.
- Doesn't create per-PR target templates as Config-as-Code — that's the `writing-ocl` skill's job. This reference stops at the Octopus REST/MCP layer.
- Doesn't auto-trigger the first preview deploy — same rule as §7 in `SKILL.md` ("never auto-trigger the first deploy"). The user merges or pushes; CI fires; they watch.
