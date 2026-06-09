# Build output and feeds

Companion reference for `SKILL.md` (`octopus-onboarding`) §2. Use this when you're working out where the user's build artifacts come from and how Octopus will pull them in.

The cardinal rule from `SKILL.md` repeated up front: **don't modify the user's CI config and don't commit to their repo.** If a CI snippet is needed, output it; let the user paste.

## §2.1 Detect the CI system from the repo

Check for these files. Multiple may coexist — the active one is usually the most recently committed or the one referenced in README. Ask if ambiguous.

| File / dir | System | Where the push step lives |
|---|---|---|
| `.github/workflows/*.yml` | GitHub Actions | `uses:` actions; `run:` steps; `env:` for registry URLs |
| `.gitlab-ci.yml` (+ `.gitlab/ci/*.yml`) | GitLab CI | `script:` blocks; `$CI_REGISTRY`, `$CI_REGISTRY_IMAGE` vars |
| `Jenkinsfile` (+ `Jenkinsfile.*`) | Jenkins | `docker.withRegistry`, `sh` blocks, shared libs |
| `azure-pipelines.yml` (+ `.azure-pipelines/*.yml`) | Azure Pipelines | `Docker@2`, `NuGetCommand`, `MavenAuthenticate` tasks |
| `.circleci/config.yml` | CircleCI | `orbs:` — especially `aws-ecr`, `gcp-gcr`, `docker` |
| `bitbucket-pipelines.yml` | Bitbucket | `script:` blocks; pipe references |
| `.buildkite/*.yml` / `buildkite.yml` | Buildkite | `steps:`, plugin refs |
| `cloudbuild.yaml` | Google Cloud Build | `steps:` with `gcr.io/cloud-builders/*` |
| `.teamcity/` or `settings.kts` | TeamCity | Kotlin DSL — hard to parse; prefer asking |
| `Makefile` + no CI config | Bespoke / local-only | Ask the user what runs it |
| None of the above | No CI yet | See §2.5 below |

Once the CI system is identified, emit it as part of the **Stack detected** one-liner from `SKILL.md` §0.6 SCAN:

```
Stack detected: <language(s)> · <deployment artifact> · CI: <system>
```

Examples (CI half of the line):
- `… · CI: GitHub Actions` — `.github/workflows/*` present.
- `… · CI: Azure Pipelines` — `azure-pipelines.yml` present.
- `… · CI: Jenkins (Jenkinsfile)` — note the file kind in parens when the system has several shapes.
- `… · CI: none detected` — none of the files above are present (jump to §2.5).

If multiple CI systems are present (e.g., `.github/workflows/` *and* `azure-pipelines.yml`), join them with a slash: `… · CI: GitHub Actions / Azure Pipelines` and ask in §0.7 CLARIFY which one is the active path.

## §2.2 Infer what's built and where it's pushed

Parse the CI config to extract three things: **artifact type**, **destination registry**, **version scheme**.

### Docker images

Match any of:
- `docker/build-push-action` (GitHub Actions) — read `tags:` and `push: true`.
- `docker build` + `docker push` in a `run:` / `script:` block.
- `aws-actions/amazon-ecr-login` → ECR; `azure/docker-login` → ACR; `google-github-actions/auth` + `gcr.io` / `pkg.dev` → GCR / Artifact Registry.
- `docker/login-action` with `registry: ghcr.io` → GHCR; no registry → Docker Hub.

Extract image coordinates: `<registry>/<namespace>/<image>:<tag>`. Record the tag pattern — it becomes the Octopus package version (§2.7).

### NuGet packages

`dotnet nuget push *.nupkg --source <url>` or `nuget push`. The `--source` URL tells you the feed: `api.nuget.org` (public), `nuget.pkg.github.com` (GitHub Packages), internal host (Artifactory / Nexus / ProGet).

### JAR / Maven artifacts

`mvn deploy`, `gradle publish`. Parse `pom.xml` for `<distributionManagement>/<repository>/<url>` or `build.gradle` for `publishing { repositories { maven { url ... } } }`.

### npm packages

`npm publish` + `package.json`'s `publishConfig.registry` or `.npmrc`.

### Python wheels

`twine upload`, `poetry publish`. Target URL from CLI args or `~/.pypirc`.

### Helm charts

`helm push` (OCI) or `helm cm-push` (ChartMuseum) or `chart-releaser` (GitHub Pages as a Helm repo).

### Generic zip / tar → bucket

`aws s3 cp`, `gsutil cp`, `az storage blob upload`. The Octopus feed is the bucket itself (S3 feed) or a Generic / Artifactory feed.

### Things that are not feeds

CI-internal artifact stashes — GitHub Actions' `actions/upload-artifact`, GitLab's `artifacts:` block. These pass files between jobs *in the same pipeline only*. Ignore them unless paired with a later push step that escapes the pipeline.

## §2.3 Map to an Octopus feed type

The `FeedType` values from `CreateFeedCommand`:

| Build pushes to | Octopus `FeedType` | Notes |
|---|---|---|
| Docker Hub or generic v2 Docker registry | `Docker` | Works for any Docker registry with creds. |
| ghcr.io | `GithubContainer` or `Docker` | Either works; `GithubContainer` has cleaner PAT auth. |
| Amazon ECR | `AwsElasticContainerRegistry` | Requires an AWS account resource. |
| Azure Container Registry | `Docker` | Uses admin creds or a service principal. |
| Google Artifact Registry / GCR | `Docker` | Uses a GCP service account key. |
| OCI Helm registry | `OciRegistry` or `Helm` | Check the enum for the installed Octopus version. |
| Helm ChartMuseum / classic Helm repo | `Helm` | |
| NuGet (nuget.org, GitHub Packages, Artifactory) | `Nuget` | |
| Maven repo | `Maven` | |
| S3 bucket (generic zips) | `S3` | Treats each object key as a package version. |
| Artifactory generic | `ArtifactoryGeneric` | Zip / tar not covered elsewhere. |
| GitHub Releases | `GitHub` | Pulls release assets as packages. |
| Octopus built-in (push model) | — no feed to create — | Packages appear under the built-in feed automatically. |

Register the feed:

- **MCP**: `execute` with `POST /api/{sp}/feeds` (write-gated through elicitation).
- **REST**: `POST /api/{sp}/feeds` directly.

Smoke-test with `GET /api/{sp}/feeds/{id}/packages/search?term=<image-or-package-name>`. Results = feed is live; `Unable to retrieve authentication token` = creds are wrong or missing (see "registry creds" below).

## §2.4 Pick a handoff model

Three patterns. Default to the one that needs the fewest CI edits — most users will not accept an agent modifying or generating their CI config.

### Model A — External feed (pull). Preferred.

Octopus points at the registry the CI already pushes to. Zero CI changes.

- Use when CI publishes to a registry the agent can give Octopus credentials for.
- Setup: create feed (§2.3) → reference `Octopus.Action.Package.FeedId` in the deployment process → pick package version at release creation.
- Optional: a project trigger that auto-creates a release when a new package version appears in the feed. `POST /api/{sp}/projects/{id}/triggers` with a `FeedTrigger` filter action. Explain it before enabling — some users want explicit release creation.

### Model B — Push to Octopus's built-in feed.

CI uploads packages directly to Octopus. One CI change (an upload step).

- Use when CI produces a package but doesn't push it durably (e.g., only `actions/upload-artifact`), or the user wants a single source of truth inside Octopus.
- Endpoints:
  - NuGet-shaped: `PUT /api/nuget/packages` (multipart, with `X-Octopus-ApiKey`). Compatible with `nuget push` and `dotnet nuget push`.
  - Anything else: `POST /api/{sp}/packages/raw` (multipart form field `raw: <file>`).
  - Large files: delta-upload endpoints (`POST /api/{sp}/packages/{packageId}/{baseVersion}/delta` + `GET .../delta-signature`).
- Output a CI snippet for the user to paste. **Do not modify their CI file.**

### Model C — CI-triggered release.

CI builds, pushes, then calls Octopus to create a release (and optionally deploy).

- Use when the user has a mature CI and wants Octopus to be deploy-only.
- Endpoint: `POST /api/{sp}/releases/create/v1` (`CreateReleaseCommandV1`) — accepts package versions by name, so CI can pass what it just built. Pairs naturally with `POST /api/{sp}/deployments` for the deploy step (gated by lifecycle for the multi-environment branch; manual for the fast-deploy branch).

### Decision

- Already pushing to a shareable registry? → **Model A.**
- Registry is unreachable from Octopus (air-gapped, strict private Artifactory)? → **Model B.**
- User wants CI to drive the whole release cadence? → **Model C.**
- Unsure? → **Model A** + optional trigger. Smallest CI footprint.

## §2.5 When the repo has no CI yet

Don't set one up. Say:

> *"There's no CI config in this repo yet. Octopus deploys built packages — something needs to build and publish them first. Two options: (1) build locally and push to Octopus's built-in feed with a one-line command — fine for prototyping; (2) add a CI that publishes to a registry, then come back and I'll wire Octopus to it. Which?"*

For (1): output the curl / `octo package upload` / `dotnet nuget push` command for `POST /api/{sp}/packages/raw` and stop. Don't offer to write a `.github/workflows/build.yml`.

## §2.6 Attach build information (traceability)

After a package lands in the feed (any model), optionally attach build info. Octopus uses it to link commits, PRs, and work items to each deploy — essential for teams in regulated industries that need an audit trail from code to production.

`POST /api/{sp}/build-information` with `CreateBuildInformationCommand`:

- `PackageId`, `Version` — the package coordinates.
- `OctopusBuildInformation`:
  - `BuildEnvironment` — "GitHub Actions", "GitLab CI", etc.
  - `VcsRoot`, `VcsCommitNumber`, `Branch`, `BuildUrl`.
  - `Commits[]` — hashes + messages between this build and the last.
  - `WorkItems[]` — Jira IDs, GitHub issue numbers, Linear refs (parsed from commit messages).

The agent can collect most of this from inside a CI run. For external feeds, output a paste-ready curl snippet — same rule as Model B: don't edit the user's config.

## §2.7 Version detection — critical detail

The *version* is what Octopus uses to create releases. Common CI patterns:

| CI pattern | Version Octopus sees |
|---|---|
| `docker build -t app:${{ github.sha }}` | git SHA (7 or 40 chars) — opaque string, sorts lexicographically |
| `docker build -t app:${{ github.ref_name }}` on tag push | semver from git tag (`v1.2.3` → `1.2.3`) |
| `mvn deploy` with `-SNAPSHOT` version | Maven snapshot — feed must allow prereleases |
| `dotnet pack -p:Version=$GITVERSION_FULLSEMVER` | semver + build metadata from GitVersion |
| `helm package --version $VERSION` | explicit arg |
| `npm version patch && npm publish` | bumped semver |

If the CI uses *only* `github.sha`, tell the user:

> *"Octopus can sort semver but not SHAs — you'll lose version ordering, channel rules, and 'deploy latest' semantics. Consider adding a semver tag on release commits. I won't change your workflow; this is just a flag."*

Then proceed with the SHA version.

## §2.8 What the agent does vs. asks

**Silent:**
- Read CI config files in the repo.
- Parse push destination, image name, version scheme.
- Infer the feed type.

**Confirm:**
- *"Your GitHub Actions workflow pushes `ghcr.io/acme/payments` tagged with the git SHA — I'll register that as a GitHub Container feed. Right?"*
- *"I see two publish destinations (Docker Hub and ECR). Which one should Octopus pull from?"*

**Ask explicitly:**
- Registry credentials the repo can't reveal (e.g., `${{ secrets.REGISTRY_TOKEN }}`).
- Which handoff model (A / B / C) when the repo is ambiguous.

**Never:**
- Modify a CI config file.
- Commit to the repo.
- Write a new CI config unless the user explicitly asked for one.

## Registry creds — the always-needed-even-for-public detail

Octopus's Docker-feed handshake **does not support anonymous pull**. Empty `Username` / `Password` fails registration with `Unable to retrieve authentication token`. You must supply *some* credentials even for:

- A public GHCR repo
- A public Docker Hub image
- A public ECR Public registry

A minimal read-only PAT with the user's own GitHub username is fine — the registry stays public; the creds just authenticate the handshake.

Per-registry minimum-scope tokens:

| Registry | Minimum scope |
|---|---|
| **GHCR** | PAT with `read:packages` scope. Username = GitHub username (classic PAT) or literal `USERNAME` (fine-grained tokens pointing at the owner's packages). Required **even for public images**. |
| **ECR** | `ecr:GetAuthorizationToken` + `ecr:BatchGetImage` + `ecr:GetDownloadUrlForLayer`. Uses the AWS account resource — no separate username/password. |
| **ACR** | `AcrPull` role assignment on a service principal, or ACR admin creds. |
| **GAR / GCR** | `roles/artifactregistry.reader`. Username = `_json_key` + password = service-account JSON is the conventional pattern; prefer Workload Identity Federation where available. |
| **Docker Hub** | Read-only access token for an account that can see the repo (even for public images, a free-tier token avoids anonymous rate limits and satisfies the handshake). |

After `POST /api/{sp}/feeds`, prove the creds work:

```
GET /api/{sp}/feeds/{id}/packages/search?term=<package-name>
```

If it fails with `Unable to retrieve authentication token`, the creds are wrong or missing — show the exact permission gap.
