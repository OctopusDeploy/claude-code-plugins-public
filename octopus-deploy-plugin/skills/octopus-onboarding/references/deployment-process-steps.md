# Deployment process: step types and the worked example

Companion reference for `SKILL.md` (`octopus-onboarding`) §3. Use this when picking an `ActionType` for a deployment process step, or when you need a full end-to-end call sequence to follow.

The deployment process is registered with `PUT /api/{sp}/projects/{id}/deploymentprocesses` (or via MCP `execute` with the same body). For Config-as-Code projects, use `?gitRef=<branch>` to write to a specific branch — but this skill assumes database-backed projects unless the user opts into Git mid-onboarding (see `SKILL.md` §3C).

## Picking an `ActionType`

For the property catalog of every action type (`Octopus.Action.*` keys with required/optional flags, defaults, enum values), grep the API spec — never invent keys:

- **MCP**: `grep_llms_txt` with pattern matching the action type, e.g. `Octopus.HelmChartUpgrade`, `Octopus.KubernetesDeployRawYaml`.
- **REST**: search `/api/experimental/llms.txt` for the same pattern in the `## Steps` section.

### Kubernetes — prefer manifest-based steps

If the user's repo has Kubernetes manifests, a Helm chart, or a Kustomize tree, the source of truth should stay in the repo — the Octopus step just *applies* it. That gives the user diffable, reviewable config and avoids a 25-key JSON blob inside Octopus that only a wizard can edit.

| Target | `ActionType` | When to use | Minimum-viable shape |
|---|---|---|---|
| Kubernetes — raw YAML in repo (**recommended**) | `Octopus.KubernetesDeployRawYaml` | Repo has `k8s/*.yaml` or similar | `Octopus.Action.Script.ScriptSource` = `GitRepository` or `Package`; `Octopus.Action.KubernetesContainers.CustomResourceYamlFileName` = `manifests/**/*.yaml` |
| Kubernetes — Helm chart (**recommended**) | `Octopus.HelmChartUpgrade` | Repo has `Chart.yaml` | `Octopus.Action.Script.ScriptSource` = `Package` or `GitRepository`; `Octopus.Action.Helm.ReleaseName`; `Octopus.Action.Helm.Namespace`; `Packages[0].{PackageId,FeedId}` |
| Kubernetes — Kustomize (**recommended**) | `Octopus.Kubernetes.Kustomize` | Repo has `kustomization.yaml` | `Octopus.Action.GitRepository.Source` = `Project`; `Octopus.Action.Kubernetes.Kustomize.OverlayPath` = `overlays/#{Octopus.Environment.Name}` |
| Kubernetes — container builder (legacy) | `Octopus.KubernetesDeployContainers` | Only when the user explicitly wants Octopus-authored Deployment / Service / Ingress JSON and has no manifests | See "Legacy container builder" below |

### Other targets

| Target | `ActionType` | Key properties |
|---|---|---|
| Azure App Service | `Octopus.AzureAppService` | `Octopus.Action.Azure.DeploymentType` = `Package` or `Container`; `Packages[0]` |
| AWS ECS (deploy task or service) | `Octopus.AwsRunScript` (or the dedicated ECS step where present) | `Octopus.Action.Aws.Region`, `Octopus.Action.AwsAccount.Variable` |
| AWS Lambda | `Octopus.AwsRunLambda` (per spec) or `Octopus.AwsRunScript` driving `aws lambda update-function-code` | `Octopus.Action.Aws.Region`, `Octopus.Action.AwsAccount.Variable` |
| Windows IIS / MSI on a Tentacle | `Octopus.TentaclePackage` | `Octopus.Action.Package.PackageId`, `Octopus.Action.Package.FeedId` |
| Static site to S3 | `Octopus.AwsUploadS3` | `Octopus.Action.Aws.S3.BucketName`, `Octopus.Action.Aws.Region` |
| Manual approval | `Octopus.Manual` | `Octopus.Action.Manual.Instructions`, `Octopus.Action.Manual.ResponsibleTeamIds` |
| Run a script | `Octopus.Script` | `Octopus.Action.Script.Syntax`, `Octopus.Action.Script.ScriptBody` |

Every step that references a package must point at the feed from `references/build-output-and-feeds.md` via `Octopus.Action.Package.FeedId`.

## Legacy container builder — `Octopus.KubernetesDeployContainers`

Only use this when the user explicitly wants Octopus to author the Kubernetes objects from a JSON property bag rather than from manifests. The minimum-viable property set is larger than the spec's "required" hints suggest. A working step needs most of these `Octopus.Action.KubernetesContainers.*` keys:

- `DeploymentResourceType` = `Deployment` (or `StatefulSet`, `DaemonSet`, `Job`)
- `DeploymentName` — Kubernetes deployment name; use `#{Octopus.Project.Name | Slugify}` or similar
- `DeploymentStyle` = `RollingUpdate` (or `Recreate`, `BlueGreen`)
- `Replicas` = `1` (integer, but string-encoded in the property bag)
- `Namespace` — may be blank to inherit from the target
- `Containers` — JSON-encoded array of container specs (`Name`, `Image` using `#{Octopus.Action.Package[<name>].PackageId}:#{Octopus.Action.Package[<name>].PackageVersion}`, `Ports`, `Env`, `Resources`, `LivenessProbe`, `ReadinessProbe`, `VolumeMounts`)
- `CombinedVolumes` — JSON array of volumes (empty `[]` is valid)
- `ServiceName`, `ServiceType` = `ClusterIP`, `ServicePorts` (JSON) — if the step also creates a Service
- `DeploymentLabels`, `DeploymentAnnotations`, `PodAnnotations` — JSON objects
- Any Ingress / ConfigMap / Secret subsections the user has enabled

Plus the step-level `Octopus.Action.Kubernetes.*` keys:

- `ResourceStatusCheck` = `True` — verify the deployment actually rolled out
- `ServerSideApply.Enabled` = `True`
- `ServerSideApply.ForceConflicts` = `True`
- `DeploymentTimeout` = `180`

And the top-level action fields:

- `Packages[]` — for each image, `{Name, PackageId, FeedId, AcquisitionLocation: "NotAcquired"}`. `NotAcquired` tells Octopus not to pre-download the image to the worker — the cluster pulls it directly.

This is why manifest-based steps are preferable: the user's YAML or chart already encodes all of this.

## Validate before saving

Before `PUT /api/{sp}/projects/{id}/deploymentprocesses`, dry-run the proposed shape:

```
POST /api/{sp}/projects/{id}/deploymentprocesses/validate
```

Returns specific warnings about missing variables, bad tag references, or missing packages. Fix them before saving — the user sees a much cleaner first-deploy experience when validation runs ahead of the write.

---

## Worked example: "deploy my project in the current repo to two tenants via Kubernetes"

Full end-to-end call sequence. Italic lines are agent-to-user text. Both transports work — MCP-tool names are noted alongside the REST endpoints.

### Phase 1: repo inspection (silent)

- Read `package.json` / `Chart.yaml` / `Dockerfile` — detect package shape.
- Scan `.github/workflows/*.yml` (or GitLab / Azure / Jenkins equivalent) — extract the `docker/build-push-action` or `docker push` step, and any `helm push` / `helm package` step.
- From those steps: image = `ghcr.io/acme/payments`, tag = `${{ github.sha }}`; chart pushed as `oci://ghcr.io/acme/charts/payments` if present.
- Conclusion: Kubernetes target, Helm chart (or raw container — take whichever is in the repo), GHCR feed. Enough to plan.

### Phase 2: confirmations (~3 lines total)

- *"I see `Chart.yaml` and a GitHub Actions workflow pushing `ghcr.io/acme/payments:<git-sha>`. I'll deploy via Helm, pulled from GHCR. Sound right?"*
- *"What are the two customers / deployment slices called?"* (e.g., `tenant-a`, `tenant-b`)
- *"I'll use the Kubernetes agent — it installs via Helm, polls outbound, and doesn't need a cluster credential stored in Octopus. OK?"*

### Phase 3: call sequence

```
1.  Establish serverUrl and agentReachableUrl (SKILL.md §0.1).
    Validate cluster → agentReachableUrl reachability.

2.  list_spaces            (REST: GET /api/spaces/all)
    → pick Spaces-1

3.  list_environments      (REST: GET /api/Spaces-1/environments/all)
    → if empty, execute POST /api/Spaces-1/environments { Name: "Production" }

4.  execute POST /api/Spaces-1/machines   (pre-create the agent target; Unknown state)
    {
      Name: "prod-cluster",
      Slug: "prod-cluster",
      Roles: [],
      EnvironmentIds: ["Environments-1"],
      TenantedDeploymentParticipation: "TenantedOrUntenanted",
      Endpoint: {
        CommunicationStyle: "KubernetesTentacle",
        TentacleEndpointConfiguration: { CommunicationMode: "Polling" },
        KubernetesAgentDetails: {
          HelmReleaseName: "octopus-agent-prod-cluster",
          KubernetesNamespace: "octopus-agent-prod-cluster"
        },
        UpgradeLocked: false
      }
    }
    → Machines-1

5.  Retrieve the agent's Helm command + registration token.
    Show the user the helm upgrade --install command (see references/kubernetes-agent-install.md).
    Wait: poll find_deployment_targets (or GET /api/Spaces-1/machines/Machines-1)
          until HealthStatus == "Healthy".

6.  execute POST /api/Spaces-1/feeds   (Model A — pull from GHCR)
    {
      Name: "GHCR - acme",
      Slug: "ghcr-acme",
      FeedType: "GithubContainer",
      FeedUri: "https://ghcr.io",
      Username: "<gh-user>",                       // mandatory, even though
      Password: { NewValue: "<PAT read:packages>", //   the package is public
                  HasValue: true }                 //   (build-output-and-feeds.md)
    }
    → Feeds-1
    Smoke-test: GET /api/Spaces-1/feeds/Feeds-1/packages/search?term=payments

7.  GET /api/Spaces-1/tenants/status
    → if multi-tenancy disabled, enable it:
      GET  /api/Spaces-1/featuresconfiguration            // round-trip the body
      execute PUT /api/Spaces-1/featuresconfiguration { ...prev, IsMultiTenancyEnabled: true }

8.  execute POST /api/Spaces-1/tagsets
    { Name: "Customer", Tags: [{Name: "tenant-a"}, {Name: "tenant-b"}] }

9.  execute POST /api/Spaces-1/tenants × 2
    { Name: "Tenant A", TenantTags: ["Customer/tenant-a"], ProjectEnvironments: {} }
    { Name: "Tenant B", TenantTags: ["Customer/tenant-b"], ProjectEnvironments: {} }

10. execute POST /api/Spaces-1/projects
    {
      Name: "payments",
      ProjectGroupId: "ProjectGroups-1",
      LifecycleId: "Lifecycles-1",
      TenantedDeploymentMode: "Tenanted"
    }
    → Projects-1

11. execute PUT /api/Spaces-1/tenants/Tenants-{1,2}
    ProjectEnvironments: { "Projects-1": ["Environments-1"] }
    (round-trip the tenant body — PUT is replace, not patch; see error-recovery.md)

12. execute PUT /api/Spaces-1/projects/Projects-1/deploymentprocesses
    Prefer Helm because Chart.yaml is in the repo:
    {
      Steps: [{
        Name: "Deploy payments chart",
        Actions: [{
          Name: "Helm upgrade",
          ActionType: "Octopus.HelmChartUpgrade",
          Properties: {
            "Octopus.Action.Script.ScriptSource": "Package",
            "Octopus.Action.Helm.ReleaseName":
              "#{Octopus.Project.Name | Slugify}-#{Octopus.Deployment.Tenant.Name | Slugify}",
            "Octopus.Action.Helm.Namespace":
              "#{Octopus.Deployment.Tenant.Name | Slugify}",
            "Octopus.Action.Helm.ResetValues": "True",
            "Octopus.Action.Kubernetes.ResourceStatusCheck": "True"
          },
          Packages: [{
            Name: "",                           // primary package
            PackageId: "acme/charts/payments",
            FeedId: "Feeds-1",
            AcquisitionLocation: "NotAcquired"
          }]
        }]
      }]
    }

    (If the repo has raw manifests instead, use Octopus.KubernetesDeployRawYaml with
     Octopus.Action.KubernetesContainers.CustomResourceYamlFileName = "manifests/**/*.yaml"
     and Octopus.Action.Script.ScriptSource = "GitRepository".)

13. execute POST /api/Spaces-1/tenants/Tenants-{1,2}/projectvariables
    per-tenant overrides (namespace, replica count, secrets)

14. (optional) execute POST /api/Spaces-1/projects/Projects-1/triggers
    FeedTrigger that auto-creates a release when a new GHCR tag appears.
    Explain before enabling.

15. create_release   (REST: POST /api/Spaces-1/releases)
    { ProjectId: "Projects-1",
      Version: "0.1.0",
      SelectedPackages: [{ ActionName: "Helm upgrade",
                           PackageReferenceName: "",
                           Version: "<chart version>" }] }
    → Releases-1
    (Gates via elicitation if MCP. User confirms.)

16. (optional) execute POST /api/Spaces-1/build-information
    attach CI run URL, commit list, work items

17. Show the user (do NOT auto-run):
    - release URL
    - two deploy commands, one per tenant:
      deploy_release   (REST: POST /api/Spaces-1/deployments)
      { ReleaseId: "Releases-1", EnvironmentId: "Environments-1", TenantId: "Tenants-1" }
      deploy_release
      { ReleaseId: "Releases-1", EnvironmentId: "Environments-1", TenantId: "Tenants-2" }
    - optionally, a paste-ready CI snippet for POST /api/.../releases/create/v1
      if the user wants CI to create releases going forward (Model C).
```

### Phase 4: what the agent must not do

- Generate or modify `.github/workflows/*.yml`. If the user wants CI-triggered releases, output a snippet — don't commit it.
- Auto-run steps 15–17. Show them.
- Create extra environments, channels, or runbooks "just in case."
- Use *tenant* / *space* / *channel* / *runbook* / *lifecycle* without grounding them first (see `references/vocabulary-and-deferrals.md`).
