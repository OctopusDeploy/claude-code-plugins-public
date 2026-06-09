# Kubernetes agent install

Companion reference for `SKILL.md` (`octopus-onboarding`), Step C and Step D. Use this when the user is registering a Kubernetes target.

The recommendation in `SKILL.md` is to default to the **Kubernetes agent** (`CommunicationStyle: "KubernetesTentacle"`) — a Helm-installed polling agent. This file walks through the install flow end-to-end. The legacy direct-API path (`CommunicationStyle: "Kubernetes"`) is at the bottom for completeness; it's only the right choice when the user has an explicit reason (existing audited service account, regulatory constraint requiring Octopus-managed credentials).

## Why the agent wins

- No long-lived service-account token for Octopus to mint, store, or rotate.
- No `SkipTlsVerification` workaround against self-signed cluster certs.
- Polling outbound only — Octopus never needs inbound reach to the cluster API. Works for Docker Desktop, NAT'd clusters, VPN'd clusters, air-gapped-with-tunnel — same shape.
- Identity scoped to the install namespace, not cluster-admin.
- Same pattern scales unchanged from `docker-desktop` to production EKS.

## Prerequisite: server reachability from the cluster

The agent pod needs to dial `agentReachableUrl` (from `SKILL.md` §0.1). Validate this *before* running anything.

| Cluster shape | What `agentReachableUrl` should be |
|---|---|
| Docker Desktop / Rancher Desktop / Minikube on the same laptop as Octopus | `http://host.docker.internal:<port>` (confirm Docker Desktop's "Kubernetes uses the same host internal DNS" — on by default) |
| Remote cluster + Octopus on a LAN box | LAN hostname or IP — `localhost` will not resolve in the cluster |
| Self-hosted Octopus behind a firewall + cloud cluster | **Stop and ask.** The user needs a tunnel (Cloudflare Tunnel, Tailscale, ngrok) or a public ingress first. Don't start the Helm install on an unreachable server. |
| Octopus Cloud | Same URL for everything |

Ask the user:

> *"Can the cluster reach `{agentReachableUrl}`? If you're not sure, run this from a shell with `kubectl` configured for the target cluster — a 200 or 401 means we're good; a timeout means we have a network problem to fix first."*

```
kubectl run curl --rm -it --image=curlimages/curl -- -sv {agentReachableUrl}/api
```

## Pre-create the target in `Unknown` state

The server provisions a registration token when the target is created and Helm uses it to register the agent. So: create the target *first*, then run Helm.

```
POST /api/{sp}/machines       (or MCP `execute` with the same body)
{
  "Name": "docker-desktop",
  "Slug": "docker-desktop",
  "Roles": [],                          // roles are tags the user picks; empty is fine
  "EnvironmentIds": ["Environments-1"],
  "TenantedDeploymentParticipation": "TenantedOrUntenanted",
  "Endpoint": {
    "CommunicationStyle": "KubernetesTentacle",
    "TentacleEndpointConfiguration": { "CommunicationMode": "Polling" },
    "KubernetesAgentDetails": {
      "HelmReleaseName": "octopus-agent-docker-desktop",
      "KubernetesNamespace": "octopus-agent-docker-desktop"
    },
    "DefaultNamespace": "default",
    "UpgradeLocked": false
  }
}
→ MachineResource; HealthStatus: "Unknown"
```

The response carries the registration metadata. Retrieve the install command from the dedicated generator endpoint:

```
GET (or POST per spec) /api/{sp}/machines/helm-upgrade-command   ← check the spec for current shape
→ { RegistrationKey, ServerCommsAddress, Thumbprint, ... }
```

Always cross-check the field names against `octopus://api/llms.txt` (or `GET /api/experimental/llms.txt`) before constructing the Helm command — endpoint shape evolves between versions.

## Show the user the Helm command

```
helm upgrade --install --atomic \
  --create-namespace --namespace octopus-agent-docker-desktop \
  --set agent.acceptEula=Y \
  --set agent.name=docker-desktop \
  --set agent.serverUrl={agentReachableUrl} \
  --set agent.serverCommsAddress={serverCommsAddress} \
  --set agent.spaceName="{spaceName}" \
  --set agent.targetEnvironments="{Production}" \
  --set agent.targetRoles="" \
  --set agent.bearerToken={registrationToken} \
  octopus-agent-docker-desktop oci://registry-1.docker.io/octopusdeploy/kubernetes-agent
```

**Do not run `helm` from your own shell.** The user runs it with `kubectl` context pointed at their cluster. Show the command, briefly explain the flags, stop. Common questions you'll be asked:

- *"What does `--atomic` do?"* — rolls back on failure so a half-installed release doesn't linger.
- *"Why a separate namespace?"* — keeps the agent's RBAC scoped; nothing in the cluster outside that namespace gets implicit Octopus access.
- *"Can I edit the values file instead?"* — yes; the `--set` flags map 1:1 to `agent.*` keys in `values.yaml`.

## Poll for `Healthy`

```
loop until HealthStatus == "Healthy" (or 5 min timeout):
  GET /api/{sp}/machines/{id}        (MCP: find_deployment_targets with the ID)
  → "Agent status: {HealthStatus} — {StatusSummary}"
```

When it flips to `Healthy`, the agent has registered itself, picked up a Tentacle thumbprint, and is ready. If it stays `Unknown` past ~2 minutes, debug:

- Helm install actually succeeded? → `kubectl get pods -n {namespace}` — look for `octopus-agent-tentacle-*` Running.
- Pod crashing or backing off? → `kubectl logs -n {namespace} deploy/octopus-agent-tentacle`. Typical failures:
  - Wrong `serverUrl` → DNS / not reachable from inside the pod.
  - Wrong `bearerToken` → expired or copy-paste error.
  - Firewall blocking outbound to Octopus.
- `{agentReachableUrl}` actually reachable from the pod? Re-run the curl-in-a-pod prerequisite check.

Only once the agent is `Healthy` do you proceed to feeds (`SKILL.md` §2) and project / process (`SKILL.md` §3).

---

## Direct-API Kubernetes (legacy)

Use `CommunicationStyle: "Kubernetes"` only when the user specifically wants Octopus to hold cluster credentials. Tell them this path exists but the agent is preferred. If they still want it, here's the auth matrix.

You'll need:

- A worker pool (`DefaultWorkerPoolId` on the endpoint) — workers run `kubectl` against the cluster.
- A cluster URL (`ClusterUrl`).
- An `Authentication` sub-object — pick one row from below.
- Cluster-side RBAC: a `ServiceAccount`, `ClusterRole` (or namespaced `Role`), and `ClusterRoleBinding`. Generate this and hand it to the user; don't `kubectl apply` for them.

| `Authentication.AuthenticationType` | Required fields |
|---|---|
| `KubernetesStandard` | `AccountId` (token or user/password account) |
| `KubernetesCertificate` | `ClientCertificate` (certificate resource ID) |
| `KubernetesAzure` | `AccountId` (Azure SP or OIDC), `ClusterName`, `ClusterResourceGroup`, `AdminLogin`? |
| `KubernetesAws` | `AccountId` (AWS), `ClusterName`, `Region`, `AssumeRole`?, `AssumedRoleArn`?, `UseInstanceRole`? |
| `KubernetesGoogle` | `AccountId` (GCP), `Project`, `Zone`/`Region`, `ClusterName` |
| `KubernetesPodService` | `TokenPath` — for in-cluster auth when Octopus itself runs inside the cluster |

Example body shape:

```json
{
  "Name": "prod-cluster",
  "EnvironmentIds": ["Environments-1"],
  "Roles": ["k8s"],
  "Endpoint": {
    "CommunicationStyle": "Kubernetes",
    "ClusterUrl": "https://kubernetes.example.com:6443",
    "ClusterCertificate": "Certificates-1",
    "SkipTlsVerification": false,
    "DefaultWorkerPoolId": "WorkerPools-1",
    "Authentication": {
      "AuthenticationType": "KubernetesAws",
      "AccountId": "Accounts-1",
      "ClusterName": "prod",
      "Region": "us-east-1",
      "UseInstanceRole": false
    }
  }
}
```

The legacy path also needs a Health task (same `POST /api/tasks` flow as in `SKILL.md` §1 Step C) — the `Unknown → Healthy` transition is the same, but failure modes are different (TLS, RBAC, cluster auth) and the diagnostics are pure `kubectl` from a worker rather than agent pod logs.
