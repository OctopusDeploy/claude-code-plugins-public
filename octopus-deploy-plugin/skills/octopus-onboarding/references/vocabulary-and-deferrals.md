# Vocabulary translation and what to defer

Companion reference for `SKILL.md` (`octopus-onboarding`). Use this whenever you're about to mention an Octopus-specific term to the user, or when you find yourself reaching for a feature the user hasn't asked about.

## Vocabulary translation table

Translate before using the Octopus term. Several Octopus words collide with concepts the user already has from other tools — those collisions cause more confusion than time-savings, even with experienced users.

| Octopus term | What it actually means | Lead with this | Notes |
|---|---|---|---|
| **Space** | Tenancy boundary inside an Octopus install. | "workspace" | If the user has only one space, don't mention spaces at all. |
| **Tenant** | Deployment-target dimension — a customer, a region, a variant. | "customer" / "deployment slice" | Frequently confused with Azure AD tenants and SaaS-billing tenants. Always define on first mention. |
| **Channel** | Release track (stable vs. beta vs. hotfix). | "release track" | Collides with Slack channels and "release channels" in other tools. Skip entirely for first-time setup unless the user asks. |
| **Runbook** | Operational automation (restart, rotate, migrate). Not a deployment. | "operational task" | Collides with incident-response runbooks. Never use without context. |
| **Lifecycle** | Ordered promotion path through environments. | "promotion path" / "pipeline stages" | Some users recognize the word; for others it's too abstract. Keep `Lifecycles-X` in API calls; use plain English to the user. |
| **Worker** | Machine that *runs* steps (vs. target that *receives* them). | "runner" | Easy to confuse with a deployment target. |
| **Feed** | Package source — registry, NuGet, Docker Hub, Maven, etc. | "registry" for containers; "feed" for NuGet/Maven | Let the domain drive the wording. |
| **Project** | Deployable unit — service / app / module. | the user's own term: "service" / "app" | Safe, no translation needed. |
| **Environment**, **Release**, **Deployment**, **Deploy**, **Rollback**, **Variable**, **Secret**, **Artifact**, **Pipeline**, **Approval**, **Health check**, **Trigger** | (familiar) | Use freely — these terms mean what they sound like. |

## What to defer / not do

These are the most-cited frustrations from real customer onboarding sessions. Don't do any of them unless the user *explicitly* asks.

- **Don't create a "Hello World" sample project.** Users want to see their real work running, not a demo.
- **Don't run a guided product tour.** Same reason. The interactive walkthrough is for someone who's never seen Octopus and wants to browse — almost no one in an onboarding flow.
- **Don't auto-generate or modify the user's CI workflow file.** If a CI snippet is needed, output it and let the user paste. CI integration is the most-resented automation surface — even users who otherwise embrace agent-driven setup will stop the session if they see their `.github/workflows/build.yml` change unexpectedly.
- **Don't write a new CI config from scratch.** If none exists, see `references/build-output-and-feeds.md` §"no CI yet" — push to the built-in feed locally for now, or ask the user to add CI themselves.
- **Don't commit anything to the user's repo.** This includes `.octopus/` Config-as-Code skeletons, generated `values.yaml`, sample manifests, anything. If the user explicitly says "yes, commit it," fine. Otherwise output as a paste block.
- **Don't create default variables, tag sets, or channels "for completeness."** Empty placeholder structures register as noise — users assume they're load-bearing and waste effort understanding why they exist.
- **Don't auto-trigger the first deploy.** Show the deploy command and the UI link; let the user fire it. Users universally want this trigger pulled themselves so they can watch the logs scroll.
- **Don't ask the user to type any Octopus ID by hand** (`Spaces-1`, `Projects-2`, `Environments-3`). Pick by display name; let the API resolve to the ID. Typed IDs are an error-prone and customer-hostile pattern.
- **Don't use *tenant* / *space* / *channel* / *runbook* / *lifecycle* without grounding them in the user's domain first.** Lead with the translated word; the Octopus-native term comes after the user has the concept.
- **Don't pre-configure rollback.** Users want to *understand* rollback mechanics — they don't want a rollback step set up unprompted, because it suggests the deploy is expected to fail.
- **Don't create extra environments "just in case."** Dev / Staging / Prod is fine when the user wanted a multi-environment pipeline. A user who wants "just deploy" gets one environment, not three.
- **Don't enable multi-tenancy unless the user is on the platform-engineer branch (`SKILL.md` §3C) or has named multiple customers / deployment slices.** Once enabled, the feature flag stays enabled — and it adds confusing fields to most API responses.
