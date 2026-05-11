# Action Types Reference

This file is the authoritative catalog of every `action_type` supported by Octopus Deploy and the property keys each one accepts. Use it as a lookup table when writing the `properties = { ... }` dictionary of an `action` block.

## How to use this file

1. Identify the `action_type` value you need. Common ones:
   - `Octopus.Script` — run an inline/package/git script (PowerShell, Bash, Python, etc.)
   - `Octopus.AzurePowerShell` — Azure-authenticated script
   - `Octopus.AwsRunScript` — AWS-authenticated script
   - `Octopus.Manual` — manual intervention (approval gate)
   - `Octopus.Package` — deploy a package
   - `Octopus.DeployRelease` — deploy a release of another project
   - `Octopus.Email` — send an email
   - `Octopus.Kubernetes.Kustomize` — apply Kustomize manifests
   - `Octopus.KubernetesRunScript` — kubectl-authenticated script
   - `Octopus.HelmChartUpgrade` — upgrade Helm release
   - `Octopus.TerraformApply` / `Octopus.TerraformPlan` / `Octopus.TerraformDestroy`
   - `Octopus.AzureAppService` / `Octopus.AzureWebApp` / `Octopus.AzureResourceGroup`
   See the catalog below for the complete list.

2. Find that action_type's section using the `(`Octopus.X`)` heading anchor. Each section lists every supported property key with: `Required` (yes/no), `Default` (if any), `Values` (allowed enum values, if constrained), and a `Description`.

3. Translate each property into an entry of the OCL `properties = { ... }` dictionary. All values are stringified — booleans appear as `"True"` / `"False"`, numbers as quoted strings (`"30"`), enums as their literal token (`"Inline"`, `"GitRepository"`, etc.).

## Rules

- **Never invent property keys.** If a key is not listed for an action_type, it is not supported and will be rejected (or silently ignored) by Octopus.
- **Required properties must be present.** Some properties are conditionally required (e.g., `Octopus.Action.Script.ScriptBody` is required when `Octopus.Action.Script.ScriptSource = "Inline"`). The Description column flags these.
- **Sensitive values** marked `(sensitive)` should not be hard-coded in OCL. Reference an Octopus variable (e.g., `#{MyApiKey}`) or a sensitive variable in `variables.ocl` (which itself stores no value — the value is set via UI).
- **Property values are always strings in OCL.** A boolean `true` from the table appears as `Octopus.Action.X.Y = "True"` in OCL.
- **Worker pool, container, packages, git_dependencies** are NOT properties — they are sibling blocks of `action`. Do not put `worker_pool` inside `properties = { }`.
- **`channels`, `environments`, `excluded_environments`, `tenant_tags`** are also action-level attributes (lists), not properties.

## Catalog

The catalog below is grouped by category (ArgoCD, AWS, Azure, Certificates, Discovery, Docker, Email, GitHub, Google Cloud, Helm, Kubernetes, Manual, Octopus, Package, PowerShell/Script, ServiceNow, Slack, SSH, Step Templates, Windows Service, etc.).

---

## Steps

Deployment process steps are identified by `ActionType`. When authoring a `DeploymentProcessResource.Steps[].Actions[]` entry, set `ActionType` to one of the values below and populate the action's `Properties` dictionary using the keys listed for that step.

### ArgoCD

#### Trigger ArgoCD Application Sync (`Octopus.TriggerArgoCDSync`)
Trigger an Argo CD application sync for all updated applications. Applies the user-configured sync policy if available.
Categories: ArgoCD

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.ArgoCD.Sync.Mode` | yes | `AllEnvironments` | `Disabled`, `AllEnvironments`, `PerEnvironment` | Controls whether sync is triggered and for which environments. 'AllEnvironments' syncs all environments, 'PerEnvironment' syncs only specific environments, and 'Disabled' turns sync off. |
| `Octopus.Action.ArgoCD.Sync.Environments` | no |  |  | A comma-separated list of environment IDs to sync. Only used when the sync mode is set to 'PerEnvironment'. |
| `Octopus.Action.ArgoCD.Sync.EnvironmentsTemplateParameter` | no |  |  | A process template parameter that specifies which environments to sync. Only used when the sync mode is set to 'PerEnvironment' within a process template. |

#### Update Argo CD Application Image Tags (`Octopus.ArgoCDUpdateImageTags`)
Update image tags in your manifests and commit the changes to Git for Argo CD to deploy.
Categories: ArgoCD

_No configurable properties._

#### Update Argo CD Application Manifests (`Octopus.ArgoCDUpdateManifests`)
Generate manifests from templates using Octopus variables and commit new or updated manifests to Git for Argo CD to deploy.
Categories: ArgoCD

_No configurable properties._

#### Verify ArgoCD Application is Healthy (`Octopus.VerifyArgoCDApplicationHealthy`)
Wait for an Argo CD application to sync and become healthy after a deployment.
Categories: ArgoCD

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.ArgoCD.StepVerification.Method` | yes | `ArgoCDApplicationHealthy` | `CommitCreated`, `PullRequestMerged`, `ArgoCDApplicationHealthy` | The method used to determine if the step was successful. 'CommitCreated' verifies that file changes are committed to Git, 'PullRequestMerged' verifies that pull requests are merged, and 'ArgoCDAppl... |
| `Octopus.Action.ArgoCD.StepVerification.Timeout` | no | `180` |  | The maximum time in seconds to wait for all Argo CD applications to sync and become healthy. Only applies when the verification method is 'ArgoCDApplicationHealthy'. Example: `300` |

#### Wait for ArgoCD Pull Requests to be Merged (`Octopus.WaitForArgoCDPullRequestsMerged`)
Verify that all pull requests created during the deployment have been merged successfully.
Categories: ArgoCD

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.ArgoCD.CommitMethod` | yes | `DirectCommit` | `DirectCommit`, `PullRequest`, `PerEnvironment` | The method used to commit changes to Git. 'DirectCommit' commits directly, 'PullRequest' opens pull requests for all environments, and 'PerEnvironment' opens pull requests for specific environments. |
| `Octopus.Action.ArgoCD.PullRequest.Environments` | no |  |  | A comma-separated list of environment IDs for which pull requests should be created. Only used when the commit method is set to 'PerEnvironment'. |
| `Octopus.Action.ArgoCD.PullRequest.EnvironmentsTemplateParameter` | no |  |  | A process template parameter that specifies which environments should have pull requests created. Only used when the commit method is 'PerEnvironment' within a process template. |
| `Octopus.Action.ArgoCD.StepVerification.Method` | yes | `PullRequestMerged` | `CommitCreated`, `PullRequestMerged`, `ArgoCDApplicationHealthy` | The method used to determine if the step was successful. Must be set to 'PullRequestMerged' to wait for pull request merges. Falls back to direct commit created for environments without pull requests. |
| `Octopus.Action.ArgoCD.StepVerification.Timeout` | no |  |  | The maximum time in seconds to wait for all pull requests to be merged. Example: `300` |

### Atlassian

#### Jira Service Desk Change Request (`Octopus.JiraIntegration.ServiceDeskAction`)
Initiate a Change Request in Jira Service Desk during a deployment.
Categories: Atlassian, Other

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.JiraIntegration.ServiceDesk.ServiceId` | yes |  |  | The Jira Service Desk Service Id used to create the change request. (sensitive) |

### AWS

#### Apply a CloudFormation Change Set (`Octopus.AwsApplyCloudFormationChangeSet`)
Apply a known CloudFormation change set to an existing CloudFormation stack.
Categories: AWS, CloudFormation

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.AwsAccount.UseInstanceRole` | no | `False` | `True`, `False` | Execute using the AWS service role for an EC2 instance. |
| `Octopus.Action.AwsAccount.Variable` | no |  |  | The AWS account variable to use for authentication. |
| `Octopus.Action.Aws.AssumeRole` | no | `False` | `True`, `False` | Whether to assume a different AWS IAM role for executing this step. |
| `Octopus.Action.Aws.AssumedRoleArn` | no |  |  | The Amazon Resource Name (ARN) of the IAM role to assume. Example: `arn:aws:iam::123456789012:role/ExampleRole` |
| `Octopus.Action.Aws.AssumedRoleSession` | no |  |  | A name for the assumed role session. Leave blank to use an automatically generated session name. |
| `Octopus.Action.Aws.AssumeRoleSessionDurationSeconds` | no |  |  | The duration in seconds for the assumed role session. If blank, defaults to 3600 seconds (1 hour). AWS requires a session duration between 900 seconds (15 minutes) and 43200 seconds (12 hours). Example: `3600` |
| `Octopus.Action.Aws.AssumeRoleExternalId` | no |  |  | An external ID to provide additional security when assuming the role. |
| `Octopus.Action.Aws.Region` | yes |  |  | The AWS region code where the CloudFormation stack is located. Example: `us-west-2` |
| `Octopus.Action.Aws.CloudFormationStackName` | yes |  |  | The name of the CloudFormation stack to apply the change set to. Example: `my-app-stack` |
| `Octopus.Action.Aws.CloudFormation.ChangeSet.Arn` | yes |  |  | The name or ARN of the change set to apply to the CloudFormation stack. |
| `Octopus.Action.Aws.WaitForCompletion` | no | `True` | `True`, `False` | Wait until the CloudFormation stack has finished applying the change set before finishing the step. Disabling this means the step will not indicate an error if the change set was not applied succes... |

#### Create an S3 Bucket (`Octopus.AwsCreateS3`)
Create an Amazon S3 bucket with optional configuration for tags, public access, and object ownership.
Categories: AWS, S3

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Aws.S3.BucketName` | yes |  |  | The name of the S3 bucket to be created. Must be between 3 and 63 characters. Only lower case letters, numbers, dots, and hyphens are allowed. Example: `my-new-bucket` |
| `Octopus.Action.Aws.Region` | yes |  |  | The AWS region where the S3 bucket will be created. Example: `us-west-2` |
| `Octopus.Action.AwsAccount.UseInstanceRole` | no | `True` | `True`, `False` | Execute using credentials configured on the worker rather than a specific AWS account. |
| `Octopus.Action.AwsAccount.Variable` | no |  |  | The AWS account to use for authentication when not using worker credentials. |
| `Octopus.Action.Aws.AssumeRole` | no | `False` | `True`, `False` | Whether to assume a different AWS IAM role for executing this step. |
| `Octopus.Action.Aws.AssumedRoleArn` | no |  |  | The Amazon Resource Name (ARN) of the IAM role to assume. Example: `arn:aws:iam::123456789012:role/ExampleRole` |
| `Octopus.Action.Aws.AssumedRoleSession` | no |  |  | A name for the assumed role session. Leave blank to use an automatically generated session name. |
| `Octopus.Action.Aws.AssumeRoleSessionDurationSeconds` | no |  |  | The duration in seconds for the assumed role session. Leave blank to use the default session duration on the role being assumed. Example: `3600` |
| `Octopus.Action.Aws.AssumeRoleExternalId` | no |  |  | An external ID to provide additional security when assuming the role. |
| `Octopus.Action.Aws.CloudFormation.Tags` | no |  |  | Tags to apply to the S3 bucket as a JSON array of key-value pairs. Tag value and key have basic restrictions applied. Example: `[{"key":"Environment","value":"Production"}]` |
| `Octopus.Action.Aws.CloudFormationStackName` | no |  |  | Optionally enter the name of the CloudFormation stack used by this deployment. If left blank, Octopus will generate a unique name. Up to 255 letters (uppercase and lowercase), numbers, hyphens, and... |
| `Octopus.Action.Aws.S3.PublicAccess` | no | `False` | `True`, `False` | Enable public access control on the S3 bucket. This setting is required when setting non-private Canned ACLs on objects. |
| `Octopus.Action.Aws.S3.ObjectWriterOwnership` | no | `False` | `True`, `False` | If enabled, the account creating the object will be the owner. If disabled, the bucket owner will be the object owner. |

#### Delete an AWS CloudFormation Stack (`Octopus.AwsDeleteCloudFormation`)
Delete an existing AWS CloudFormation stack.
Categories: AWS, CloudFormation

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.AwsAccount.UseInstanceRole` | no | `False` | `True`, `False` | Execute using the AWS service role for an EC2 instance. |
| `Octopus.Action.AwsAccount.Variable` | no |  |  | The AWS account variable to use for authentication. |
| `Octopus.Action.Aws.AssumeRole` | no | `False` | `True`, `False` | Whether to assume a different AWS IAM role for executing this step. |
| `Octopus.Action.Aws.AssumedRoleArn` | no |  |  | The Amazon Resource Name (ARN) of the IAM role to assume. Example: `arn:aws:iam::123456789012:role/ExampleRole` |
| `Octopus.Action.Aws.AssumedRoleSession` | no |  |  | A name for the assumed role session. Leave blank to use an automatically generated session name. |
| `Octopus.Action.Aws.AssumeRoleSessionDurationSeconds` | no |  |  | The duration in seconds for the assumed role session. If blank, defaults to 3600 seconds (1 hour). AWS requires a session duration between 900 seconds (15 minutes) and 43200 seconds (12 hours). Example: `3600` |
| `Octopus.Action.Aws.AssumeRoleExternalId` | no |  |  | An external ID to provide additional security when assuming the role. |
| `Octopus.Action.Aws.Region` | yes |  |  | The AWS region code where the CloudFormation stack is located. Example: `us-west-2` |
| `Octopus.Action.Aws.CloudFormationStackName` | yes |  |  | The name of the CloudFormation stack to delete. Example: `my-app-stack` |
| `Octopus.Action.Aws.WaitForCompletion` | no | `True` | `True`, `False` | Wait until the CloudFormation stack has been deleted before finishing the step. Disabling this means the step will not indicate an error if the stack was not removed successfully. |

#### Deploy Amazon ECS Service (`Octopus.AwsDeployEcsService`)
Deploys a service to an Amazon ECS cluster via a CloudFormation template generated from ECS step inputs.
Categories: AWS, ECS

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.AwsAccount.UseInstanceRole` | no | `False` | `True`, `False` | Execute using the AWS service role for an EC2 instance. |
| `Octopus.Action.AwsAccount.Variable` | no |  |  | The AWS account variable to use for authentication. |
| `Octopus.Action.Aws.AssumeRole` | no | `False` | `True`, `False` | Whether to assume a different AWS IAM role for executing this step. |
| `Octopus.Action.Aws.AssumedRoleArn` | no |  |  | The Amazon Resource Name (ARN) of the IAM role to assume. Example: `arn:aws:iam::123456789012:role/ExampleRole` |
| `Octopus.Action.Aws.AssumedRoleSession` | no |  |  | A name for the assumed role session. Leave blank to use an automatically generated session name. |
| `Octopus.Action.Aws.AssumeRoleSessionDurationSeconds` | no |  |  | The duration in seconds for the assumed role session. If blank, defaults to 3600 seconds (1 hour). AWS requires a session duration between 900 seconds (15 minutes) and 43200 seconds (12 hours). Example: `3600` |
| `Octopus.Action.Aws.AssumeRoleExternalId` | no |  |  | An external ID to provide additional security when assuming the role. |
| `Octopus.Action.Aws.Region` | yes |  |  | The default AWS region code for this step. Example: `us-west-2` |
| `Octopus.Action.Aws.Ecs.ClusterName` | yes |  |  | The name of the ECS cluster the service is deployed into. |
| `Octopus.Action.Aws.Ecs.ServiceName` | yes |  |  | The name of the ECS service. Also used as the task definition family name and to label post-deployment task diagnostics. |
| `Octopus.Action.Aws.CloudFormationStackName` | yes |  |  | The CloudFormation stack name used to deploy the ECS service. |
| `Octopus.Action.Aws.TemplateSource` | yes | `Inline` | `Inline` | The source of the CloudFormation template. Always Inline for this step; the template is generated by the step package mapper from the ECS step inputs. |
| `Octopus.Action.Aws.CloudFormationTemplate` | yes |  |  | The inline CloudFormation template body that deploys the ECS task definition and service. Generated by the step package mapper. |
| `Octopus.Action.Aws.CloudFormationTemplateParameters` | no |  |  | CloudFormation template parameters as a JSON array of ParameterKey/ParameterValue pairs. Example: `[{"ParameterKey":"ClusterName","ParameterValue":"my-cluster"}]` |
| `Octopus.Action.Aws.Ecs.WaitOption.Type` | yes | `waitUntilCompleted` | `waitUntilCompleted`, `waitWithTimeout`, `dontWait` | Whether and how to wait for the stack and ECS task deployment to complete. |
| `Octopus.Action.Aws.Ecs.WaitOption.Timeout` | no |  |  | Timeout in milliseconds. Required when Wait Option is waitWithTimeout; otherwise ignored. |
| `Octopus.Action.Aws.CloudFormation.Tags` | no |  |  | User-supplied stack tags as a JSON array of key-value pairs. Calamari merges these with 19 default Octopus tags (project/release/environment/tenant identifiers and names) and propagates them to all... Example: `[{"key":"Environment","value":"Production"}]` |

#### Deploy an AWS CloudFormation Template (`Octopus.AwsRunCloudFormation`)
Deploy an AWS CloudFormation template to create or update a CloudFormation stack.
Categories: AWS, CloudFormation

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.AwsAccount.UseInstanceRole` | no | `False` | `True`, `False` | Execute using the AWS service role for an EC2 instance. |
| `Octopus.Action.AwsAccount.Variable` | no |  |  | The AWS account variable to use for authentication. |
| `Octopus.Action.Aws.AssumeRole` | no | `False` | `True`, `False` | Whether to assume a different AWS IAM role for executing this step. |
| `Octopus.Action.Aws.AssumedRoleArn` | no |  |  | The Amazon Resource Name (ARN) of the IAM role to assume. Example: `arn:aws:iam::123456789012:role/ExampleRole` |
| `Octopus.Action.Aws.AssumedRoleSession` | no |  |  | A name for the assumed role session. Leave blank to use an automatically generated session name. |
| `Octopus.Action.Aws.AssumeRoleSessionDurationSeconds` | no |  |  | The duration in seconds for the assumed role session. If blank, defaults to 3600 seconds (1 hour). AWS requires a session duration between 900 seconds (15 minutes) and 43200 seconds (12 hours). Example: `3600` |
| `Octopus.Action.Aws.AssumeRoleExternalId` | no |  |  | An external ID to provide additional security when assuming the role. |
| `Octopus.Action.Aws.Region` | yes |  |  | The default AWS region code for this step. Example: `us-west-2` |
| `Octopus.Action.Aws.CloudFormationStackName` | yes |  |  | The name of the CloudFormation stack to create or update. Example: `my-app-stack` |
| `Octopus.Action.Aws.CloudFormation.RoleArn` | no |  |  | The Amazon Resource Name (ARN) of an AWS Identity and Access Management (IAM) role that AWS CloudFormation assumes when executing any operations. This role will be used for any future operations on... Example: `arn:aws:iam::123456789012:role/CloudFormationRole` |
| `Octopus.Action.Aws.IamCapabilities` | no |  | `CAPABILITY_IAM`, `CAPABILITY_NAMED_IAM`, `CAPABILITY_AUTO_EXPAND` | Additional capabilities required for templates that have IAM resources or named IAM resources. Specified as a JSON array. Example: `["CAPABILITY_IAM","CAPABILITY_NAMED_IAM"]` |
| `Octopus.Action.Aws.CloudFormation.Tags` | no |  |  | Stack-level tags to be propagated to resources that CloudFormation supports. Specified as a JSON array of key-value pairs. If no tags are specified and the stack already exists, current tags will b... Example: `[{"key":"Environment","value":"Production"}]` |
| `Octopus.Action.Aws.DisableRollback` | no | `False` | `True`, `False` | Disable the automatic rollback of a CloudFormation stack if it failed to be created successfully. This has no effect if the stack exists and is being updated. |
| `Octopus.Action.Aws.WaitForCompletion` | no | `True` | `True`, `False` | Wait until the CloudFormation stack operation has completed before finishing the step. Disabling this means output variables may not be created or may contain outdated values, and the step will not... |
| `Octopus.Action.Aws.TemplateSource` | yes | `Inline` | `Inline`, `Package`, `S3URL`, `GitRepository` | The source of the CloudFormation template. Templates can be entered as source code, contained in a package, a Git repository, or referenced by S3 URL. |
| `Octopus.Action.Aws.CloudFormationTemplate` | no |  |  | The CloudFormation template content (when source is Inline) or the relative path to the template file in the package or Git repository. |
| `Octopus.Action.Aws.CloudFormationTemplateParametersRaw` | no |  |  | The CloudFormation template parameters as a JSON array of ParameterKey/ParameterValue pairs, or the relative path to the parameters file when using a package or Git repository source. Example: `[{"ParameterKey":"InstanceType","ParameterValue":"t2.micro"}]` |
| `Octopus.Action.Aws.CloudFormationTemplateS3URL` | no |  |  | The S3 URL to the CloudFormation template. Templates hosted in S3 are not processed by Octopus and cannot use variable substitutions. Example: `https://s3.amazonaws.com/mybucket/mytemplate.yml` |
| `Octopus.Action.Aws.CloudFormationParametersS3URL` | no |  |  | The S3 URL to the CloudFormation parameters file. Parameter files hosted in S3 are downloaded and processed during deployment and can use variable substitutions. Example: `https://s3.amazonaws.com/mybucket/myparams.json` |
| `Octopus.Action.Aws.CloudFormation.ChangeSet.GenerateName` | no | `True` | `True`, `False` | Whether to automatically generate a unique change set name or manually specify one. Only applies when CloudFormation change sets are enabled. |
| `Octopus.Action.Aws.CloudFormation.ChangeSet.Name` | no |  |  | The name of the change set. Change set names must be unique for a given stack. Only applies when change sets are enabled and automatic name generation is disabled. |
| `Octopus.Action.Aws.CloudFormation.ChangeSet.Defer` | no | `False` | `True`, `False` | Whether to defer change set execution or execute it immediately. Only applies when change sets are enabled. |

#### Run an AWS CLI Script (`Octopus.AwsRunScript`)
Log into AWS and run a script with the AWS CLI.
Categories: AWS

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.AwsAccount.UseInstanceRole` | no | `False` | `True`, `False` | Execute using the AWS service role for an EC2 instance. |
| `Octopus.Action.AwsAccount.Variable` | no |  |  | The AWS account variable to use for authentication. |
| `Octopus.Action.Aws.AssumeRole` | no | `False` | `True`, `False` | Whether to assume a different AWS IAM role for executing this step. |
| `Octopus.Action.Aws.AssumedRoleArn` | no |  |  | The Amazon Resource Name (ARN) of the IAM role to assume. Example: `arn:aws:iam::123456789012:role/ExampleRole` |
| `Octopus.Action.Aws.AssumedRoleSession` | no |  |  | A name for the assumed role session. Leave blank to use an automatically generated session name. |
| `Octopus.Action.Aws.AssumeRoleSessionDurationSeconds` | no |  |  | The duration in seconds for the assumed role session. If blank, defaults to 3600 seconds (1 hour). AWS requires a session duration between 900 seconds (15 minutes) and 43200 seconds (12 hours). Example: `3600` |
| `Octopus.Action.Aws.AssumeRoleExternalId` | no |  |  | An external ID to provide additional security when assuming the role. |
| `Octopus.Action.Aws.Region` | yes |  |  | The default AWS region code for this step. Example: `us-west-2` |
| `Octopus.Action.Script.ScriptSource` | yes |  | `Inline`, `Package`, `GitRepository` | The source of the script to run. |
| `Octopus.Action.Script.ScriptBody` | no |  |  | The inline script body to execute when the script source is set to Inline. |
| `Octopus.Action.Script.Syntax` | no |  | `PowerShell`, `Bash` | The scripting language to use. Required when the script source is Inline. |
| `Octopus.Action.Script.ScriptFileName` | no |  |  | The path to the script file inside the package or Git repository. Required when the script source is Package or GitRepository. Example: `Deploy.sh` |
| `Octopus.Action.Script.ScriptParameters` | no |  |  | Parameters to pass to the script. |
| `OctopusUseBundledTooling` | no | `False` | `True`, `False` | Select whether to use the bundled AWS tools or tooling pre-installed on the worker. Using bundled tools is no longer supported. |

#### Update Amazon ECS Service (`Octopus.AwsUpdateEcsService`)
Update an existing Amazon ECS Service using the ECS API
Categories: AWS, ECS

_No configurable properties._

#### Upload a Package to an S3 Bucket (`Octopus.AwsUploadS3`)
Upload a package or specific files within a package to an AWS S3 bucket.
Categories: AWS, S3

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.AwsAccount.UseInstanceRole` | no | `False` | `True`, `False` | Execute using the AWS service role for an EC2 instance. |
| `Octopus.Action.AwsAccount.Variable` | no |  |  | The AWS account variable to use for authentication. |
| `Octopus.Action.Aws.AssumeRole` | no | `False` | `True`, `False` | Whether to assume a different AWS IAM role for executing this step. |
| `Octopus.Action.Aws.AssumedRoleArn` | no |  |  | The Amazon Resource Name (ARN) of the IAM role to assume. Example: `arn:aws:iam::123456789012:role/ExampleRole` |
| `Octopus.Action.Aws.AssumedRoleSession` | no |  |  | A name for the assumed role session. Leave blank to use an automatically generated session name. |
| `Octopus.Action.Aws.AssumeRoleSessionDurationSeconds` | no |  |  | The duration in seconds for the assumed role session. If blank, defaults to 3600 seconds (1 hour). AWS requires a session duration between 900 seconds (15 minutes) and 43200 seconds (12 hours). Example: `3600` |
| `Octopus.Action.Aws.AssumeRoleExternalId` | no |  |  | An external ID to provide additional security when assuming the role. |
| `Octopus.Action.Aws.Region` | yes |  |  | The default AWS region code for this step. Example: `us-west-2` |
| `Octopus.Action.Aws.S3.BucketName` | yes |  |  | The name of the S3 bucket to upload to. Example: `my-deployment-bucket` |
| `Octopus.Action.Package.PackageId` | yes |  |  | The ID of the package to upload. |
| `Octopus.Action.Package.FeedId` | yes |  |  | The feed that contains the package to upload. |
| `Octopus.Action.Aws.S3.TargetMode` | no | `EntirePackage` | `EntirePackage`, `FileSelections` | Choose how the package should be uploaded: as an entire package file or as specific file(s) within the package. |
| `Octopus.Action.Aws.S3.PackageOptions` | no |  |  | A JSON object specifying options for uploading the entire package including bucket key behaviour, storage class, canned ACL, tags, and metadata. |
| `Octopus.Action.Aws.S3.FileSelections` | no |  |  | A JSON array of file selection configurations specifying which files to upload, their bucket key settings, storage class, canned ACL, tags, and metadata. Each selection can target a single file or ... |

### Azure

#### Deploy a Bicep template (Native) (`Octopus.AzureDeployBicepTemplate`)
Deploy a Bicep template (Native)
Categories: BuiltInStep, Azure

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `OctopusUseBundledTooling` | no | `False` | `True`, `False` | Select whether to use the bundled Azure tools or tooling pre-installed on the worker when running custom scripts. |
| `Octopus.Action.Azure.AccountId` | yes |  |  | Select the account to use for the connection. Example: `azure-account-1` |
| `Octopus.Action.Azure.ResourceGroupName` | yes |  |  | The name of the Azure resource group to deploy the template to. Example: `my-resource-group` |
| `Octopus.Action.Azure.ResourceGroupDeploymentMode` | yes | `Incremental` | `Incremental`, `Complete` | The deployment mode to use when deploying the Bicep template. |
| `Octopus.Action.Azure.TemplateSource` | yes | `Inline` | `Inline`, `Package`, `GitRepository` | Select the source of the template. Templates can be entered as source code, contained in a package or a Git repository. |
| `Octopus.Action.Azure.BicepTemplateSourceCode` | no |  |  | The Bicep template source code when using inline source, or the relative path to the Bicep template file in the package or Git repository. Example: `template.bicep` |
| `Octopus.Action.Azure.BicepTemplateParameters` | no |  |  | The template parameters JSON when using inline source, or the relative path to the JSON parameters file in the package or Git repository. Example: `parameters.json` |

#### Deploy a Service Fabric Application (`Octopus.AzureServiceFabricApp`)
Deploy a Service Fabric application package to an Azure Service Fabric cluster.
Categories: Azure, Service Fabric, Deployment

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `OctopusUseBundledTooling` | no | `False` | `True`, `False` | Select whether to use the bundled Azure tools or tooling pre-installed on the worker when running custom scripts. |
| `Octopus.Action.ServiceFabric.ConnectionEndpoint` | no |  |  | The connection endpoint of the Service Fabric cluster. Used in legacy mode only. Example: `tcp://mycluster.eastus.cloudapp.azure.com:19000` |
| `Octopus.Action.ServiceFabric.SecurityMode` | no | `Unsecure` | `Unsecure`, `SecureClientCertificate`, `SecureAzureAD` | The security mode used to connect to the Service Fabric cluster. Used in legacy mode only. |
| `Octopus.Action.ServiceFabric.ServerCertThumbprint` | no |  |  | The server certificate thumbprint used to communicate with the secure cluster. |
| `Octopus.Action.ServiceFabric.ClientCertVariable` | no |  |  | The client certificate variable used to communicate with the secure cluster when using client certificate security mode. (sensitive) |
| `Octopus.Action.ServiceFabric.AadCredentialType` | no | `UserCredential` | `UserCredential`, `ClientCredential` | The Azure Active Directory credential type when using AAD security mode. |
| `Octopus.Action.ServiceFabric.AadClientCredentialSecret` | no |  |  | The client application secret used to communicate with the secure cluster via AAD client credentials. (sensitive) |
| `Octopus.Action.ServiceFabric.AadUserCredentialUsername` | no |  |  | The Azure AD user's username used to communicate with the secure cluster. |
| `Octopus.Action.ServiceFabric.AadUserCredentialPassword` | no |  |  | The Azure AD user's password used to communicate with the secure cluster. (sensitive) |
| `Octopus.Action.ServiceFabric.PublishProfileFile` | no |  |  | The path to the publish profile file. Octopus will use the ApplicationParameters file referenced in this publish profile. Example: `PublishProfiles\Cloud.xml` |
| `Octopus.Action.ServiceFabric.DeployOnly` | no | `False` | `True`, `False` | If selected, the Service Fabric application will not be created or upgraded after registering the application type. |
| `Octopus.Action.ServiceFabric.UnregisterUnusedApplicationVersionsAfterUpgrade` | no | `False` | `True`, `False` | Select to unregister any unused application versions that exist after an upgrade is finished. |
| `Octopus.Action.ServiceFabric.OverrideUpgradeBehavior` | no | `None` | `None`, `ForceUpgrade`, `VetoUpgrade` | Indicates the behavior used to override the upgrade settings specified by the publish profile. |
| `Octopus.Action.ServiceFabric.OverwriteBehavior` | no | `SameAppTypeAndVersion` | `Never`, `Always`, `SameAppTypeAndVersion` | Overwrite behavior if an application exists in the cluster with the same name. This setting is not applicable when upgrading an application. |
| `Octopus.Action.ServiceFabric.SkipPackageValidation` | no | `False` | `True`, `False` | Select to tell Service Fabric not to validate the package before deployment. |
| `Octopus.Action.ServiceFabric.CopyPackageTimeoutSec` | no |  |  | A value in seconds to override the timeout for copying an application package to the image store. Example: `600` |
| `Octopus.Action.ServiceFabric.RegisterApplicationTypeTimeoutSec` | no |  |  | A value in seconds to override the timeout for registering application type. Example: `600` |
| `Octopus.Action.ServiceFabric.IsLegacyMode` | no | `False` | `True`, `False` | Select legacy mode if you wish to configure connection-related properties on the step and not through Azure Targets. |

#### Deploy an ARM Template (`Octopus.AzureResourceGroup`)
Deploy an Azure Resource Manager (ARM) template to a resource group.
Categories: Azure, Deployment

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `OctopusUseBundledTooling` | no | `False` | `True`, `False` | Select whether to use the bundled Azure tools or tooling pre-installed on the worker when running custom scripts. |
| `Octopus.Action.Azure.AccountId` | yes |  |  | Select the account to use for the connection. Example: `azure-account-1` |
| `Octopus.Action.Azure.ResourceGroupName` | yes |  |  | The name of the Azure resource group to deploy the template to. Example: `my-resource-group` |
| `Octopus.Action.Azure.ResourceGroupDeploymentMode` | yes | `Incremental` | `Incremental`, `Complete` | The deployment mode to use when deploying the ARM template. |
| `Octopus.Action.Azure.TemplateSource` | yes | `Inline` | `Inline`, `Package`, `GitRepository` | Select the source of the template. Templates can be entered as source code, contained in a package or a Git repository. |
| `Octopus.Action.Azure.ResourceGroupTemplate` | no |  |  | The ARM template JSON when using inline source, or the relative path to the JSON template file in the package or Git repository. Example: `template.json` |
| `Octopus.Action.Azure.ResourceGroupTemplateParameters` | no |  |  | The ARM template parameters JSON when using inline source, or the relative path to the JSON parameters file in the package or Git repository. Example: `parameters.json` |

#### Deploy an Azure App Service (`Octopus.AzureAppService`)
Deploy a package or container image to an Azure App Service using Zip Deploy.
Categories: Azure, Deployment

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `OctopusUseBundledTooling` | no | `False` | `True`, `False` | Select whether to use the bundled Azure tools or tooling pre-installed on the worker when running custom scripts. |
| `Octopus.Action.Azure.DeploymentType` | no | `Package` | `Package`, `Container` | Select whether to deploy from a package (zip, WAR, NuGet) or a container image. |
| `Octopus.Action.Azure.DeploymentSlot` | no |  |  | The slot to deploy to. Slots allow deploying different versions of your app to the same app service. Octopus variable expressions can be used. Example: `#{DeploymentSlot}` |
| `Octopus.Action.Azure.AppSettings` | no |  |  | App Service Application Settings provided as JSON, using the same schema as the bulk edit in the Azure portal. Octopus variable-binding syntax may be used. Example: `[{"name": "Greeting", "value": "#{Greeting}", "slotSetting": false}]` |
| `Octopus.Action.Azure.ConnectionStrings` | no |  |  | Connection strings provided as JSON, using the same schema as the bulk edit in the Azure portal. Octopus variable-binding syntax may be used. Example: `[{"name": "AcmeDB", "value": "#{Acme.Database.ConnectionString}", "type": "SQLAzure", "slotSetting": false}]` |

#### Deploy an Azure Web App (`Octopus.AzureWebApp`)
Deploy a package to an Azure Web App using Web Deploy (MSDeploy).
Categories: Azure, Deployment

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `OctopusUseBundledTooling` | no | `False` | `True`, `False` | Select whether to use the bundled Azure tools or tooling pre-installed on the worker when running custom scripts. |
| `Octopus.Action.Azure.AccountId` | no |  |  | Select the Azure account to use for the deployment. Used in legacy mode only. Example: `azure-account-1` |
| `Octopus.Action.Azure.IsLegacyMode` | no | `False` | `True`, `False` | Select legacy mode if you wish to configure account-related properties on the step and not through Azure Targets. |
| `Octopus.Action.Azure.WebAppName` | yes |  |  | The name of the Azure Web App to deploy to. Example: `my-web-app` |
| `Octopus.Action.Azure.ResourceGroupName` | no |  |  | The name of the resource group containing the Web App. Example: `my-resource-group` |
| `Octopus.Action.Azure.DeploymentSlot` | no |  |  | The deployment slot to deploy to. Slots let you deploy different versions of your web app to different URLs. Example: `staging` |
| `Octopus.Action.Azure.PhysicalPath` | no |  |  | Physical path relative to site root. e.g. 'foo' will deploy to 'site\wwwroot\foo'. Leave blank to deploy to root. Example: `foo` |
| `Octopus.Action.Azure.RemoveAdditionalFiles` | no | `False` | `True`, `False` | Select to remove additional files on the destination that are not part of the deployment. |
| `Octopus.Action.Azure.PreserveAppData` | no | `False` | `True`, `False` | Select to preserve files in the App_Data folder before deployment. |
| `Octopus.Action.Azure.AppOffline` | no | `False` | `True`, `False` | Select to safely bring down the app domain with app_offline.html in root. |
| `Octopus.Action.Azure.UseChecksum` | no | `False` | `True`, `False` | Select which method will be used to determine which files will be updated during deployment. Checksum method may increase deployment time significantly. |

#### Run a Service Fabric SDK PowerShell Script (`Octopus.AzureServiceFabricPowerShell`)
Run a PowerShell script using a Service Fabric cluster context.
Categories: Azure, Service Fabric, Scripts

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `OctopusUseBundledTooling` | no | `False` | `True`, `False` | Select whether to use the bundled Azure tools or tooling pre-installed on the worker. |
| `Octopus.Action.ServiceFabric.ConnectionEndpoint` | no |  |  | The connection endpoint of the Service Fabric cluster. Used in legacy mode only. Example: `tcp://mycluster.eastus.cloudapp.azure.com:19000` |
| `Octopus.Action.ServiceFabric.SecurityMode` | no | `Unsecure` | `Unsecure`, `SecureClientCertificate`, `SecureAzureAD` | The security mode used to connect to the Service Fabric cluster. Used in legacy mode only. |
| `Octopus.Action.ServiceFabric.ServerCertThumbprint` | no |  |  | The server certificate thumbprint used to communicate with the secure cluster. |
| `Octopus.Action.ServiceFabric.ClientCertVariable` | no |  |  | The client certificate variable used to communicate with the secure cluster when using client certificate security mode. (sensitive) |
| `Octopus.Action.ServiceFabric.AadCredentialType` | no | `UserCredential` | `UserCredential`, `ClientCredential` | The Azure Active Directory credential type when using AAD security mode. |
| `Octopus.Action.ServiceFabric.AadClientCredentialSecret` | no |  |  | The client application secret used to communicate with the secure cluster via AAD client credentials. (sensitive) |
| `Octopus.Action.ServiceFabric.AadUserCredentialUsername` | no |  |  | The Azure AD user's username used to communicate with the secure cluster. |
| `Octopus.Action.ServiceFabric.AadUserCredentialPassword` | no |  |  | The Azure AD user's password used to communicate with the secure cluster. (sensitive) |
| `Octopus.Action.Script.ScriptSource` | yes | `Inline` | `Inline`, `Package`, `GitRepository` | The source of the script to run. |
| `Octopus.Action.Script.Syntax` | no | `PowerShell` | `PowerShell` | The scripting language to use. Only PowerShell is supported for Service Fabric scripts. |
| `Octopus.Action.Script.ScriptBody` | no |  |  | The inline script body to execute when script source is Inline. |
| `Octopus.Action.Script.ScriptFileName` | no |  |  | The path to the script file inside the package or Git repository. Example: `Deploy.ps1` |
| `Octopus.Action.Script.ScriptParameters` | no |  |  | Parameters to pass to the script. Example: `-Environment #{Octopus.Environment.Name}` |
| `Octopus.Action.ServiceFabric.IsLegacyMode` | no | `False` | `True`, `False` | Select legacy mode if you wish to configure connection-related properties on the step and not through Azure Targets. |

#### Run an Azure PowerShell Script (`Octopus.AzurePowerShell`)
Run a script using an Azure subscription, with Azure modules loaded by default.
Categories: Azure, Scripts

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Azure.AccountId` | yes |  |  | Select the Azure account to use to run the script. Example: `azure-account-1` |
| `OctopusUseBundledTooling` | no | `False` | `True`, `False` | Select whether to use the bundled Azure tools or tooling pre-installed on the worker. |
| `Octopus.Action.Script.ScriptSource` | yes | `Inline` | `Inline`, `Package`, `GitRepository` | The source of the script to run. |
| `Octopus.Action.Script.Syntax` | no |  | `PowerShell`, `Bash` | The scripting language to use. |
| `Octopus.Action.Script.ScriptBody` | no |  |  | The inline script body to execute when script source is Inline. |
| `Octopus.Action.Script.ScriptFileName` | no |  |  | The path to the script file inside the package or Git repository. Example: `Deploy.ps1` |
| `Octopus.Action.Script.ScriptParameters` | no |  |  | Parameters to pass to the script. Example: `-Environment #{Octopus.Environment.Name}` |

### Certificates

#### Import Certificate (`Octopus.Certificate.Import`)
Import a certificate into the Windows Certificate Store.
Categories: Certificates

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Certificate.Variable` | yes |  |  | The certificate variable to be imported. |
| `Octopus.Action.Certificate.StoreLocation` | no | `LocalMachine` | `LocalMachine`, `CurrentUser`, `Custom User` | The location of the certificate store. |
| `Octopus.Action.Certificate.StoreUser` | no |  |  | A user to use as the certificate store location when Store Location is set to Custom User. Example: `DomainB\UserB` |
| `Octopus.Action.Certificate.StoreName` | no | `My` | `My`, `Root`, `CA`, `TrustedPeople`, `TrustedPublisher`, `AuthRoot`, `AddressBook` | The name of the Windows certificate store. Use one of the pre-defined stores, or enter a custom store name. |
| `Octopus.Action.Certificate.PrivateKeyExportable` | no | `False` | `True`, `False` | If the certificate includes a private key, it will be marked as exportable. |
| `Octopus.Action.Certificate.PrivateKeyAccessRules` | no | `[]` |  | A JSON array defining who has access to the private key on the target machine. Each rule specifies an Identity and an Access type (ReadOnly or FullControl). By default, the machine Administrators g... |

### Discovery

#### Discover Azure Web Apps (`Octopus.TargetDiscovery.AzureWebApp`)
Discover available Azure Web Apps.
Categories: Discovery, Azure

_No configurable properties._

#### Discover Kubernetes Clusters (`Octopus.Action.Kubernetes.Discovery`)
Discover available Kubernetes Clusters.
Categories: Discovery, Kubernetes

_No configurable properties._

### Docker

#### Create a Docker Network (`Octopus.DockerNetwork`)
Create a Docker Network for use by Docker Containers.
Categories: BuiltInStep, Docker

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Docker.NetworkType` | no | `bridge` | `bridge`, `other` | The type of Docker network to create. |
| `Octopus.Action.Docker.NetworkCustomDriver` | no |  |  | Use an installed third party or own custom network driver. Only applicable when Network Type is 'other'. |
| `Octopus.Action.Docker.NetworkName` | no |  |  | Optional network name. If not specified this will be randomly generated. |
| `Octopus.Action.Docker.NetworkSubnet` | no |  |  | Subnet in CIDR format that represents a network segment. On a bridge network you can only specify a single subnet. Specified as a comma-separated list. Example: `172.28.5.0/24` |
| `Octopus.Action.Docker.NetworkIPRange` | no |  |  | Allocate container ip from a sub-range. Specified as a comma-separated list in CIDR format. Example: `172.28.5.0/24` |
| `Octopus.Action.Docker.NetworkGateway` | no |  |  | IPv4 or IPv6 gateway for the master subnet. By default the Docker engine will select one from inside a preferred pool. Specified as a comma-separated list. Example: `172.28.5.1` |
| `Octopus.Action.Docker.Args` | no |  |  | Additional arguments that will be passed to the docker network create command. |

#### Run a Docker Container (`Octopus.DockerRun`)
Run a Docker Container sourced from a Container Registry.
Categories: BuiltInStep, Docker

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Package.PackageId` | yes |  |  | The Docker image to run. This step is used to run a docker container sourced from a container registry. Example: `my-app:latest` |
| `Octopus.Action.Package.FeedId` | yes |  |  | The container registry feed from which the Docker image will be sourced. Example: `feeds-docker-hub` |
| `Octopus.Action.Docker.NetworkType` | no |  | ``, `none`, `bridge`, `host`, `container`, `network` | Connect a container to a network. |
| `Octopus.Action.Docker.NetworkContainer` | no |  |  | Use the network stack of another container, specified via its name or id. Only applicable when Network Type is 'container'. |
| `Octopus.Action.Docker.NetworkName` | no |  |  | Connects the container to a user created network. Only applicable when Network Type is 'network'. |
| `Octopus.Action.Docker.NetworkAlias` | no |  |  | Add network-scoped alias for the container. Only applicable when Network Type is 'network'. |
| `Octopus.Action.Docker.PortMapping` | no |  |  | Publish a container's port or a range of ports to the host. Specified as a JSON object mapping container ports to host ports. Example: `{"8080":"80"}` |
| `Octopus.Action.Docker.PortAutoMap` | no | `False` | `True`, `False` | Allows mapping exposed network port in the container to ports on the host. |
| `Octopus.Action.Docker.AddedHost` | no |  |  | Adds a line to /etc/hosts. Specified as a JSON object mapping IP addresses to host names. Example: `{"127.0.0.1":"myhost"}` |
| `Octopus.Action.Docker.VolumeBindings` | no |  |  | A data volume is a specially-designated directory within one or more containers that bypasses the Union File System. Specified as a JSON object mapping container paths to host path, readOnly, and n... Example: `{"/container/path":{"host":"/host/path","readOnly":"False","noCopy":"False"}}` |
| `Octopus.Action.Docker.VolumeDriver` | no |  |  | Optional volume driver for the container. |
| `Octopus.Action.Docker.VolumesFrom` | no |  |  | Mount all volumes from the given container(s). Specified as a comma-separated list of container names. Example: `container1,container2` |
| `Octopus.Action.Docker.EnvVariable` | no |  |  | Passes through variables into the container accessible as environment variables. Specified as a JSON object mapping variable names to values. Example: `{"ENV_VAR":"value"}` |
| `Octopus.Action.Docker.RestartPolicy` | no | `no` | `no`, `on-failure`, `always`, `unless-stopped` | Restart policy to apply when a container exits. |
| `Octopus.Action.Docker.RestartPolicyMax` | no |  |  | Maximum number of retry attempts to start a container. Only applicable when Restart Policy is 'on-failure'. Example: `4` |
| `Octopus.Action.Docker.DontRun` | no | `False` | `True`, `False` | Perform 'create' command instead of 'run'. This creates the writable layer on top of the image and prepares it for running without actually starting it. |
| `Octopus.Action.Docker.Command` | no |  |  | Override default CMD instruction provided by the image. Example: `echo env` |
| `Octopus.Action.Docker.Args` | no |  |  | Additional arguments that will be passed to the docker run command. Example: `-P -e key=value` |

#### Stop a Docker Resource (`Octopus.DockerStop`)
Stops or Removes a Docker Resource created by a previous deployment.
Categories: BuiltInStep, Docker

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Docker.Remove` | no | `False` | `False`, `True` | Action to take place. 'False' to stop only, 'True' to stop and remove the resource. |
| `Octopus.Action.Docker.RemoveSteps` | no |  |  | Resources created from these Docker steps should be stopped (or stopped and removed). Specified as a comma-separated list of step action IDs. If left blank, resources from all Docker steps will be ... |
| `Octopus.Action.Docker.RemoveByEnvironment` | no | `False` | `True`, `False` | Resources will be removed only where they also match the current environment. |
| `Octopus.Action.Docker.RemoveByRelease` | no | `False` | `True`, `False` | Resources will be removed only where they also match the current release. |
| `Octopus.Action.Docker.RemoveByTenant` | no | `False` | `True`, `False` | If the release includes tenants then resources will be removed only where they also match the currently deploying tenant. |
| `Octopus.Action.Docker.RemoveCustomTags` | no |  |  | Tags included when searching for resources using the '--filter "label=X"' argument. If the value is not provided, the resource will be included so long as the tag is present with any value. Specifi... Example: `{"app":"myapp"}` |
| `Octopus.Action.Docker.StopTimeout` | no | `10` |  | Seconds to wait for process to stop before killing it. The main process inside the container will receive SIGTERM, and after a grace period, SIGKILL. Example: `10` |
| `Octopus.Action.Docker.RemoveAll` | no | `False` | `True`, `False` | If set, removes all matching Docker resources regardless of other filters. |

### Google Cloud

#### Run gcloud in a Script (`Octopus.GoogleCloudScripting`)
Run a script with the gcloud CLI, authenticated against a Google Cloud account.
Categories: Google Cloud, Script

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.GoogleCloud.UseVMServiceAccount` | yes | `True` | `True`, `False` | When running in a Compute Engine virtual machine, use the associated VM service account for authentication. |
| `Octopus.Action.GoogleCloudAccount.Variable` | no |  |  | The variable that references the Google Cloud account to use for authentication. Required when not using a VM service account. Example: `GoogleCloudAccount` |
| `Octopus.Action.GoogleCloud.ImpersonateServiceAccount` | yes | `False` | `True`, `False` | Whether to impersonate a service account when executing the script. |
| `Octopus.Action.GoogleCloud.ServiceAccountEmails` | no |  |  | A comma-separated list of service account emails to impersonate, forming a delegation chain. Required when impersonating service accounts. Example: `my-service-account@my-project.iam.gserviceaccount.com` |
| `Octopus.Action.GoogleCloud.Project` | no |  |  | The default Google Cloud project. Sets the CLOUDSDK_CORE_PROJECT environment variable. Example: `my-gcp-project` |
| `Octopus.Action.GoogleCloud.Region` | no |  |  | The default Google Cloud region. Sets the CLOUDSDK_COMPUTE_REGION environment variable. Example: `us-central1` |
| `Octopus.Action.GoogleCloud.Zone` | no |  |  | The default Google Cloud zone. Sets the CLOUDSDK_COMPUTE_ZONE environment variable. Example: `us-central1-a` |
| `Octopus.Action.Script.ScriptSource` | yes | `Inline` | `Inline`, `Package`, `GitRepository` | The source of the script to run. |
| `Octopus.Action.Script.Syntax` | no |  | `PowerShell`, `Bash` | The scripting language to use. Required when the script source is Inline. |
| `Octopus.Action.Script.ScriptBody` | no |  |  | The inline script body to execute when the script source is set to Inline. |
| `Octopus.Action.Script.ScriptFileName` | no |  |  | The path to the script file inside the package or Git repository. Example: `Deploy.sh` |
| `Octopus.Action.Script.ScriptParameters` | no |  |  | Parameters to pass to the script. |

### HealthCheck

#### HealthCheck an Azure ServiceFabric cluster (`Octopus.HealthCheck.AzureServiceFabricApp`)
HealthCheck an Azure ServiceFabric cluster.
Categories: HealthCheck, Azure

_No configurable properties._

#### HealthCheck an Azure Web App (`Octopus.HealthCheck.AzureWebApp`)
HealthCheck an Azure Web App.
Categories: HealthCheck, Azure

_No configurable properties._

#### Run a health check on a Kubernetes target (`Octopus.HealthCheck.Kubernetes`)
Runs kubectl to check the health of a Kubernetes target
Categories: HealthCheck, Kubernetes

_No configurable properties._

### Java

#### Deploy Certificate to Tomcat (`Octopus.TomcatDeployCertificate`)
Deploy a certificate to Tomcat 7+ by configuring the server.xml file.
Categories: Java, Certificates

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Tomcat.Certificate.CatalinaHome` | yes |  |  | The root directory of the binary distribution of Tomcat. |
| `Tomcat.Certificate.CatalinaBase` | no |  |  | The root directory of the active configuration of Tomcat. Leave blank if CATALINA_HOME and CATALINA_BASE directories are the same. |
| `Java.Certificate.Variable` | yes |  |  | The certificate variable to deploy. |
| `Tomcat.Certificate.Service` | no | `Catalina` |  | The name of the Service element in the server.xml file. Example: `Catalina` |
| `Tomcat.Certificate.Implementation` | no | `NIO` | `BIO`, `NIO`, `NIO2`, `APR` | The standard SSL implementation provided by Tomcat to support HTTPS. Not all versions of Tomcat support all implementations. |
| `Tomcat.Certificate.Port` | no |  |  | The HTTPS port that this certificate will be bound to. Example: `8443` |
| `Tomcat.Certificate.Hostname` | no |  |  | In Tomcat 8.5 and above, certificates can be bound to a host name using SNI. Leave blank when deploying to Tomcat 7 or 8. |
| `Tomcat.Certificate.Default` | no | `False` | `True`, `False` | In Tomcat 8.5 and above, a certificate can be designated as the default one to respond to any request to a host that does not have a specific certificate assigned. This field has no effect when dep... |
| `Java.Certificate.Password` | no |  |  | An optional password assigned to the private key. If left blank, the default password defined by Tomcat will be used where possible, or the private key will remain unencrypted. (sensitive) |
| `Tomcat.Certificate.PrivateKeyFilename` | no |  |  | The optional path of the private key file generated when configuring the APR implementation. If left blank, a new file will be created in the Tomcat conf directory. |
| `Tomcat.Certificate.PublicKeyFilename` | no |  |  | The optional path of the public key file generated when configuring the APR implementation. If left blank, a new file will be created in the Tomcat conf directory. |
| `Java.Certificate.KeystoreFilename` | no |  |  | The optional path of the keystore generated when configuring the BIO, NIO, and NIO2 implementations. If left blank, a keystore file will be created in the Tomcat conf directory. If the file exists,... |
| `Java.Certificate.KeystoreAlias` | no |  |  | The optional alias under which the certificate information will be stored when configuring the BIO, NIO, and NIO2 implementations. If left blank, the default alias of 'octopus' will be used. |

#### Deploy Certificate to WildFly or EAP (`Octopus.WildFlyCertificateDeploy`)
Configure a certificate in WildFly 10+ or JBoss EAP 6+ standalone or domain server.
Categories: Java, Certificates

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `WildFly.Deploy.Controller` | yes | `localhost` |  | The hostname or IP address of the application server that the certificate will be configured in. Example: `localhost` |
| `WildFly.Deploy.Port` | no | `9990` |  | The management port that the application server is listening to. Example: `9990` |
| `WildFly.Deploy.Protocol` | no | `remote+http` |  | The management protocol that the application server is listening to. Example: `remote+http` |
| `WildFly.Deploy.User` | no |  |  | The username to use with the management interface. By default WildFly does not require credentials for local connections. |
| `WildFly.Deploy.Password` | no |  |  | The password to use with the management interface. By default WildFly does not require credentials for local connections. (sensitive) |
| `WildFly.Deploy.ServerType` | no | `Standalone` | `Standalone`, `Domain` | Select the kind of server you are deploying to. |
| `WildFly.Deploy.DeployCertificate` | no | `True` | `True`, `False` | Whether to create a new keystore or reference an existing keystore. Only applicable for standalone servers. |
| `Java.Certificate.Variable` | no |  |  | The certificate variable to deploy. Used when creating a new keystore. |
| `Java.Certificate.KeystoreFilename` | no |  |  | The path of the keystore file. When creating a new keystore, this must be an absolute path. When referencing an existing keystore, this can be relative to a base path. If left blank when creating, ... |
| `Java.Certificate.Password` | no |  |  | An optional password assigned to the private key. If left blank, the default password of 'changeit' will be used. (sensitive) |
| `Java.Certificate.KeystoreAlias` | no |  |  | The optional alias to assign the private key to. If left blank, the default alias of 'Octopus' will be used. |
| `WildFly.Deploy.CertificateRelativeTo` | no | `NONE` | `NONE`, `jboss.home.dir`, `user.home`, `user.dir`, `java.home`, `jboss.server.base.dir`, `jboss.server.config.dir`, `jboss.server.data.dir`, `jboss.server.log.dir`, `jboss.server.temp.dir`, `jboss.controller.temp.dir`, `jboss.domain.base.dir`, `jboss.domain.config.dir`, `jboss.domain.data.dir`, `jboss.domain.log.dir`, `jboss.domain.temp.dir`, `jboss.domain.deployment.dir`, `jboss.domain.servers.dir` | The relative path that the keystore file will be resolved from. Select 'NONE' if the keystore filename is an absolute path. |
| `WildFly.Deploy.CertificateProfiles` | no | `default` |  | A comma-separated list that defines the domain server groups where the certificate will be configured. Only applicable for domain servers. |
| `WildFly.Deploy.HTTPSPortBindingName` | no |  |  | The port binding used to listen to HTTPS requests in the application server. If left blank, defaults to 'https'. |
| `WildFly.Deploy.SecurityRealmName` | no |  |  | The name of the security realm that will be created in applicable application servers. If left blank, defaults to 'OctopusHttps'. |
| `WildFly.Deploy.ElytronKeystoreName` | no |  |  | The name of the Elytron key store. If left blank, defaults to 'OctopusHttpsKS'. |
| `WildFly.Deploy.ElytronKeymanagerName` | no |  |  | The name of the Elytron key manager. If left blank, defaults to 'OctopusHttpsKM'. |
| `WildFly.Deploy.ElytronSSLContextName` | no |  |  | The name of the Elytron server SSL context. If left blank, defaults to 'OctopusHttpsSSC'. |

#### Deploy Java Archive (`Octopus.JavaArchive`)
Deploy a Java archive (.jar, .ear, .war) to one or more machines.
Categories: Java

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.JavaArchive.DeployExploded` | no | `False` | `True`, `False` | If selected, the package will be deployed extracted rather than re-created as an archive. |
| `Octopus.Action.Package.JavaArchiveCompression` | no | `True` | `True`, `False` | Whether the jar command will compress the items in the repackaged archive. Disabling compression is useful for some packages, like Spring Boot archives, that expect library files to be uncompressed. |
| `Octopus.Action.Package.CustomPackageFileName` | no |  |  | The optional filename of the copied package. If left blank, the original filename from the feed will be retained. |
| `Octopus.Action.Package.UseCustomInstallationDirectory` | no | `False` | `True`, `False` | By default the package will be deployed to the target's application directory. This option allows setting a custom deployment directory. |
| `Octopus.Action.Package.CustomInstallationDirectory` | no |  |  | The installed package will be copied to this location on the remote machine. |
| `Octopus.Action.Package.CustomInstallationDirectoryShouldBePurgedBeforeDeployment` | no | `False` | `True`, `False` | Before the installed package is copied, all files in the custom installation directory will be removed. |
| `Octopus.Action.Package.CustomInstallationDirectoryPurgeExclusions` | no |  |  | A newline-separated list of file or directory names, relative to the installation directory, to leave when it is purged. Extended wildcard syntax is supported. |

#### Deploy Java Keystore Certificate (`Octopus.JavaDeployCertificate`)
Deploy a certificate to a Java keystore on the target filesystem.
Categories: Java, Certificates

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Java.Certificate.Variable` | yes |  |  | The certificate variable to deploy. |
| `Java.Certificate.KeystoreFilename` | yes |  |  | The path of the keystore file to create. This must be an absolute path, the parent directory must exist, and the path must include the keystore filename. If the file exists, it will be overwritten. Example: `/opt/server/conf/keys.store` |
| `Java.Certificate.Password` | no |  |  | An optional password assigned to the private key. If left blank, the default password of 'changeit' will be used. (sensitive) |
| `Java.Certificate.KeystoreAlias` | no |  |  | The optional alias to assign the private key to. If left blank, the default alias of 'Octopus' will be used. |

#### Deploy to Tomcat via Manager (`Octopus.TomcatDeploy`)
Deploy a Java application to Tomcat 7+ via the Tomcat Manager.
Categories: Java, Application Servers

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Tomcat.Deploy.Controller` | yes | `http://localhost:8080/manager` |  | The URL of the Tomcat Manager instance that the package will be uploaded to. This URL is relative to the host that is executing the step. Example: `http://localhost:8080/manager` |
| `Tomcat.Deploy.User` | yes |  |  | The username to use with the management interface. This user must be assigned to the manager-script group in the tomcat-users.xml file. |
| `Tomcat.Deploy.Password` | yes |  |  | The password to use with the management interface. (sensitive) |
| `Tomcat.Deploy.Name` | no |  |  | The context path of the deployed artifact. For example, setting this field to 'myapp' will result in the deployment having the context path /myapp in Tomcat. Set the value to / to deploy to the roo... Example: `myapp` |
| `Tomcat.Deploy.Version` | no |  |  | The Tomcat version used with parallel deployments. Leave blank to leave the Tomcat version undefined, which will overwrite any existing deployment with the same context path. |
| `Tomcat.Deploy.Enabled` | no | `True` | `True`, `False` | Whether to leave the application running or stop the application after deployment. |

#### Deploy to WildFly or EAP (`Octopus.WildFlyDeploy`)
Deploy a Java application to WildFly 10+ or Red Hat JBoss EAP 6+ via the management CLI.
Categories: Java, Application Servers

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `WildFly.Deploy.Controller` | yes | `localhost` |  | The hostname or IP address of the application server that the package will be uploaded to. This value is relative to the host that is executing the step. Example: `localhost` |
| `WildFly.Deploy.Port` | yes | `9990` |  | The management port that the application server is listening to. For WildFly 10+ and Red Hat JBoss EAP 7+, the default is 9990. For Red Hat JBoss EAP 6, the default is 9999. Example: `9990` |
| `WildFly.Deploy.Protocol` | yes | `remote+http` |  | The management protocol that the application server is listening to. For WildFly 10+ and Red Hat JBoss EAP 7+, the default is remote+http. For Red Hat JBoss EAP 6, the default is remoting. Example: `remote+http` |
| `WildFly.Deploy.User` | no |  |  | The username to use with the management interface. Typically configured via the add-user script. By default WildFly does not require credentials for local connections. |
| `WildFly.Deploy.Password` | no |  |  | The password to use with the management interface. By default WildFly does not require credentials for local connections. (sensitive) |
| `WildFly.Deploy.Name` | no |  |  | An optional field to override the name of the deployed artifact. For example, setting this field to 'myapp.war' will result in the deployment having that name in WildFly regardless of the package n... Example: `myapp.war` |
| `WildFly.Deploy.ServerType` | no | `Standalone` | `Standalone`, `Domain` | Select the kind of server you are deploying to. |
| `WildFly.Deploy.Enabled` | no | `True` | `True`, `False` | Whether to deploy the application in an enabled or disabled state to a standalone server. This value has no effect when deploying to domain servers. |
| `WildFly.Deploy.EnabledServerGroup` | no |  |  | A comma-separated list that defines the domain server groups where the deployment will be assigned and enabled. This value is ignored when deploying to a standalone instance. |
| `WildFly.Deploy.DisabledServerGroup` | no |  |  | A comma-separated list that defines the domain server groups where the deployment will be assigned and disabled. This value is ignored when deploying to a standalone instance. |

#### Enable or Disable a WildFly/EAP Application (`Octopus.WildFlyState`)
Enable or disable an application in WildFly 10+ or Red Hat JBoss EAP 6+.
Categories: Java, Application Servers

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `WildFly.Deploy.Controller` | yes | `localhost` |  | The hostname or IP address of the application server hosting the application to be enabled or disabled. This value is relative to the host that is executing the step. Example: `localhost` |
| `WildFly.Deploy.Port` | yes | `9990` |  | The management port that the application server is listening to. For WildFly 10+ and Red Hat JBoss EAP 7+, the default is 9990. For Red Hat JBoss EAP 6, the default is 9999. Example: `9990` |
| `WildFly.Deploy.Protocol` | yes | `remote+http` |  | The management protocol that the application server is listening to. For WildFly 10+ and Red Hat JBoss EAP 7+, the default is remote+http. For Red Hat JBoss EAP 6, the default is remoting. Example: `remote+http` |
| `WildFly.Deploy.User` | no |  |  | The username to use with the management interface. Typically configured via the add-user script. By default WildFly does not require credentials for local connections. |
| `WildFly.Deploy.Password` | no |  |  | The password to use with the management interface. By default WildFly does not require credentials for local connections. (sensitive) |
| `WildFly.Deploy.Name` | yes |  |  | The name of the deployment to enable or disable. |
| `WildFly.Deploy.ServerType` | no | `Standalone` | `Standalone`, `Domain` | Select the kind of server you are deploying to. |
| `WildFly.Deploy.Enabled` | no | `True` | `True`, `False` | Whether to enable or disable the application on a standalone server. This value has no effect when deploying to domain servers. |
| `WildFly.Deploy.EnabledServerGroup` | no |  |  | A comma-separated list that defines the domain server groups where the deployment will be enabled. This value is ignored when deploying to a standalone instance. |
| `WildFly.Deploy.DisabledServerGroup` | no |  |  | A comma-separated list that defines the domain server groups where the deployment will be disabled. This value is ignored when deploying to a standalone instance. |

#### Start/Stop a Tomcat Application (`Octopus.TomcatState`)
Change the state of an application in Tomcat 7+.
Categories: Java, Application Servers

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Tomcat.Deploy.Controller` | yes | `http://localhost:8080/manager` |  | The URL of the Tomcat Manager instance. This URL is relative to the host that is executing the step. Example: `http://localhost:8080/manager` |
| `Tomcat.Deploy.User` | yes |  |  | The username to use with the management interface. This user must be assigned to the manager-script group in the tomcat-users.xml file. |
| `Tomcat.Deploy.Password` | yes |  |  | The password to use with the management interface. (sensitive) |
| `Tomcat.Deploy.Name` | yes |  |  | The context path of the deployed artifact. For example, setting this field to 'myapp' will result in the context path /myapp in Tomcat. Set the value to / for the root context. Example: `myapp` |
| `Tomcat.Deploy.Version` | no |  |  | The Tomcat version used with parallel deployments. Leave blank to leave the Tomcat version undefined. |
| `Tomcat.Deploy.Enabled` | no | `True` | `True`, `False` | Whether to leave the application running or stop the application. |

### Kubernetes

#### Deploy ConfigMap (`Octopus.KubernetesDeployConfigMap`)
Deploy a ConfigMap resource to Kubernetes.
Categories: BuiltInStep, Kubernetes

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.KubernetesContainers.ConfigMapName` | yes |  |  | The name of the ConfigMap resource. Example: `my-config-map` |
| `Octopus.Action.KubernetesContainers.ConfigMapValues` | no |  |  | The ConfigMap resource values. Provided as a JSON object of key-value pairs. Example: `{"key1": "value1"}` |
| `Octopus.Action.KubernetesContainers.DeploymentLabels` | no |  |  | The labels applied to the ConfigMap resource. Provided as a JSON object of key-value pairs. |
| `Octopus.Action.KubernetesContainers.Namespace` | no |  |  | The Kubernetes namespace for the ConfigMap. |
| `Octopus.Action.Kubernetes.ResourceStatusCheck` | no | `True` | `True`, `False` | Whether to verify that the ConfigMap reached its desired state after deployment. |
| `Octopus.Action.Kubernetes.ServerSideApply.Enabled` | no | `True` | `True`, `False` | Whether to use server-side apply (--server-side) when calling kubectl apply. |
| `Octopus.Action.Kubernetes.ServerSideApply.ForceConflicts` | no | `True` | `True`, `False` | Whether to forcefully overwrite conflicts (--force-conflicts) when using server-side apply. |

#### Deploy Ingress (`Octopus.KubernetesDeployIngress`)
Deploy an ingress resource to Kubernetes.
Categories: BuiltInStep, Kubernetes

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.KubernetesContainers.IngressName` | yes |  |  | The name of the Kubernetes ingress resource. Example: `my-ingress` |
| `Octopus.Action.KubernetesContainers.IngressClassName` | no |  |  | The name of the ingress class. Example: `nginx` |
| `Octopus.Action.KubernetesContainers.IngressAnnotations` | no |  |  | Annotations applied to the ingress resource. Provided as a JSON array of key-value pairs. |
| `Octopus.Action.KubernetesContainers.IngressRules` | no |  |  | The ingress rules defining host and path routing. Provided as a JSON array. |
| `Octopus.Action.KubernetesContainers.IngressTlsCertificates` | no |  |  | The ingress TLS certificate/host mappings. Provided as a JSON array. |
| `Octopus.Action.KubernetesContainers.DefaultRulePort` | no |  |  | The port to send all unmatched traffic to when no other ingress rule matches. |
| `Octopus.Action.KubernetesContainers.DefaultRuleServiceName` | no |  |  | The service name to send all unmatched traffic to when no other ingress rule matches. |
| `Octopus.Action.KubernetesContainers.DeploymentLabels` | no |  |  | Labels applied to the ingress resource. Labels are optional name/value pairs. Provided as a JSON object. |
| `Octopus.Action.KubernetesContainers.Namespace` | no |  |  | The Kubernetes namespace for the ingress. |
| `Octopus.Action.Kubernetes.ResourceStatusCheck` | no | `True` | `True`, `False` | Whether to verify that the ingress reached its desired state after deployment. |
| `Octopus.Action.Kubernetes.DeploymentTimeout` | no | `180` |  | The maximum time in seconds the step is allowed to run before being terminated. |
| `Octopus.Action.Kubernetes.ServerSideApply.Enabled` | no | `True` | `True`, `False` | Whether to use server-side apply (--server-side) when calling kubectl apply. |
| `Octopus.Action.Kubernetes.ServerSideApply.ForceConflicts` | no | `True` | `True`, `False` | Whether to forcefully overwrite conflicts (--force-conflicts) when using server-side apply. |

#### Deploy Kubernetes Containers (`Octopus.KubernetesDeployContainers`)
Deploys Kubernetes resources such as Deployments, StatefulSets, DaemonSets, and Jobs with container images, services, ingresses, config maps, and secrets.
Categories: BuiltInStep, Kubernetes, Featured

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.KubernetesContainers.DeploymentResourceType` | yes | `Deployment` | `Deployment`, `StatefulSet`, `DaemonSet`, `Job` | Select whether to deploy a Deployment, StatefulSet, DaemonSet, or Job. |
| `Octopus.Action.KubernetesContainers.DeploymentName` | yes |  |  | The name of the deployment resource. Must be unique, and is used by Kubernetes when updating an existing resource. Example: `my-deployment` |
| `Octopus.Action.KubernetesContainers.Replicas` | no | `1` |  | The number of pod replicas to create from this deployment. Example: `3` |
| `Octopus.Action.KubernetesContainers.RevisionHistoryLimit` | no |  |  | The number of revisions to retain. Example: `10` |
| `Octopus.Action.KubernetesContainers.ProgressDeadlineSeconds` | no | `600` |  | The maximum time in seconds for a deployment to make progress before it is considered to be failed. Example: `600` |
| `Octopus.Action.KubernetesContainers.TerminationGracePeriodSeconds` | no | `30` |  | The time to wait for a pod to be terminated. |
| `Octopus.Action.KubernetesContainers.DeploymentLabels` | no |  |  | Labels applied to the deployment resource, pods managed by the deployment resource, the service, and the ingress. Provided as a JSON object of key-value pairs. Example: `{"app": "my-app"}` |
| `Octopus.Action.KubernetesContainers.DeploymentStyle` | no | `RollingUpdate` | `Recreate`, `RollingUpdate`, `BlueGreen` | How the deployment will be updated. |
| `Octopus.Action.KubernetesContainers.MaxUnavailable` | no |  |  | When a RollingUpdate is used, this value indicates how many pods can be offline during an update. |
| `Octopus.Action.KubernetesContainers.MaxSurge` | no |  |  | When a RollingUpdate is used, this value indicates how many additional pods can be online during an update. |
| `Octopus.Action.KubernetesContainers.Containers` | yes |  |  | The details of the containers, including ports, environment variables, volume mounts, resource limits, probes, and lifecycle hooks. Provided as a JSON array. |
| `Octopus.Action.KubernetesContainers.CombinedVolumes` | no |  |  | The details of the volumes available to the containers, including ConfigMap, Secret, EmptyDir, HostPath, PersistentVolumeClaim, and raw YAML volumes. Provided as a JSON array. |
| `Octopus.Action.KubernetesContainers.PodAffinity` | no |  |  | Pod affinity rules for scheduling. Provided as a JSON array. |
| `Octopus.Action.KubernetesContainers.PodAntiAffinity` | no |  |  | Pod anti-affinity rules for scheduling. Provided as a JSON array. |
| `Octopus.Action.KubernetesContainers.NodeAffinity` | no |  |  | Node affinity rules for pod placement. Provided as a JSON array. |
| `Octopus.Action.KubernetesContainers.Tolerations` | no |  |  | Specifies which taints the pod can tolerate. Provided as a JSON array. |
| `Octopus.Action.KubernetesContainers.NodeSelectors` | no |  |  | Node selectors for pod placement, as a JSON object of key-value pairs. |
| `Octopus.Action.KubernetesContainers.DeploymentAnnotations` | no |  |  | Annotations applied to the deployment resource. Provided as a JSON object of key-value pairs. |
| `Octopus.Action.KubernetesContainers.PodAnnotations` | no |  |  | Annotations applied to the pod resource created by the deployment. Provided as a JSON object of key-value pairs. |
| `Octopus.Action.KubernetesContainers.Namespace` | no |  |  | The Kubernetes namespace for the deployment. |
| `Octopus.Action.KubernetesContainers.DeploymentPaused` | no | `False` | `True`, `False` | Whether the deployment is paused. |
| `Octopus.Action.KubernetesContainers.PodReadinessGates` | no |  |  | Readiness gates assigned to a pod. Provided as a JSON array of condition types. |
| `Octopus.Action.KubernetesContainers.PodServiceAccountName` | no |  |  | The pod service account name. |
| `Octopus.Action.KubernetesContainers.PriorityClassName` | no |  |  | The priority class name of the pod. |
| `Octopus.Action.KubernetesContainers.HostAliases` | no |  |  | Host aliases for the pod. Provided as a JSON array of IP and hostname mappings. |
| `Octopus.Action.KubernetesContainers.RestartPolicy` | no | `Always` | `Always`, `OnFailure`, `Never` | The restart policy applied to all containers in the Pod. |
| `Octopus.Action.KubernetesContainers.DnsPolicy` | no |  | `Default`, `ClusterFirst`, `ClusterFirstWithHostNet`, `None` | The DNS policy for the pods in this deployment. |
| `Octopus.Action.KubernetesContainers.DnsConfigNameservers` | no |  |  | DNS policy config nameservers for the pod. |
| `Octopus.Action.KubernetesContainers.DnsConfigSearches` | no |  |  | DNS policy config searches for the pod. |
| `Octopus.Action.KubernetesContainers.DnsConfigOptions` | no |  |  | DNS policy config options for the pod. |
| `Octopus.Action.KubernetesContainers.HostNetwork` | no | `False` | `True`, `False` | Whether the pod uses host networking. |
| `Octopus.Action.KubernetesContainers.PodSecurityFsGroup` | no |  |  | The Pod Security fsGroup field. |
| `Octopus.Action.KubernetesContainers.PodSecurityRunAsGroup` | no |  |  | The Pod Security runAsGroup field. |
| `Octopus.Action.KubernetesContainers.PodSecurityRunAsUser` | no |  |  | The Pod Security runAsUser field. |
| `Octopus.Action.KubernetesContainers.PodSecurityRunAsNonRoot` | no |  | `True`, `False` | The Pod Security runAsNonRoot field. |
| `Octopus.Action.KubernetesContainers.PodSecuritySeLinuxLevel` | no |  |  | The Pod Security SELinux level field. |
| `Octopus.Action.KubernetesContainers.PodSecuritySeLinuxRole` | no |  |  | The Pod Security SELinux role field. |
| `Octopus.Action.KubernetesContainers.PodSecuritySeLinuxType` | no |  |  | The Pod Security SELinux type field. |
| `Octopus.Action.KubernetesContainers.PodSecuritySeLinuxUser` | no |  |  | The Pod Security SELinux user field. |
| `Octopus.Action.KubernetesContainers.PodSecuritySysctls` | no |  |  | The Pod Security sysctls list. |
| `Octopus.Action.KubernetesContainers.PodSecuritySupplementalGroups` | no |  |  | The Pod Security supplemental groups as a comma-separated list. |
| `Octopus.Action.KubernetesContainers.PodManagementPolicy` | no | `OrderedReady` | `OrderedReady`, `Parallel` | The StatefulSet pod management policy. |
| `Octopus.Action.KubernetesContainers.ServiceNameType` | no |  | `None`, `External`, `Managed` | The type of service that the StatefulSet is linked to. Can be None, External, or Managed. |
| `Octopus.Action.KubernetesContainers.StatefulSetServiceName` | no |  |  | The name of the service that the StatefulSet is linked to. |
| `Octopus.Action.KubernetesContainers.PersistentVolumeClaims` | no |  |  | Persistent volume claims for a StatefulSet. Provided as a JSON array. |
| `Octopus.Action.KubernetesContainers.Partition` | no |  |  | The deployment partition for a StatefulSet. |
| `Octopus.Action.KubernetesContainers.MinReadySeconds` | no |  |  | The minimum number of seconds for which a newly created pod should be ready. |
| `Octopus.Action.KubernetesContainers.Completions` | no |  |  | The number of pods to create for a Kubernetes Job. |
| `Octopus.Action.KubernetesContainers.Parallelism` | no |  |  | The number of pods that can run at the same time for a Kubernetes Job. |
| `Octopus.Action.KubernetesContainers.BackoffLimit` | no |  |  | The maximum number of pods that can be created for a Kubernetes Job when the job fails. |
| `Octopus.Action.KubernetesContainers.ActiveDeadlineSeconds` | no |  |  | The number of seconds a Kubernetes Job should run. |
| `Octopus.Action.KubernetesContainers.TtlSecondsAfterFinished` | no |  |  | The number of seconds after completion before a Kubernetes Job is cleaned up. |
| `Octopus.Action.KubernetesContainers.ServiceName` | no |  |  | The name of the Kubernetes service resource. |
| `Octopus.Action.KubernetesContainers.ServiceType` | no | `ClusterIP` | `ClusterIP`, `NodePort`, `LoadBalancer`, `ExternalName` | The type of the Kubernetes service. |
| `Octopus.Action.KubernetesContainers.ServicePorts` | no |  |  | The details of the service ports. Provided as a JSON array. |
| `Octopus.Action.KubernetesContainers.ServiceClusterIp` | no |  |  | The cluster IP address of the service. |
| `Octopus.Action.KubernetesContainers.ServiceLoadBalancerIp` | no |  |  | The load balancer IP address of the service. |
| `Octopus.Action.KubernetesContainers.LoadBalancerAnnotations` | no |  |  | Annotations associated with the Kubernetes service. Provided as a JSON object of key-value pairs. |
| `Octopus.Action.KubernetesContainers.IngressName` | no |  |  | The name of the Kubernetes ingress resource. |
| `Octopus.Action.KubernetesContainers.IngressClassName` | no |  |  | The name of the ingress class. |
| `Octopus.Action.KubernetesContainers.IngressAnnotations` | no |  |  | Annotations applied to the ingress. Provided as a JSON array. |
| `Octopus.Action.KubernetesContainers.IngressRules` | no |  |  | The ingress rules. Provided as a JSON array of host and path configurations. |
| `Octopus.Action.KubernetesContainers.IngressTlsCertificates` | no |  |  | The ingress TLS certificate/host mappings. Provided as a JSON array. |
| `Octopus.Action.KubernetesContainers.ConfigMapName` | no |  |  | The name of the ConfigMap resource. |
| `Octopus.Action.KubernetesContainers.ConfigMapValues` | no |  |  | The ConfigMap resource values. Provided as a JSON object of key-value pairs. |
| `Octopus.Action.KubernetesContainers.SecretName` | no |  |  | The name of the Secret resource. |
| `Octopus.Action.KubernetesContainers.SecretValues` | no |  |  | The Secret resource values. Provided as a JSON object of key-value pairs. (sensitive) |
| `Octopus.Action.KubernetesContainers.CustomResourceYaml` | no |  |  | Raw YAML for additional resources managed during a deployment. |
| `Octopus.Action.KubernetesContainers.CustomResourceYamlFileName` | no |  |  | The name of the file containing the Kubernetes custom resources. |
| `Octopus.Action.KubernetesContainers.DeploymentWait` | no | `NoWait` | `Wait`, `NoWait` | Controls the verification strategy for the deployment. 'Wait' uses the legacy rollout status check. 'NoWait' is used with the modern resource status check or no verification. |
| `Octopus.Action.Kubernetes.ResourceStatusCheck` | no | `True` | `True`, `False` | Whether to verify that Kubernetes objects reached their desired state after deployment. |
| `Octopus.Action.Kubernetes.WaitForJobs` | no | `False` | `True`, `False` | Whether to wait for Kubernetes Jobs to complete during the step. Only applicable when the resource type is Job. |
| `Octopus.Action.Kubernetes.DeploymentTimeout` | no | `180` |  | The maximum time in seconds the step is allowed to run before being terminated. |
| `Octopus.Action.Kubernetes.ServerSideApply.Enabled` | no | `True` | `True`, `False` | Whether to use server-side apply (--server-side) when calling kubectl apply. |
| `Octopus.Action.Kubernetes.ServerSideApply.ForceConflicts` | no | `True` | `True`, `False` | Whether to forcefully overwrite conflicts (--force-conflicts) when using server-side apply. |

#### Deploy Raw YAML (`Octopus.KubernetesDeployRawYaml`)
Deploy raw YAML resources to Kubernetes from inline YAML, a package, or a Git repository.
Categories: BuiltInStep, Kubernetes

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Script.ScriptSource` | yes | `GitRepository` | `Inline`, `Package`, `GitRepository` | The source of the Kubernetes resources. Can be inline YAML, from a package, or from a Git repository. |
| `Octopus.Action.KubernetesContainers.CustomResourceYaml` | no |  |  | The inline YAML content to deploy when the source is set to Inline. |
| `Octopus.Action.KubernetesContainers.CustomResourceYamlFileName` | no |  |  | The path(s) to the YAML file(s) in the package or Git repository. Supports glob patterns and newline-separated paths. Example: `manifests/**/*.yaml` |
| `Octopus.Action.GitRepository.Source` | no |  | `Project`, `External` | The source of the Git repository. Can be the Project repository or an External repository. |
| `Octopus.Action.KubernetesContainers.Namespace` | no |  |  | The Kubernetes namespace for the deployment. |
| `Octopus.Action.Kubernetes.ResourceStatusCheck` | no | `True` | `True`, `False` | Whether to verify that Kubernetes objects reached their desired state after deployment. |
| `Octopus.Action.Kubernetes.DeploymentTimeout` | no | `180` |  | The maximum time in seconds the step is allowed to run before being terminated. |
| `Octopus.Action.Kubernetes.WaitForJobs` | no | `False` | `True`, `False` | Whether to wait for Kubernetes Jobs to complete during the step. |
| `Octopus.Action.Kubernetes.ServerSideApply.Enabled` | no | `True` | `True`, `False` | Whether to use server-side apply (--server-side) when calling kubectl apply. |
| `Octopus.Action.Kubernetes.ServerSideApply.ForceConflicts` | no | `True` | `True`, `False` | Whether to forcefully overwrite conflicts (--force-conflicts) when using server-side apply. |

#### Deploy Secret (`Octopus.KubernetesDeploySecret`)
Deploy a secret resource to Kubernetes.
Categories: BuiltInStep, Kubernetes

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.KubernetesContainers.SecretName` | yes |  |  | The name of the Kubernetes secret resource. Example: `my-secret` |
| `Octopus.Action.KubernetesContainers.SecretValues` | no |  |  | The secret resource values. Provided as a JSON object of key-value pairs. (sensitive) |
| `Octopus.Action.KubernetesContainers.DeploymentLabels` | no |  |  | The labels applied to the secret resource. Provided as a JSON object of key-value pairs. |
| `Octopus.Action.KubernetesContainers.Namespace` | no |  |  | The Kubernetes namespace for the secret. |
| `Octopus.Action.Kubernetes.ResourceStatusCheck` | no | `True` | `True`, `False` | Whether to verify that the secret reached its desired state after deployment. |
| `Octopus.Action.Kubernetes.ServerSideApply.Enabled` | no | `True` | `True`, `False` | Whether to use server-side apply (--server-side) when calling kubectl apply. |
| `Octopus.Action.Kubernetes.ServerSideApply.ForceConflicts` | no | `True` | `True`, `False` | Whether to forcefully overwrite conflicts (--force-conflicts) when using server-side apply. |

#### Deploy Service (`Octopus.KubernetesDeployService`)
Deploy a service resource to Kubernetes.
Categories: BuiltInStep, Kubernetes

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.KubernetesContainers.ServiceName` | yes |  |  | The name of the Kubernetes service resource. Example: `my-service` |
| `Octopus.Action.KubernetesContainers.ServiceType` | yes | `ClusterIP` | `ClusterIP`, `NodePort`, `LoadBalancer`, `ExternalName` | The type of the Kubernetes service. |
| `Octopus.Action.KubernetesContainers.ServicePorts` | no |  |  | The service port mappings including name, port, target port, node port, and protocol. Provided as a JSON array. |
| `Octopus.Action.KubernetesContainers.ServiceClusterIp` | no |  |  | The cluster IP address of the service. |
| `Octopus.Action.KubernetesContainers.ServiceLoadBalancerIp` | no |  |  | The load balancer IP address of the service. |
| `Octopus.Action.KubernetesContainers.LoadBalancerAnnotations` | no |  |  | Annotations associated with the Kubernetes service. Provided as a JSON object of key-value pairs. |
| `Octopus.Action.KubernetesContainers.DeploymentLabels` | no |  |  | Labels applied to the service resource. Labels are optional name/value pairs. Provided as a JSON object. |
| `Octopus.Action.KubernetesContainers.SelectorLabels` | no |  |  | Selector labels used to match the pods that traffic will be sent to. Provided as a JSON object of key-value pairs. |
| `Octopus.Action.KubernetesContainers.Namespace` | no |  |  | The Kubernetes namespace for the service. |
| `Octopus.Action.Kubernetes.ResourceStatusCheck` | no | `True` | `True`, `False` | Whether to verify that the service reached its desired state after deployment. |
| `Octopus.Action.Kubernetes.DeploymentTimeout` | no | `180` |  | The maximum time in seconds the step is allowed to run before being terminated. |
| `Octopus.Action.Kubernetes.ServerSideApply.Enabled` | no | `True` | `True`, `False` | Whether to use server-side apply (--server-side) when calling kubectl apply. |
| `Octopus.Action.Kubernetes.ServerSideApply.ForceConflicts` | no | `True` | `True`, `False` | Whether to forcefully overwrite conflicts (--force-conflicts) when using server-side apply. |

#### Deploy with Kustomize (`Octopus.Kubernetes.Kustomize`)
Deploy Kubernetes resources using Kustomize from a Git repository.
Categories: BuiltInStep, Kubernetes, Kustomize

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.GitRepository.Source` | yes |  | `Project`, `External` | The source of the Git repository. Can be the Project repository or an External repository. |
| `Octopus.Action.Kubernetes.Kustomize.OverlayPath` | yes |  |  | The path to the directory where your kustomization.yaml file is located. Directory paths are case-sensitive. Consider using Octopus variables to dynamically populate overlays for environments and t... Example: `overlays/#{Octopus.Environment.Name}` |
| `Octopus.Action.SubstituteInFiles.TargetFiles` | no |  |  | A newline-separated list of files to perform variable substitution on, relative to the root of the Git repository. Supports glob expression syntax. Example: `**/*.env` |
| `Octopus.Action.Kubernetes.ResourceStatusCheck` | no | `True` | `True`, `False` | Whether to verify that Kubernetes objects reached their desired state after deployment. |
| `Octopus.Action.Kubernetes.DeploymentTimeout` | no | `180` |  | The maximum time in seconds the step is allowed to run before being terminated. |
| `Octopus.Action.Kubernetes.WaitForJobs` | no | `False` | `True`, `False` | Whether to wait for Kubernetes Jobs to complete during the step. |
| `Octopus.Action.Kubernetes.ServerSideApply.Enabled` | no | `True` | `True`, `False` | Whether to use server-side apply (--server-side) when calling kubectl apply. |
| `Octopus.Action.Kubernetes.ServerSideApply.ForceConflicts` | no | `True` | `True`, `False` | Whether to forcefully overwrite conflicts (--force-conflicts) when using server-side apply. |

#### Run Script in Kubernetes (`Octopus.KubernetesRunScript`)
Run a script within an environment configured using the Kubernetes target.
Categories: BuiltInStep, Kubernetes, Script

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Script.ScriptSource` | yes | `Inline` | `Inline`, `Package`, `GitRepository` | The source of the script to run. Can be inline, from a package, or from a Git repository. |
| `Octopus.Action.Script.Syntax` | no |  | `PowerShell`, `Bash` | The syntax of the script being run. Required when the script source is Inline. |
| `Octopus.Action.Script.ScriptBody` | no |  |  | The inline script body to execute. |
| `Octopus.Action.Script.ScriptFileName` | no |  |  | The path to the script file to execute when the source is a package or Git repository. |
| `Octopus.Action.Script.ScriptParameters` | no |  |  | Parameters to pass to the script file. |
| `Octopus.Action.KubernetesContainers.Namespace` | no |  |  | The Kubernetes namespace for the script execution. |

#### Upgrade a Helm Chart (`Octopus.HelmChartUpgrade`)
Upgrade a Helm chart from a package or Git repository, configuring release name, namespace, template values, and Helm options.
Categories: BuiltInStep, Kubernetes, Helm

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Script.ScriptSource` | yes | `Package` | `Package`, `GitRepository` | The source of the Helm chart. Charts can be contained in a package or a Git repository. |
| `Octopus.Action.Helm.ChartDirectory` | no |  |  | Optionally specify the directory inside the package or Git repository where the chart is located. |
| `Octopus.Action.Helm.ReleaseName` | no |  |  | The Helm release name. If a release by this name doesn't already exist, an install will be run. Must consist of only lower case alphanumeric and dash characters. Example: `#{Octopus.Action.Name \| ToLower}-#{Octopus.Environment.Name \| ToLower}` |
| `Octopus.Action.Helm.Namespace` | no |  |  | The namespace the chart will be installed into. Overrides the target namespace and is passed as the --namespace option to the Helm client. |
| `Octopus.Action.Helm.TemplateValuesSources` | no |  |  | The sources for Helm template values. Provided as a JSON array describing chart values, inline YAML, key-value pairs, Git repository, or package-based values files. |
| `Octopus.Action.Helm.ResetValues` | no | `True` | `True`, `False` | When upgrading, reset the values to the ones built into the chart. Maps to --reset-values. |
| `Octopus.Action.Helm.Timeout` | no |  |  | The --timeout duration for Helm. Helm uses a default timeout of 300 seconds. When specifying the timeout, Helm expects a duration like 300s. Example: `300s` |
| `Octopus.Action.Helm.AdditionalArgs` | no |  |  | Additional parameters that will be passed to the helm upgrade command. |
| `Octopus.Action.Helm.CustomHelmExecutable` | no |  |  | The path to a custom Helm executable on the worker or target. |
| `Octopus.Action.Kubernetes.ResourceStatusCheck` | no | `True` | `True`, `False` | Whether to verify that Kubernetes objects reached their desired state. Maps to --wait. |
| `Octopus.Action.Kubernetes.WaitForJobs` | no | `False` | `True`, `False` | Whether to wait for Kubernetes Jobs to complete during the step. Maps to --wait-for-jobs. |

### Other

#### Deploy a Release (`Octopus.DeployRelease`)
Deploys a release of an Octopus project.
Categories: BuiltInStep, Other, Featured

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.DeployRelease.ProjectId` | yes |  |  | Select a project that will be deployed. |
| `Octopus.Action.DeployRelease.ReleaseVersion` | no |  |  | The version of the release to deploy. If not specified, the release version will be determined at deployment time. |
| `Octopus.Action.DeployRelease.DeploymentCondition` | no | `Always` | `Always`, `IfNotCurrentVersion`, `IfNewer` | Control when this deployment should run. |
| `Octopus.Action.DeployRelease.Variables` | no |  |  | Pass variables through to the child deployment. Specified as key=value pairs. |

#### Health Check (`Octopus.HealthCheck`)
Perform a health check and take action based on the result.
Categories: BuiltInStep, Other

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.HealthCheck.Type` | yes | `FullHealthCheck` | `FullHealthCheck`, `ConnectionTest` | Choose to perform a full health check or a connection-only test. |
| `Octopus.Action.HealthCheck.ErrorHandling` | yes | `TreatExceptionsAsErrors` | `TreatExceptionsAsErrors`, `TreatExceptionsAsWarnings` | Select which action to take on a health check error. Fail the deployment or skip deployment targets that are unavailable. |
| `Octopus.Action.HealthCheck.IncludeMachinesInDeployment` | yes | `DoNotAlterMachines` | `DoNotAlterMachines`, `IncludeCheckedMachines` | Select which action to take on new machines. Ignore any newly available deployment targets or include them in the deployment. |

#### Manual Intervention Required (`Octopus.Manual`)
Pauses deployment for human intervention, often used for approval workflow.
Categories: BuiltInStep, Other, Featured

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Manual.Instructions` | yes |  |  | These instructions will be presented to the user to follow. Example: `Don't break anything :)` |
| `Octopus.Action.Manual.ResponsibleTeamIds` | no |  |  | Select the teams responsible for this manual step. If no teams are specified, all users who have permission to deploy the project will be able to perform the manual step. When one or more teams are... Example: `teams-123,teams-124` |
| `Octopus.Action.Manual.BlockConcurrentDeployments` | no | `False` | `True`, `False` | Should other deployments be blocked while this manual intervention is awaiting action. When set to False, another deployment may begin while this step is awaiting intervention. |

#### Process Template (`Octopus.ProcessTemplate`)
Executes an Octopus process template.

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.ProcessTemplate.Reference.Slug` | yes |  |  | The slug identifier of the process template to execute. |
| `Octopus.Action.ProcessTemplate.Reference.VersionMask` | no |  |  | The version mask used to select which version of the process template to use. If not specified, the latest version is used. |

#### Send an Email (`Octopus.Email`)
Sends an email using a configured SMTP server.
Categories: BuiltInStep, Other

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Email.To` | no |  |  | Email addresses to send the email to. Separate multiple email addresses with ; or ,. At least one To address or Team must be entered. Example: `user@example.com` |
| `Octopus.Action.Email.ToTeamIds` | no |  |  | The teams whose members should receive the email. Example: `teams-123` |
| `Octopus.Action.Email.CC` | no |  |  | Email addresses to CC. Separate multiple email addresses with ; or ,. |
| `Octopus.Action.Email.CCTeamIds` | no |  |  | The teams whose members should be CC'd on the email. |
| `Octopus.Action.Email.Bcc` | no |  |  | Email addresses to BCC. Separate multiple email addresses with ; or ,. |
| `Octopus.Action.Email.BccTeamIds` | no |  |  | The teams whose members should be BCC'd on the email. |
| `Octopus.Action.Email.Subject` | yes |  |  | The subject line of the email. |
| `Octopus.Action.Email.Body` | yes |  |  | The email body as raw text or HTML. Supports Octopus variable substitution syntax including conditional if/unless and iteration with each. |
| `Octopus.Action.Email.IsHtml` | no | `False` | `True`, `False` | Whether the email body is HTML or plain text. |
| `Octopus.Action.Email.Priority` | no | `Normal` | `Low`, `Normal`, `High` | Select the priority of the email. |

### Package

#### Deploy a Package (`Octopus.TentaclePackage`)
Deploy the contents of a package to one or more deployment targets.
Categories: BuiltInStep, Package

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Package.PackageId` | yes |  |  | The ID of the package being deployed. Example: `OctoFx.RateService` |
| `Octopus.Action.Package.FeedId` | yes |  |  | The ID of the package feed from which the package being deployed was pulled. Example: `feeds-123` |
| `Octopus.Action.Package.PackageVersion` | no |  |  | The version of the package being deployed. Example: `1.2.3` |
| `Octopus.Action.Package.DownloadOnTentacle` | no | `False` | `True`, `False` | If true, the package will be downloaded by the Tentacle, rather than pushed by the Octopus Server. |
| `Octopus.Action.Package.CustomInstallationDirectory` | no |  |  | If set, a specific directory to which the package will be copied after extraction. Example: `C:\InetPub\WWWRoot\OctoFx` |
| `Octopus.Action.Package.CustomInstallationDirectoryShouldBePurgedBeforeDeployment` | no | `False` | `True`, `False` | If true, all files in the custom installation directory will be deleted before deployment. |
| `Octopus.Action.Package.SkipIfAlreadyInstalled` | no | `False` | `True`, `False` | If true, and the version of the package being deployed is already present on the machine, its re-deployment will be skipped (use with caution). |
| `Octopus.Action.Package.AutomaticallyRunConfigurationTransformationFiles` | no | `True` | `True`, `False` | Whether to automatically run configuration transformation files. |
| `Octopus.Action.Package.AutomaticallyUpdateAppSettingsAndConnectionStrings` | no | `True` | `True`, `False` | Whether to automatically update appSettings and connectionStrings entries in configuration files. |

#### Deploy a VHD (`Octopus.Vhd`)
Deploy a VHD from a package and add it to a Hyper-V virtual machine.
Categories: BuiltInStep, Package

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Vhd.ApplicationPath` | yes |  |  | The folder within the VHD that contains your application. Octopus will look in this folder when performing substitutions and transforms. Example: `PublishedApps\MyApplication` |
| `Octopus.Action.Vhd.DeployVhdToVm` | no | `False` | `True`, `False` | Attach the VHD to a Hyper-V Virtual Machine. |
| `Octopus.Action.Vhd.VmName` | no |  |  | The name of the Hyper-V Virtual Machine to update. The VM will be shut down, the VHD will be added replacing the current first virtual drive if there is one, then the VM will be restarted. |

#### Deploy a Windows Service (`Octopus.WindowsService`)
Deploy the contents of a package and configure a Windows Service.
Categories: BuiltInStep, Package, WindowsServer

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.WindowsService.CreateOrUpdateService` | no | `True` | `True`, `False` | If enabled, Tentacle will use sc.exe to attempt to create a Windows Service using the settings below. If the service already exists, Tentacle will stop the service, reconfigure it, and start it again. |
| `Octopus.Action.WindowsService.ServiceName` | yes |  |  | The name of the Windows Service to create or configure. |
| `Octopus.Action.WindowsService.DisplayName` | no |  |  | Optional display name of the Windows Service. |
| `Octopus.Action.WindowsService.Description` | no |  |  | User-friendly description of the service. |
| `Octopus.Action.WindowsService.ExecutablePath` | yes |  |  | Path to the executable file for the service. The path should be relative to the package installation directory. Example: `MyService.exe` |
| `Octopus.Action.WindowsService.Arguments` | no |  |  | Command line arguments that will be passed to the service when it starts each time. |
| `Octopus.Action.WindowsService.ServiceAccount` | no | `LocalSystem` | `LocalSystem`, `NT Authority\NetworkService`, `NT Authority\LocalService`, `_CUSTOM` | Which built-in account will the service run under. |
| `Octopus.Action.WindowsService.CustomAccountName` | no |  |  | The Windows/domain account of the custom user that the service will run under. You will need to ensure that this user has the 'Log on as a service' right. Example: `YOURDOMAIN\YourAccount` |
| `Octopus.Action.WindowsService.CustomAccountPassword` | no |  |  | The password for the custom account given above. (sensitive) |
| `Octopus.Action.WindowsService.StartMode` | no | `auto` | `auto`, `delayed-auto`, `demand`, `disabled`, `unchanged` | When will the service start. Use Unchanged to use the current status. |
| `Octopus.Action.WindowsService.DesiredStatus` | no | `Default` | `Started`, `Stopped`, `Unchanged`, `Default` | Defines the state of the service after it has been deployed. Started will start the service. Stopped will leave the service stopped. Unchanged will start the service if it already existed and was r... |
| `Octopus.Action.WindowsService.Dependencies` | no |  |  | Any dependencies that the service has. Separate the names using forward slashes (/). Example: `LanmanWorkstation/TCPIP/MSSQL` |

#### Deploy to IIS (`Octopus.IIS`)
Deploy the contents of a package to a Website, a Web Application or a Virtual Directory.
Categories: BuiltInStep, Package, WindowsServer, Featured

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.IISWebSite.DeploymentType` | yes | `webSite` | `webSite`, `virtualDirectory`, `webApplication` | Configure the type of IIS deployment: IIS Web Site, IIS Virtual Directory, or IIS Web Application. |
| `Octopus.Action.IISWebSite.CreateOrUpdateWebSite` | no | `True` | `True`, `False` | If enabled, Tentacle will use the PowerShell Web Administration module to attempt to create or modify an IIS Web Site and Application Pool. |
| `Octopus.Action.IISWebSite.WebSiteName` | yes |  |  | The display name of the IIS Web Site to create or reconfigure. Example: `MyWebSite` |
| `Octopus.Action.IISWebSite.WebRootType` | no | `packageRoot` | `packageRoot`, `relativeToPackageRoot` | Whether to serve content from the package installation directory or a relative path under it. |
| `Octopus.Action.IISWebSite.WebRoot` | no |  |  | A path relative to the package installation directory from which the IIS Web Site will serve content. |
| `Octopus.Action.IISWebSite.StartWebSite` | no | `True` | `True`, `False` | Whether the deployment step should start the IIS Web Site after a successful deployment or not. |
| `Octopus.Action.IISWebSite.Bindings` | yes |  |  | Bindings define what protocols, ports and host names this site will listen on. Stored as a JSON-encoded collection of binding objects. |
| `Octopus.Action.IISWebSite.ExistingBindings` | no | `Replace` | `Replace`, `Merge` | How to handle existing bindings on the IIS Web Site: Replace removes all existing bindings; Merge combines them with the configured bindings. |
| `Octopus.Action.IISWebSite.EnableAnonymousAuthentication` | no | `False` | `True`, `False` | Whether IIS should allow anonymous authentication. |
| `Octopus.Action.IISWebSite.EnableBasicAuthentication` | no | `False` | `True`, `False` | Whether IIS should allow basic authentication with a 401 challenge. |
| `Octopus.Action.IISWebSite.EnableWindowsAuthentication` | no | `True` | `True`, `False` | Whether IIS should allow integrated Windows authentication with a 401 challenge. |
| `Octopus.Action.IISWebSite.ApplicationPoolName` | yes |  |  | Name of the application pool in IIS to create or reconfigure. |
| `Octopus.Action.IISWebSite.ApplicationPoolFrameworkVersion` | no | `v4.0` | `v4.0`, `v2.0`, `No Managed Code` | The version of the .NET common language runtime that this application pool will use. |
| `Octopus.Action.IISWebSite.ApplicationPoolIdentityType` | no | `ApplicationPoolIdentity` | `ApplicationPoolIdentity`, `LocalService`, `LocalSystem`, `NetworkService`, `SpecificUser` | Which built-in account will the application pool run under. |
| `Octopus.Action.IISWebSite.ApplicationPoolUsername` | no |  |  | The Windows/domain account of the custom user that the application pool will run under. Example: `YOURDOMAIN\YourAccount` |
| `Octopus.Action.IISWebSite.ApplicationPoolPassword` | no |  |  | The password for the custom account given above. (sensitive) |
| `Octopus.Action.IISWebSite.StartApplicationPool` | no | `True` | `True`, `False` | Whether the deployment step should start the IIS Application Pool after a successful deployment or not. |
| `Octopus.Action.IISWebSite.VirtualDirectory.CreateOrUpdate` | no | `False` | `True`, `False` | If enabled, Tentacle will use the PowerShell Web Administration module to attempt to create or modify an IIS Virtual Directory using the settings below. |
| `Octopus.Action.IISWebSite.VirtualDirectory.WebSiteName` | no |  |  | The name of the parent Web Site for this Virtual Directory in IIS. The Web Site should already exist when this step runs. |
| `Octopus.Action.IISWebSite.VirtualDirectory.VirtualPath` | no |  |  | The virtual path in IIS relative to the parent Web Site. All parent directories/applications must already exist. Example: `/application1/directory2/mydirectory` |
| `Octopus.Action.IISWebSite.VirtualDirectory.PhysicalPath` | no |  |  | A path relative to the package installation directory from which the Virtual Directory will serve content. |
| `Octopus.Action.IISWebSite.WebApplication.CreateOrUpdate` | no | `False` | `True`, `False` | If enabled, Tentacle will use the PowerShell Web Administration module to attempt to create or modify an IIS Web Application and Application Pool using the settings below. |
| `Octopus.Action.IISWebSite.WebApplication.WebSiteName` | no |  |  | The name of the parent Web Site for this Web Application in IIS. The Web Site should already exist when this step runs. |
| `Octopus.Action.IISWebSite.WebApplication.VirtualPath` | no |  |  | The virtual path in IIS relative to the parent Web Site. All parent directories/applications must already exist. Example: `/myapplication` |
| `Octopus.Action.IISWebSite.WebApplication.PhysicalPath` | no |  |  | A path relative to the package installation directory from which the Web Application will serve content. |
| `Octopus.Action.WebApplication.WebRootType` | no | `packageRoot` | `packageRoot`, `relativeToPackageRoot` | Whether the Web Application should serve content from the package installation directory or a relative path under it. |
| `Octopus.Action.IISWebSite.WebApplication.ApplicationPoolName` | no |  |  | Name of the application pool in IIS to create or reconfigure for the Web Application. |
| `Octopus.Action.IISWebSite.WebApplication.ApplicationPoolFrameworkVersion` | no | `v4.0` | `v4.0`, `v2.0`, `No Managed Code` | The version of the .NET common language runtime that the Web Application's application pool will use. |
| `Octopus.Action.IISWebSite.WebApplication.ApplicationPoolIdentityType` | no | `ApplicationPoolIdentity` | `ApplicationPoolIdentity`, `LocalService`, `LocalSystem`, `NetworkService`, `SpecificUser` | Which built-in account will the Web Application's application pool run under. |
| `Octopus.Action.IISWebSite.WebApplication.ApplicationPoolUsername` | no |  |  | The Windows/domain account of the custom user that the Web Application's application pool will run under. Example: `YOURDOMAIN\YourAccount` |
| `Octopus.Action.IISWebSite.WebApplication.ApplicationPoolPassword` | no |  |  | The password for the custom account given above. (sensitive) |

#### Deploy to NGINX (`Octopus.Nginx`)
Deploy the contents of a package and configure NGINX.
Categories: BuiltInStep, Package

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Nginx.Server.HostName` | no |  |  | The Host header that this server will listen on. The value can be a full (exact) name, a wildcard, or a regular expression. Leave empty to use any Host header. Example: `www.contoso.com` |
| `Octopus.Action.Nginx.Server.Bindings` | no |  |  | Configure the NGINX bindings. Stored as a JSON-encoded collection of binding objects with protocol, port, IP address, and optional SSL certificate settings. |
| `Octopus.Action.Nginx.Server.Locations` | no |  |  | Configure the virtual server locations. Stored as a JSON-encoded collection of location objects with path, directives, headers, and optional reverse proxy settings. |
| `Octopus.Action.Nginx.Server.ConfigName` | no | `#{Octopus.Action.Package.PackageId}.#{Octopus.Environment.Name}#{if Octopus.Deployment.Tenant.Name}.#{Octopus.Deployment.Tenant.Name}#{/if}` |  | This value defines the base name used for the NGINX configuration files. For example, on a Linux system the configuration files will be /etc/nginx/conf.d/[value].conf.d. If left blank, the package ... |

#### Transfer a Package (`Octopus.TransferPackage`)
Transfer a package to a target via an external or built-in feed.
Categories: BuiltInStep, Package

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Package.PackageId` | yes |  |  | The ID of the package to transfer. Example: `OctoFx.RateService` |
| `Octopus.Action.Package.FeedId` | yes |  |  | The ID of the package feed from which the package will be sourced. Example: `feeds-123` |
| `Octopus.Action.Package.PackageVersion` | no |  |  | The version of the package to transfer. Example: `1.2.3` |
| `Octopus.Action.Package.TransferPath` | yes |  |  | The location that the package should be moved to once it is uploaded to the remote target. Example: `C:\temp\MyDir` |
| `Octopus.Action.Package.DownloadOnTentacle` | no | `False` | `True`, `False` | If true, the package will be downloaded by the Tentacle, rather than pushed by the Octopus Server. |

### Script

#### Commit to Git (`Octopus.CommitToGit`)
Commit packages and files to a Git repository
Categories: BuiltInStep, Script

_No configurable properties._

#### Run a Script (`Octopus.Script`)
Runs a PowerShell, Bash, C#, or Python script.
Categories: BuiltInStep, Script, Featured

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Script.ScriptSource` | yes | `Inline` | `Inline`, `Package`, `GitRepository` | The source of the script to run. Can be inline, from a package, or from a Git repository. |
| `Octopus.Action.Script.Syntax` | no | `PowerShell` | `PowerShell`, `CSharp`, `Bash`, `FSharp`, `Python` | The syntax of the script being run in a script step. Required when the script source is Inline. Example: `PowerShell` |
| `Octopus.Action.Script.ScriptBody` | no |  |  | The inline script body to execute. Example: `Write-Host 'Hello!'` |
| `Octopus.Action.Script.ScriptFileName` | no |  |  | The path to the script file to execute when the source is a package or Git repository. Example: `Scripts/MyScript.sh` |
| `Octopus.Action.Script.ScriptParameters` | no |  |  | Parameters to pass to the script file. |
| `Octopus.Action.RunOnServer` | no | `False` | `True`, `False` | Whether or not the action runs on the Octopus Server. Example: `True` |
| `OctopusUseBundledTooling` | no | `False` | `True`, `False` | Whether to use bundled tooling or tooling available on the worker/target. |

### Terraform

#### Apply a Terraform Template (`Octopus.TerraformApply`)
Apply a Terraform template to create, update, or delete infrastructure resources.
Categories: Terraform

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Terraform.ManagedAccount` | no | `None` | `None`, `AWS` | Enable AWS managed account integration. If an account is selected, those credentials do not need to be included in the Terraform template. |
| `Octopus.Action.AwsAccount.Variable` | no |  |  | The variable that references the AWS account to use for authentication. |
| `Octopus.Action.AwsAccount.UseInstanceRole` | no | `False` | `True`, `False` | Use the EC2 instance role for authentication instead of an AWS account variable. |
| `Octopus.Action.Aws.Region` | no |  |  | The AWS region code for the target region. Example: `us-west-2` |
| `Octopus.Action.Aws.AssumeRole` | no | `False` | `True`, `False` | Whether to assume a different AWS service role for authentication. |
| `Octopus.Action.Aws.AssumedRoleArn` | no |  |  | The ARN of the AWS role to assume. Example: `arn:aws:iam::123456789012:role/RoleName` |
| `Octopus.Action.Aws.AssumedRoleSession` | no |  |  | The name for the assumed role session. |
| `Octopus.Action.Aws.AssumeRoleSessionDurationSeconds` | no |  |  | The duration in seconds for the assumed role session. |
| `Octopus.Action.Aws.AssumeRoleExternalId` | no |  |  | The external ID to use when assuming the role. |
| `Octopus.Action.Terraform.AzureAccount` | no | `False` | `True`, `False` | Enable Azure managed account integration. |
| `Octopus.Action.AzureAccount.Variable` | no |  |  | The variable that references the Azure account to use for authentication. |
| `Octopus.Action.Terraform.GoogleCloudAccount` | no | `False` | `True`, `False` | Enable Google Cloud managed account integration. |
| `Octopus.Action.GoogleCloudAccount.Variable` | no |  |  | The variable that references the Google Cloud account to use for authentication. |
| `Octopus.Action.GoogleCloud.UseVMServiceAccount` | no | `True` | `True`, `False` | Use the VM service account for authentication instead of an account variable. |
| `Octopus.Action.GoogleCloud.ImpersonateServiceAccount` | no | `False` | `True`, `False` | Impersonate a service account. Only works with Terraform google provider version 3.45.0 or above. Sets the GOOGLE_IMPERSONATE_SERVICE_ACCOUNT environment variable. |
| `Octopus.Action.GoogleCloud.ServiceAccountEmails` | no |  |  | The service account emails to impersonate. |
| `Octopus.Action.GoogleCloud.Project` | no |  |  | The default Google Cloud project. Sets the GOOGLE_PROJECT environment variable. |
| `Octopus.Action.GoogleCloud.Region` | no |  |  | The default Google Cloud region. Sets the GOOGLE_REGION environment variable. |
| `Octopus.Action.GoogleCloud.Zone` | no |  |  | The default Google Cloud zone. Sets the GOOGLE_ZONE environment variable. |
| `Octopus.Action.Script.ScriptSource` | yes | `Inline` | `Inline`, `Package`, `GitRepository` | Select the source of the Terraform template. Templates can be entered as source code, contained in a Git repository, or a package. |
| `Octopus.Action.Terraform.Template` | no |  |  | The inline Terraform template source code in HCL or JSON format. |
| `Octopus.Action.Terraform.TemplateParameters` | no |  |  | The JSON-encoded values for the Terraform template variables. |
| `Octopus.Action.Terraform.TemplateDirectory` | no |  |  | The optional directory inside the package or repository that contains the Terraform template source files. |
| `Octopus.Action.Terraform.RunAutomaticFileSubstitution` | no | `True` | `True`, `False` | Replace variables in all *.tf, *.tfvars, *.tf.json, and *.tfvars.json files using the #{Variable} substitution syntax. |
| `Octopus.Action.Terraform.FileSubstitution` | no |  |  | A newline-separated list of file names to substitute variables using the #{Variable} substitution syntax, relative to the package contents. Extended wildcard syntax is supported. Example: `Config\*.json` |
| `Octopus.Action.Terraform.VarFiles` | no |  |  | An optional newline-separated list of files that are passed as -var-file parameters. Files called terraform.tfvars, terraform.tfvars.json, *.auto.tfvars, and *.auto.tfvars.json are automatically lo... |
| `Octopus.Action.Terraform.Workspace` | no |  |  | The Terraform workspace to use. |
| `Octopus.Action.Terraform.PluginsDirectory` | no |  |  | The optional directory that holds the Terraform plugins. This directory will be copied to a temporary workspace for each deployment to avoid downloading the plugins from the Internet. Specify TF_PL... |
| `Octopus.Action.Terraform.AllowPluginDownloads` | no | `True` | `True`, `False` | Allow Terraform to download plugins that are not found in the plugin cache directory. Note: this option was removed in Terraform v0.15.0; starting with v0.15.0 Terraform always installs plugins. |
| `Octopus.Action.Terraform.CustomTerraformExecutable` | no |  |  | The path to a custom Terraform executable. |
| `Octopus.Action.Terraform.AdditionalInitParams` | no |  |  | An optional list of additional parameters to pass to the terraform init command. |
| `Octopus.Action.Terraform.AdditionalActionParams` | no |  |  | An optional list of additional parameters to pass to the terraform apply command. |
| `Octopus.Action.Terraform.AttachLogFile` | no | `False` | `True`, `False` | Whether to attach the Terraform log file as an artifact. |
| `Octopus.Action.Terraform.EnvVariables` | no |  |  | Passes through variables into Terraform CLI accessible as environment variables. Environment variables specified here will override options specified in other sections, with a few exceptions such a... Example: `{"TF_LOG":"DEBUG"}` |
| `Octopus.Action.GitRepository.Source` | no |  | `Project`, `External` | The source of the Git repository when template source is set to Git repository. |

#### Destroy Terraform Resources (`Octopus.TerraformDestroy`)
Destroy infrastructure resources managed by a Terraform template.
Categories: Terraform

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Terraform.ManagedAccount` | no | `None` | `None`, `AWS` | Enable AWS managed account integration. If an account is selected, those credentials do not need to be included in the Terraform template. |
| `Octopus.Action.AwsAccount.Variable` | no |  |  | The variable that references the AWS account to use for authentication. |
| `Octopus.Action.AwsAccount.UseInstanceRole` | no | `False` | `True`, `False` | Use the EC2 instance role for authentication instead of an AWS account variable. |
| `Octopus.Action.Aws.Region` | no |  |  | The AWS region code for the target region. Example: `us-west-2` |
| `Octopus.Action.Aws.AssumeRole` | no | `False` | `True`, `False` | Whether to assume a different AWS service role for authentication. |
| `Octopus.Action.Aws.AssumedRoleArn` | no |  |  | The ARN of the AWS role to assume. Example: `arn:aws:iam::123456789012:role/RoleName` |
| `Octopus.Action.Aws.AssumedRoleSession` | no |  |  | The name for the assumed role session. |
| `Octopus.Action.Aws.AssumeRoleSessionDurationSeconds` | no |  |  | The duration in seconds for the assumed role session. |
| `Octopus.Action.Aws.AssumeRoleExternalId` | no |  |  | The external ID to use when assuming the role. |
| `Octopus.Action.Terraform.AzureAccount` | no | `False` | `True`, `False` | Enable Azure managed account integration. |
| `Octopus.Action.AzureAccount.Variable` | no |  |  | The variable that references the Azure account to use for authentication. |
| `Octopus.Action.Terraform.GoogleCloudAccount` | no | `False` | `True`, `False` | Enable Google Cloud managed account integration. |
| `Octopus.Action.GoogleCloudAccount.Variable` | no |  |  | The variable that references the Google Cloud account to use for authentication. |
| `Octopus.Action.GoogleCloud.UseVMServiceAccount` | no | `True` | `True`, `False` | Use the VM service account for authentication instead of an account variable. |
| `Octopus.Action.GoogleCloud.ImpersonateServiceAccount` | no | `False` | `True`, `False` | Impersonate a service account. Only works with Terraform google provider version 3.45.0 or above. Sets the GOOGLE_IMPERSONATE_SERVICE_ACCOUNT environment variable. |
| `Octopus.Action.GoogleCloud.ServiceAccountEmails` | no |  |  | The service account emails to impersonate. |
| `Octopus.Action.GoogleCloud.Project` | no |  |  | The default Google Cloud project. Sets the GOOGLE_PROJECT environment variable. |
| `Octopus.Action.GoogleCloud.Region` | no |  |  | The default Google Cloud region. Sets the GOOGLE_REGION environment variable. |
| `Octopus.Action.GoogleCloud.Zone` | no |  |  | The default Google Cloud zone. Sets the GOOGLE_ZONE environment variable. |
| `Octopus.Action.Script.ScriptSource` | yes | `Inline` | `Inline`, `Package`, `GitRepository` | Select the source of the Terraform template. Templates can be entered as source code, contained in a Git repository, or a package. |
| `Octopus.Action.Terraform.Template` | no |  |  | The inline Terraform template source code in HCL or JSON format. |
| `Octopus.Action.Terraform.TemplateParameters` | no |  |  | The JSON-encoded values for the Terraform template variables. |
| `Octopus.Action.Terraform.TemplateDirectory` | no |  |  | The optional directory inside the package or repository that contains the Terraform template source files. |
| `Octopus.Action.Terraform.RunAutomaticFileSubstitution` | no | `True` | `True`, `False` | Replace variables in all *.tf, *.tfvars, *.tf.json, and *.tfvars.json files using the #{Variable} substitution syntax. |
| `Octopus.Action.Terraform.FileSubstitution` | no |  |  | A newline-separated list of file names to substitute variables using the #{Variable} substitution syntax, relative to the package contents. Extended wildcard syntax is supported. Example: `Config\*.json` |
| `Octopus.Action.Terraform.VarFiles` | no |  |  | An optional newline-separated list of files that are passed as -var-file parameters. Files called terraform.tfvars, terraform.tfvars.json, *.auto.tfvars, and *.auto.tfvars.json are automatically lo... |
| `Octopus.Action.Terraform.Workspace` | no |  |  | The Terraform workspace to use. |
| `Octopus.Action.Terraform.PluginsDirectory` | no |  |  | The optional directory that holds the Terraform plugins. This directory will be copied to a temporary workspace for each deployment to avoid downloading the plugins from the Internet. Specify TF_PL... |
| `Octopus.Action.Terraform.AllowPluginDownloads` | no | `True` | `True`, `False` | Allow Terraform to download plugins that are not found in the plugin cache directory. Note: this option was removed in Terraform v0.15.0; starting with v0.15.0 Terraform always installs plugins. |
| `Octopus.Action.Terraform.CustomTerraformExecutable` | no |  |  | The path to a custom Terraform executable. |
| `Octopus.Action.Terraform.AdditionalInitParams` | no |  |  | An optional list of additional parameters to pass to the terraform init command. |
| `Octopus.Action.Terraform.AdditionalActionParams` | no |  |  | An optional list of additional parameters to pass to the terraform destroy command. |
| `Octopus.Action.Terraform.AttachLogFile` | no | `False` | `True`, `False` | Whether to attach the Terraform log file as an artifact. |
| `Octopus.Action.Terraform.EnvVariables` | no |  |  | Passes through variables into Terraform CLI accessible as environment variables. Environment variables specified here will override options specified in other sections, with a few exceptions such a... Example: `{"TF_LOG":"DEBUG"}` |
| `Octopus.Action.GitRepository.Source` | no |  | `Project`, `External` | The source of the Git repository when template source is set to Git repository. |

#### Plan Terraform Changes (`Octopus.TerraformPlan`)
Plan the changes required by a Terraform template without applying them, and optionally output the plan in JSON format.
Categories: Terraform

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Terraform.ManagedAccount` | no | `None` | `None`, `AWS` | Enable AWS managed account integration. If an account is selected, those credentials do not need to be included in the Terraform template. |
| `Octopus.Action.AwsAccount.Variable` | no |  |  | The variable that references the AWS account to use for authentication. |
| `Octopus.Action.AwsAccount.UseInstanceRole` | no | `False` | `True`, `False` | Use the EC2 instance role for authentication instead of an AWS account variable. |
| `Octopus.Action.Aws.Region` | no |  |  | The AWS region code for the target region. Example: `us-west-2` |
| `Octopus.Action.Aws.AssumeRole` | no | `False` | `True`, `False` | Whether to assume a different AWS service role for authentication. |
| `Octopus.Action.Aws.AssumedRoleArn` | no |  |  | The ARN of the AWS role to assume. Example: `arn:aws:iam::123456789012:role/RoleName` |
| `Octopus.Action.Aws.AssumedRoleSession` | no |  |  | The name for the assumed role session. |
| `Octopus.Action.Aws.AssumeRoleSessionDurationSeconds` | no |  |  | The duration in seconds for the assumed role session. |
| `Octopus.Action.Aws.AssumeRoleExternalId` | no |  |  | The external ID to use when assuming the role. |
| `Octopus.Action.Terraform.AzureAccount` | no | `False` | `True`, `False` | Enable Azure managed account integration. |
| `Octopus.Action.AzureAccount.Variable` | no |  |  | The variable that references the Azure account to use for authentication. |
| `Octopus.Action.Terraform.GoogleCloudAccount` | no | `False` | `True`, `False` | Enable Google Cloud managed account integration. |
| `Octopus.Action.GoogleCloudAccount.Variable` | no |  |  | The variable that references the Google Cloud account to use for authentication. |
| `Octopus.Action.GoogleCloud.UseVMServiceAccount` | no | `True` | `True`, `False` | Use the VM service account for authentication instead of an account variable. |
| `Octopus.Action.GoogleCloud.ImpersonateServiceAccount` | no | `False` | `True`, `False` | Impersonate a service account. Only works with Terraform google provider version 3.45.0 or above. Sets the GOOGLE_IMPERSONATE_SERVICE_ACCOUNT environment variable. |
| `Octopus.Action.GoogleCloud.ServiceAccountEmails` | no |  |  | The service account emails to impersonate. |
| `Octopus.Action.GoogleCloud.Project` | no |  |  | The default Google Cloud project. Sets the GOOGLE_PROJECT environment variable. |
| `Octopus.Action.GoogleCloud.Region` | no |  |  | The default Google Cloud region. Sets the GOOGLE_REGION environment variable. |
| `Octopus.Action.GoogleCloud.Zone` | no |  |  | The default Google Cloud zone. Sets the GOOGLE_ZONE environment variable. |
| `Octopus.Action.Script.ScriptSource` | yes | `Inline` | `Inline`, `Package`, `GitRepository` | Select the source of the Terraform template. Templates can be entered as source code, contained in a Git repository, or a package. |
| `Octopus.Action.Terraform.Template` | no |  |  | The inline Terraform template source code in HCL or JSON format. |
| `Octopus.Action.Terraform.TemplateParameters` | no |  |  | The JSON-encoded values for the Terraform template variables. |
| `Octopus.Action.Terraform.TemplateDirectory` | no |  |  | The optional directory inside the package or repository that contains the Terraform template source files. |
| `Octopus.Action.Terraform.RunAutomaticFileSubstitution` | no | `True` | `True`, `False` | Replace variables in all *.tf, *.tfvars, *.tf.json, and *.tfvars.json files using the #{Variable} substitution syntax. |
| `Octopus.Action.Terraform.FileSubstitution` | no |  |  | A newline-separated list of file names to substitute variables using the #{Variable} substitution syntax, relative to the package contents. Extended wildcard syntax is supported. Example: `Config\*.json` |
| `Octopus.Action.Terraform.VarFiles` | no |  |  | An optional newline-separated list of files that are passed as -var-file parameters. Files called terraform.tfvars, terraform.tfvars.json, *.auto.tfvars, and *.auto.tfvars.json are automatically lo... |
| `Octopus.Action.Terraform.PlanJsonOutput` | no | `False` | `True`, `False` | Specify the output format for the plan operation. Plain text output captures the written description of the changes. JSON output captures the changes as JSON blobs, which can be inspected and parse... |
| `Octopus.Action.Terraform.Workspace` | no |  |  | The Terraform workspace to use. |
| `Octopus.Action.Terraform.PluginsDirectory` | no |  |  | The optional directory that holds the Terraform plugins. This directory will be copied to a temporary workspace for each deployment to avoid downloading the plugins from the Internet. Specify TF_PL... |
| `Octopus.Action.Terraform.AllowPluginDownloads` | no | `True` | `True`, `False` | Allow Terraform to download plugins that are not found in the plugin cache directory. Note: this option was removed in Terraform v0.15.0; starting with v0.15.0 Terraform always installs plugins. |
| `Octopus.Action.Terraform.CustomTerraformExecutable` | no |  |  | The path to a custom Terraform executable. |
| `Octopus.Action.Terraform.AdditionalInitParams` | no |  |  | An optional list of additional parameters to pass to the terraform init command. |
| `Octopus.Action.Terraform.AdditionalActionParams` | no |  |  | An optional list of additional parameters to pass to the terraform plan command. |
| `Octopus.Action.Terraform.AttachLogFile` | no | `False` | `True`, `False` | Whether to attach the Terraform log file as an artifact. |
| `Octopus.Action.Terraform.EnvVariables` | no |  |  | Passes through variables into Terraform CLI accessible as environment variables. Environment variables specified here will override options specified in other sections, with a few exceptions such a... Example: `{"TF_LOG":"DEBUG"}` |
| `Octopus.Action.GitRepository.Source` | no |  | `Project`, `External` | The source of the Git repository when template source is set to Git repository. |

#### Plan Terraform Destruction (`Octopus.TerraformPlanDestroy`)
Plan the destruction of infrastructure resources managed by a Terraform template without executing it, and optionally output the plan in JSON format.
Categories: Terraform

| Key | Required | Default | Values | Description |
|---|---|---|---|---|
| `Octopus.Action.Terraform.ManagedAccount` | no | `None` | `None`, `AWS` | Enable AWS managed account integration. If an account is selected, those credentials do not need to be included in the Terraform template. |
| `Octopus.Action.AwsAccount.Variable` | no |  |  | The variable that references the AWS account to use for authentication. |
| `Octopus.Action.AwsAccount.UseInstanceRole` | no | `False` | `True`, `False` | Use the EC2 instance role for authentication instead of an AWS account variable. |
| `Octopus.Action.Aws.Region` | no |  |  | The AWS region code for the target region. Example: `us-west-2` |
| `Octopus.Action.Aws.AssumeRole` | no | `False` | `True`, `False` | Whether to assume a different AWS service role for authentication. |
| `Octopus.Action.Aws.AssumedRoleArn` | no |  |  | The ARN of the AWS role to assume. Example: `arn:aws:iam::123456789012:role/RoleName` |
| `Octopus.Action.Aws.AssumedRoleSession` | no |  |  | The name for the assumed role session. |
| `Octopus.Action.Aws.AssumeRoleSessionDurationSeconds` | no |  |  | The duration in seconds for the assumed role session. |
| `Octopus.Action.Aws.AssumeRoleExternalId` | no |  |  | The external ID to use when assuming the role. |
| `Octopus.Action.Terraform.AzureAccount` | no | `False` | `True`, `False` | Enable Azure managed account integration. |
| `Octopus.Action.AzureAccount.Variable` | no |  |  | The variable that references the Azure account to use for authentication. |
| `Octopus.Action.Terraform.GoogleCloudAccount` | no | `False` | `True`, `False` | Enable Google Cloud managed account integration. |
| `Octopus.Action.GoogleCloudAccount.Variable` | no |  |  | The variable that references the Google Cloud account to use for authentication. |
| `Octopus.Action.GoogleCloud.UseVMServiceAccount` | no | `True` | `True`, `False` | Use the VM service account for authentication instead of an account variable. |
| `Octopus.Action.GoogleCloud.ImpersonateServiceAccount` | no | `False` | `True`, `False` | Impersonate a service account. Only works with Terraform google provider version 3.45.0 or above. Sets the GOOGLE_IMPERSONATE_SERVICE_ACCOUNT environment variable. |
| `Octopus.Action.GoogleCloud.ServiceAccountEmails` | no |  |  | The service account emails to impersonate. |
| `Octopus.Action.GoogleCloud.Project` | no |  |  | The default Google Cloud project. Sets the GOOGLE_PROJECT environment variable. |
| `Octopus.Action.GoogleCloud.Region` | no |  |  | The default Google Cloud region. Sets the GOOGLE_REGION environment variable. |
| `Octopus.Action.GoogleCloud.Zone` | no |  |  | The default Google Cloud zone. Sets the GOOGLE_ZONE environment variable. |
| `Octopus.Action.Script.ScriptSource` | yes | `Inline` | `Inline`, `Package`, `GitRepository` | Select the source of the Terraform template. Templates can be entered as source code, contained in a Git repository, or a package. |
| `Octopus.Action.Terraform.Template` | no |  |  | The inline Terraform template source code in HCL or JSON format. |
| `Octopus.Action.Terraform.TemplateParameters` | no |  |  | The JSON-encoded values for the Terraform template variables. |
| `Octopus.Action.Terraform.TemplateDirectory` | no |  |  | The optional directory inside the package or repository that contains the Terraform template source files. |
| `Octopus.Action.Terraform.RunAutomaticFileSubstitution` | no | `True` | `True`, `False` | Replace variables in all *.tf, *.tfvars, *.tf.json, and *.tfvars.json files using the #{Variable} substitution syntax. |
| `Octopus.Action.Terraform.FileSubstitution` | no |  |  | A newline-separated list of file names to substitute variables using the #{Variable} substitution syntax, relative to the package contents. Extended wildcard syntax is supported. Example: `Config\*.json` |
| `Octopus.Action.Terraform.VarFiles` | no |  |  | An optional newline-separated list of files that are passed as -var-file parameters. Files called terraform.tfvars, terraform.tfvars.json, *.auto.tfvars, and *.auto.tfvars.json are automatically lo... |
| `Octopus.Action.Terraform.PlanJsonOutput` | no | `False` | `True`, `False` | Specify the output format for the plan operation. Plain text output captures the written description of the changes. JSON output captures the changes as JSON blobs, which can be inspected and parse... |
| `Octopus.Action.Terraform.Workspace` | no |  |  | The Terraform workspace to use. |
| `Octopus.Action.Terraform.PluginsDirectory` | no |  |  | The optional directory that holds the Terraform plugins. This directory will be copied to a temporary workspace for each deployment to avoid downloading the plugins from the Internet. Specify TF_PL... |
| `Octopus.Action.Terraform.AllowPluginDownloads` | no | `True` | `True`, `False` | Allow Terraform to download plugins that are not found in the plugin cache directory. Note: this option was removed in Terraform v0.15.0; starting with v0.15.0 Terraform always installs plugins. |
| `Octopus.Action.Terraform.CustomTerraformExecutable` | no |  |  | The path to a custom Terraform executable. |
| `Octopus.Action.Terraform.AdditionalInitParams` | no |  |  | An optional list of additional parameters to pass to the terraform init command. |
| `Octopus.Action.Terraform.AdditionalActionParams` | no |  |  | An optional list of additional parameters to pass to the terraform plan -destroy command. |
| `Octopus.Action.Terraform.AttachLogFile` | no | `False` | `True`, `False` | Whether to attach the Terraform log file as an artifact. |
| `Octopus.Action.Terraform.EnvVariables` | no |  |  | Passes through variables into Terraform CLI accessible as environment variables. Environment variables specified here will override options specified in other sections, with a few exceptions such a... Example: `{"TF_LOG":"DEBUG"}` |
| `Octopus.Action.GitRepository.Source` | no |  | `Project`, `External` | The source of the Git repository when template source is set to Git repository. |