# Error recovery

Companion reference for `SKILL.md` (`octopus-onboarding`). Use this when a call to Octopus (via MCP `execute` or direct REST) fails, or when a target / feed / process refuses to behave as expected.

## The universal rule: `PUT` is replace, not patch

Every mutation endpoint in the Octopus API takes the **complete resource body**. If you omit a field that has a required non-zero default on the server side, you may pass validation on create (server fills it) and fail validation on update (your zero collides with a `>= 1` rule).

**Always `GET` first, mutate the fields you want to change, `PUT` the whole object back.** Do not hand-craft `PUT` bodies from scratch.

This applies broadly — feeds, environments, lifecycles, projects, tenants, deployment processes, machines. Any mutation that goes through `PUT`. (`POST` creates and accepts partial bodies; `PUT` does not.)

## Common errors and what they mean

| API error | Likely cause | Action |
|---|---|---|
| `401` / `403` on first call | API key scope or expiry. | `GET /api/users/me` (MCP: `get_current_user`). Direct the user to Profile → My API Keys to mint a fresh key. |
| `409 Conflict` on project / env / tenant / feed create | Name already exists. | `GET` the existing one by partial name (`?partialName=...`) — ask if the user meant to reuse it before forcing a different name. |
| Target stays `Unknown` after agent Helm install | `agentReachableUrl` not reachable from the cluster, wrong bearer token, or the agent pod isn't running. | `kubectl get pods -n <agent-ns>`; `kubectl logs deploy/octopus-agent-tentacle -n <agent-ns>`. Re-run the curl-in-a-pod reachability check from `references/kubernetes-agent-install.md`. |
| Health task fails with `Unable to retrieve authentication token` | Feed credentials missing — including for "public" registries. | Docker feeds always need credentials. See `references/build-output-and-feeds.md` §"registry creds". |
| `PUT` errors on a field you didn't set (e.g., `DownloadAttempts must be between 1 and 5`) | Some fields have hidden defaults that serialize to `0` when omitted; `PUT` is a full replace, so your missing field becomes `0`. | **Round-trip the body.** `GET` the resource, mutate only what you want to change, `PUT` the whole object back. |
| Validation fails on deployment process | Missing variable, bad tag reference, missing package, or property key the action type doesn't accept. | `POST /api/{sp}/projects/{id}/deploymentprocesses/validate` *before* `PUT`. Surface the specific warnings. |
| Feed `packages/search` returns empty | Wrong creds, wrong registry URL, or nothing has been published yet. | Re-check the CI log — the push step may not have run. Confirm the image coordinates Octopus has match what's in the feed. |
| `Missing tenant variables` warning at deploy time | Required template vars not set per tenant. | `GET /api/{sp}/tenants/variables-missing` (MCP: `get_missing_tenant_variables`); then `POST /api/{sp}/tenants/{id}/projectvariables`. |
| Release creation fails with "package version not found" | Feed hasn't indexed yet, or version mismatch between CI and release. | `GET /api/{sp}/feeds/{id}/packages/versions?packageId=...` to list what Octopus actually sees; compare against the CI log. |
| Deployment fails midway with no clear cause | The activity log has the answer, but it's often megabytes long. | Use `grep_task_log` (MCP) or the equivalent `GET /api/{sp}/tasks/{id}/details` filtered client-side. Patterns to grep for: `Error`, `Fatal`, `failed`, `Unable to`, `Exception`. |
| `404` on `POST /api/{sp}/...` immediately after creating the space | New spaces sometimes lag a moment before all their sub-collections exist. | Retry once after a short delay. If still failing, double-check the space ID. |
| Elicitation prompt loops or repeats | Client doesn't actually advertise elicitation capability, but the tool insists on confirmation. | Set the `confirm: true` field in the tool's input schema (the per-tool fallback path), or set `OCTOPUS_SKIP_ELICITATION=true` for automation. See `references/mcp-and-api.md` §"Writes — gated through elicitation". |

## Health-check failure decoding

When a Health task fails (`POST /api/tasks` with `Name: "Health"`), fetch `GET /api/{sp}/tasks/{taskId}/details` and map common errors:

- **Port unreachable** — firewall, security group, or network partition. The user fixes it on the network side; the agent can't.
- **TLS handshake failed / cert thumbprint mismatch** — a Tentacle's cert changed (was reinstalled, certificate rotated). Update `Endpoint.Thumbprint` after `GET`-ing the new thumbprint via `GET /api/{sp}/machines/{id}` once the Tentacle is up.
- **`401 Unauthorized`** on a feed test — credentials wrong or scope insufficient. See `references/build-output-and-feeds.md`.
- **`Unable to retrieve authentication token`** — the Docker feed handshake. Always means credentials, even for public registries.
- **`Connection refused` from a worker** to a cloud API — usually IAM / OIDC trust policy, not network. Check `GET /api/{sp}/accounts/{id}` and the trust policy on the AWS/Azure/GCP side.

## When a call fails through MCP `execute`

The `execute` tool surfaces the upstream HTTP response. Treat it like a raw REST error — same diagnosis flow as above. The MCP layer doesn't reinterpret the response, it just transports it.

If `execute` itself errors before reaching Octopus (e.g., "MCP write tools require `--no-read-only`"), the user's MCP server is in read-only mode by default. They need to launch it with `--no-read-only` for write tools to work — direct them to that flag, don't try to work around it.

## When the spec disagrees with reality

If a property key works in the running Octopus instance but isn't in `octopus://api/llms.txt`, or vice versa: the running instance is authoritative for *behaviour*, the spec is authoritative for *contracts*. Usually this means the spec lags a recent server feature, or the user's instance is older than the spec.

Cross-check with `octopus://api/capabilities` (MCP) — it reports the running version and the feature flags actually enabled. If a tool you expect requires a newer Octopus version (e.g., `get_kubernetes_live_status` needs 2025.3+), the capabilities resource will tell you, and you can fall through to a less-capable but more-portable equivalent.
