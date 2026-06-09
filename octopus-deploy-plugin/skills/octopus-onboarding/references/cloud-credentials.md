# Cloud credentials

Companion reference for `SKILL.md` (`octopus-onboarding`). Read this when the user is connecting Octopus to AWS, Azure, GCP, or any other cloud account. The rule from `SKILL.md` §1 Step B: **show the permission surface before asking for credentials.** Over-scoped credentials are the #1 early-stage blocker.

## The general pattern

For any cloud, the order is:

1. Determine the minimum-scope policy / role for the specific Octopus step types the user is about to run.
2. Show the policy as a paste-ready block. Don't ask for keys yet.
3. Offer OIDC by default where the cloud and Octopus version support it. Long-lived keys are the fallback, not the default.
4. After `POST /api/{sp}/accounts` (MCP: `execute` with that path), call `GET /api/{sp}/accounts/{id}/usages` to demonstrate the scope is empty — nothing yet trusts this account. Reassures the user that even mistyped creds aren't dangerous yet.

## Account types and `CommunicationStyle` mapping

| Detected target | Account type (`CreateAccountCommand.AccountType`) |
|---|---|
| AWS ECS / EKS / Lambda / S3 | `AmazonWebServicesAccount` (IAM user keys) **or** OIDC role (`AwsOidcAccount`) |
| Azure App Service / AKS / Functions | `AzureServicePrincipal` **or** `AzureOidcAccount` |
| GCP / GKE / Artifact Registry | `GoogleCloudAccount` |
| Generic OIDC provider | `GenericOidcAccount` |
| SSH-managed Linux host | `SshKeyPair` or `UsernamePassword` |
| Username/password (SQL, generic) | `UsernamePassword` |
| Bearer token (generic) | `Token` |
| On-prem Windows / Linux Tentacle | **No account.** Register the target directly. |

## AWS

### Step-type-driven minimum scope

Generate a scoped IAM policy based on the steps the user is about to run. Don't dump the same admin-y blob every time.

| Step | Required actions |
|---|---|
| `Octopus.AwsRunCloudFormation` | `cloudformation:*` on stack ARN; passthrough actions for resources the template provisions |
| `Octopus.AwsRunScript` | Whatever the script needs — surface that explicitly |
| `Octopus.AwsApplyCloudFormationChangeSet` | `cloudformation:CreateChangeSet`, `ExecuteChangeSet`, `DescribeChangeSet`, `Describe*`, `GetTemplate` |
| `Octopus.AwsUploadS3` | `s3:PutObject`, `s3:PutObjectAcl`, `s3:ListBucket` on the target bucket |
| `Octopus.AwsRunECSTask` (or ECS deploys) | `ecs:RegisterTaskDefinition`, `ecs:UpdateService`, `ecs:DescribeServices`, `ecs:DescribeTasks`, `iam:PassRole` for the task execution role |
| `Octopus.AwsRunLambda` (Lambda deploys) | `lambda:UpdateFunctionCode`, `lambda:UpdateFunctionConfiguration`, `lambda:GetFunction`, `lambda:PublishVersion`, `lambda:UpdateAlias` |
| Feed type `AwsElasticContainerRegistry` | `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer` |

Worked example for ECS deployments to a single cluster (the most common AWS shape):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RegisterAndUpdate",
      "Effect": "Allow",
      "Action": [
        "ecs:RegisterTaskDefinition",
        "ecs:UpdateService",
        "ecs:DescribeServices",
        "ecs:DescribeTasks",
        "ecs:DescribeTaskDefinition"
      ],
      "Resource": "*"
    },
    {
      "Sid": "PassExecutionRole",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::<account>:role/<task-execution-role>"
    },
    {
      "Sid": "PullImagesForECS",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer"
      ],
      "Resource": "*"
    }
  ]
}
```

### OIDC (preferred)

If the Octopus instance supports OIDC accounts (most current versions do), offer `AwsOidcAccount` instead of IAM user keys. The user creates an IAM role in AWS with a trust policy pointing at Octopus's OIDC issuer, then points Octopus at the role ARN. No long-lived keys, automatic rotation by IAM.

Trust policy snippet (paste-ready):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Federated": "arn:aws:iam::<account>:oidc-provider/<octopus-issuer>" },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": { "<octopus-issuer>:aud": "<configured-audience>" }
      }
    }
  ]
}
```

Pull the exact issuer URL and audience from the Octopus instance's OIDC configuration before showing the snippet — they're per-instance.

## Azure

### Service Principal (the long-lived path)

Required permissions vary by step type. Common ones:

| Deploy target | Role assignment |
|---|---|
| App Service | `Website Contributor` on the resource group |
| Functions | `Website Contributor` on the resource group |
| AKS | `Azure Kubernetes Service Cluster User` (for kubeconfig) + `Azure Kubernetes Service RBAC Cluster Admin` (for in-cluster RBAC) |
| Container Registry | `AcrPull` on the registry |
| Service Bus / Storage / SQL | scoped roles per service |

Avoid suggesting `Contributor` at subscription level. If the user already has it, document it but flag that it's wider than necessary.

### Azure OIDC (preferred)

`AzureOidcAccount`. Configure a federated identity credential on the App Registration that trusts the Octopus issuer. Same shape as AWS OIDC but Azure-specific.

## GCP

| Deploy target | IAM role |
|---|---|
| GKE | `roles/container.developer` (or `container.admin` if the deployments need to manage the cluster, not just workloads) |
| Cloud Run | `roles/run.developer` + `roles/iam.serviceAccountUser` on the runtime SA |
| Cloud Functions | `roles/cloudfunctions.developer` + `roles/iam.serviceAccountUser` |
| Artifact Registry / GCR | `roles/artifactregistry.reader` (or `.writer` if Octopus pushes) |
| GCS | `roles/storage.objectAdmin` on the bucket |

For service account JSON keys: scope them to a single service account with only the roles above. Encourage the user to switch to Workload Identity Federation (GCP's OIDC equivalent) where possible.

## Generic OIDC and tokens

`GenericOidcAccount` — for any OIDC provider Octopus federates with that isn't AWS / Azure / GCP. Issuer URL + audience.

`Token` — bearer token for arbitrary HTTP-style services. Treat as a long-lived secret; recommend rotation reminders.

## SSH

`SshKeyPair` (preferred) or `UsernamePassword`. The host's fingerprint must be confirmed at registration time — surface the fingerprint to the user before saving, don't accept the first one offered without prompting.

## After creating the account

```
GET /api/{sp}/accounts/{accountId}/usages
→ shows what currently references this account (deployment processes, runbooks, machine endpoints)
```

For a freshly created account, this list should be empty. Show the user. It's the cheapest demonstration that nothing has yet trusted this credential — useful confidence-builder when the user is about to paste a key for the first time.

## Cross-cutting hygiene

- **Never paste a user's actual key, ARN, secret, or service account JSON into chat.** Keep them in the `POST /accounts` body and in the user's secret store.
- **Don't reuse one account across "many things."** Octopus's account model is cheap — one account per cloud × per environment scope is fine and audits better.
- **Surface rotation timeline** — IAM keys, SP secrets, GCP keys all have rotation expectations. Mention this when saving long-lived creds; it's a routine operational concern users expect to be reminded of.
