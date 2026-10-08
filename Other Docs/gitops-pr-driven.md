# PR-Driven GitOps from a Developer Portal — Tool Landscape

> Living version (Claude Doc): https://claude.ai/code/artifact/0f830e86-c641-443c-8584-0c77fbf54fb8
> Snapshot exported 2026-10-05. The reference-architecture diagram is only in the living doc.

Oct 4, 2026 · @vajira

## TL;DR

As of October 2026, no single tool gives developers the whole loop in one Backstage-style UI on stock Argo CD or Flux. The loop is: golden-path form, PR, checks and review, merge, sync, status, then day-2 changes and promotion. There are three realistic routes, and most teams combine them.

- **Assemble it in Backstage (open source).** Scaffolder actions open PRs and MRs, Roadie's utils actions edit YAML in place, forge plugins list PRs, and the community Argo CD and Flux plugins show sync and health. The missing piece is a small custom tracker that links a template run to its PR, merge commit, GitOps revision and health.
- **Adopt a platform that already treats a change as a PR.** Plural is the most complete, but uses its own console and engine. Harness IDP with GitOps PR pipelines, Kratix SKE's Backstage PR mode, and Port or Cortex workflows (Argo CD-centric) come next.
- **Add a promotion engine for environment-to-environment moves.** Kargo has open, wait and merge PR steps and a promotion UI. GitOps Promoter keeps one PR per environment with gates as commit statuses, but is still experimental. Neither has a Backstage plugin.

Two 2026 trends make the build route cheaper. GitOps engines now write status into the PR itself (Flux 2.8 PR comments and commit statuses, Argo CD notifications). Backstage MCP actions also let AI agents run the same golden-path templates.

OpenChoreo already has the GitOps data model and workflows that open PRs. Its gap is portal actions that open PRs, plus a portal view of PR and Flux state.

## The target experience

The ask breaks into eight capabilities. The matrix and the sections below score each tool against them.

1. **Golden path.** Platform engineers publish templates and policies. Developers pick one and fill in a form.
2. **Create a PR to deploy.** The form renders manifests or values into the GitOps repo and opens a PR or MR.
3. **See the PR's status.** Checks (diff preview, policy), reviews and merge state, linked to the request that created the PR.
4. **See deployment status after merge.** Which revision Argo CD or Flux synced, its health, and what runs in each environment.
5. **Day-2 changes via PR.** Bump an image tag, or change env vars, replicas or config, from the same UI.
6. **Promote via PR.** Move a release from dev to staging to prod, with gates and approvals.
7. **Preview environments.** One environment per PR, torn down on merge or close.
8. **Rollback.** A revert PR, or a promotion of an older release.

Approval lives in the forge. GitHub and GitLab environment protection rules gate CI jobs, not pull-based syncs, so PR review (CODEOWNERS, branch protection) is the real gate.

## Landscape at a glance

Plural is the only tool that covers all six PR-loop capabilities natively, and it does so on its own console and engine. Everything built on Backstage with Argo CD or Flux needs some glue.

Key: ✓ native · ◐ partial or needs glue · — not covered · commit = writes straight to Git, no PR · pipeline = promotes through the platform, not a PR

| Tool | Licence | Engine | Create PR | PR status | Deploy status | Day-2 PR | Promote PR | Previews |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| [Plural](https://docs.plural.sh/plural-features/pr-automation) | AGPL-3.0 core + paid tiers | Own pull agent | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| [Harness IDP + GitOps PR pipelines](https://developer.harness.io/docs/continuous-delivery/gitops/pr-pipelines/pr-pipelines-basics/) | Commercial | Argo CD | ✓ | ◐ | ✓ | ✓ | ✓ | ◐ |
| [Port](https://docs.port.io/workflows/overview/) | Commercial | Argo CD | ✓ | ✓ | ✓ | ✓ | ✓ | — |
| [Kratix SKE, Backstage PR mode](https://docs.kratix.io/ske/releases/backstage) | Commercial (Kratix is Apache-2.0) | Flux or Argo CD | ✓ | ✓ | ✓ | ✓ | — | — |
| [Cortex](https://docs.cortex.io/workflows/blocks) | Commercial | Argo CD (events only) | ✓ | ✓ | ◐ | ✓ | ◐ | — |
| Backstage, assembled from OSS plugins | Apache-2.0 | Argo CD or Flux | ✓ | ◐ | ✓ | ◐ | ◐ | — |
| [Red Hat Developer Hub](https://developers.redhat.com/articles/2026/02/09/how-integrate-developer-hub-openshift-gitops) | Commercial | Argo CD | ✓ | ◐ | ✓ | ◐ | — | — |
| [Octopus Deploy, Argo CD steps](https://octopus.com/docs/argo-cd/steps) | Commercial | Argo CD | ◐ | ◐ | ✓ | ✓ | ✓ | — |
| [Kargo](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/git-open-pr) | Apache-2.0 (+ Akuity Platform) | Argo CD | — | ◐ | ✓ | ◐ | ✓ | — |
| [GitOps Promoter](https://github.com/argoproj-labs/gitops-promoter) | Apache-2.0, experimental | Argo CD | — | ✓ | ✓ | ◐ | ✓ | — |
| [Devtron](https://docs.devtron.ai/docs/user-guide/app-management/configurations/gitops) | Apache-2.0 + enterprise | Argo CD or Flux | commit | — | ✓ | commit | pipeline | — |
| [Akamai App Platform](https://techdocs.akamai.com/app-platform/docs/team-workloads) | Apache-2.0 | Argo CD | commit | — | ✓ | commit | — | — |
| [OpenChoreo](https://openchoreo.dev/docs/platform-engineer-guide/gitops/overview/), today | Apache-2.0 | Flux | ◐ | — | ◐ | ◐ | ◐ | — |

Engine UIs that only show status (Argo CD, Flux Web UI, Rancher Fleet) and portals that don't open PRs are covered in later sections, not the matrix.

## Backstage, assembled from open-source plugins

Every step of the loop has a maintained Backstage building block in 2026, but nothing joins them into one tracked change. Versions below are as of Backstage v1.55.0 (15 Sep 2026).

| Capability | Building block | What it does and what it lacks |
| --- | --- | --- |
| Create PR (GitHub) | [`publish:github:pull-request`](https://roadie.io/backstage/scaffolder-actions/publish-github-pull-request/) | Commits only the files under `sourcePath` to `targetPath`. `update: true` reuses the branch and updates the open PR, so day-2 templates can be re-run. Has reviewers, team reviewers and draft inputs, but no labels or auto-merge. |
| Create MR (GitLab) | [`publish:gitlab:merge-request`](https://github.com/backstage/backstage/blob/master/plugins/scaffolder-backend-module-gitlab/src/actions/gitlabMergeRequest.ts) | `commitAction: auto` diffs each file against the repo. Supports labels and reviewers from approval rules; no draft input. |
| Create PR (Bitbucket) | [`publish:bitbucketServer:pull-request`](https://github.com/backstage/backstage/blob/master/plugins/scaffolder-backend-module-bitbucket-server/src/actions/bitbucketServerPullRequest.ts), `publish:bitbucketCloud:pull-request` | Clones the repo and overlays the workspace. Cannot update an existing PR. |
| Create PR (Azure DevOps) | [Community Azure DevOps module](https://github.com/backstage/community-plugins/blob/main/workspaces/azure-devops/plugins/scaffolder-backend-module-azure-devops/README.md) | `azure:repository:clone`, then `azure:repository:push`, then `azure:pr:create`, with auto-complete and merge strategy. |
| Create PR (Gitea) | [`publish:gitea`](https://github.com/backstage/backstage/blob/master/plugins/scaffolder-backend-module-gitea/README.md) | Creates repos only; there is no PR action. |
| Edit YAML in place | `fetch:plain:file` + [Roadie utils](https://github.com/RoadieHQ/roadie-backstage-plugins/blob/main/plugins/scaffolder-actions/scaffolder-backend-module-utils/README.md) (`roadiehq:utils:merge`, `jsonata:yaml:transform`) or [`regex:replace`](https://roadie.io/backstage/scaffolder-actions/regex-replace/) | Fetch the environment's values file, merge the new tag or replica count, then open the PR on a predictable branch. |
| Launch from an entity page | `?formData=` deep links; [SeatGeek entity scaffolder](https://github.com/seatgeek/backstage-plugins/blob/main/plugins/entity-scaffolder-content/README.md); [RHDH Orchestrator](https://github.com/redhat-developer/rhdh-plugins/blob/main/workspaces/orchestrator/README.md) | Upstream has no "actions for this entity" launcher. The SeatGeek plugin was last released in Sep 2024; Orchestrator is OpenShift-only. |
| PR status | [Roadie GitHub PRs](https://github.com/RoadieHQ/roadie-backstage-plugins/blob/main/plugins/frontend/backstage-plugin-github-pull-requests/README.md), [GitHub PR board](https://github.com/backstage/community-plugins/tree/main/workspaces/github/plugins/github-pull-requests-board), [GitLab](https://github.com/immobiliare/backstage-plugin-gitlab), [Azure DevOps](https://github.com/backstage/community-plugins/tree/main/workspaces/azure-devops/plugins/azure-devops), [Bitbucket PRs](https://github.com/backstage/community-plugins/tree/main/workspaces/bitbucket-pull-requests) | Lists PRs per entity, team or user; Roadie adds home-page "your open PRs" and "review requests" cards. None ties a PR to the template run that created it. |
| Sync and health | [Roadie Argo CD](https://roadie.io/backstage/plugins/argo-cd/), [community Argo CD](https://github.com/backstage/community-plugins/tree/main/workspaces/argocd/plugins/argocd), [community Flux](https://github.com/backstage/community-plugins/tree/main/workspaces/flux/plugins/flux), core Kubernetes plugin | The community Argo CD plugin (formerly `plugin-redhat-argocd`) also shows Argo Rollouts. The Flux plugin replaces the archived Weaveworks one and can sync, suspend and resume. |
| Push events to developers | [Notifications + Signals](https://backstage.io/docs/notifications/), [Argo CD notifications webhook](https://github.com/backstage/community-plugins/blob/main/workspaces/argocd/docs/notifications-integration.md), [GitHub events module](https://github.com/backstage/backstage/blob/master/plugins/events-backend-module-github/README.md) | Argo CD posts sync succeeded or failed, health degraded and deployed events, routed by the `backstage.io/entity-ref` annotation. |
| AI agents | [MCP actions](https://backstage.io/docs/ai/mcp-actions) (v1.40), [`execute-template`](https://backstage.io/docs/releases/v1.50.0/) (v1.50) | Agents can run the same golden-path templates and read task logs. |

What teams build themselves:

- A change tracker that links a template run to its PR, merge SHA, Argo CD or Flux revision and health, fed by forge webhooks.
- A per-component environment and promotion view. No Kargo or GitOps Promoter plugin exists; kubriX embeds Kargo through an iframe.
- Preview-environment visibility, plus label, auto-merge and revert actions for GitHub PRs.

## Commercial Backstage distributions

Only Harness ships a PR-native GitOps flow behind its portal. The other distributions add hosting, RBAC and polish, but have the same gaps as open-source Backstage.

| Product | What it adds for this loop | What is still missing |
| --- | --- | --- |
| [Harness IDP + GitOps PR pipelines](https://developer.harness.io/continuous-delivery/use-gitops/pr-pipelines/pr-pipelines-basics) | Portal workflows trigger Harness pipelines. **Update Release Repo** commits to a new branch and raises a PR; `waitForMerge` blocks until it is merged in the forge. Merge PR, Revert PR, GitOps Sync and GitOps Rollback cover the rest. [Linked workflows](https://developer.harness.io/internal-developer-portal/use-idp/self-service-workflows/link-workflows-to-entities) add an Execute button on entity pages, prefilled from the entity. Works with GitHub, GitLab, Bitbucket Cloud and Server, Harness Code and Azure Repos. | Argo CD only. PR state shows in the pipeline run rather than on an entity card. |
| [Red Hat Developer Hub](https://developers.redhat.com/blog/2026/03/13/whats-new-red-hat-developer-hub-19) 1.10 (Backstage 1.49.4) | Argo CD plugin with Argo Rollouts, multi-cluster since 1.9. Tekton and Topology plugins. Bulk Import (Tech Preview) opens `catalog-info.yaml` PRs and tracks them as "Waiting for approval" or "Imported". Orchestrator workflows and MCP tools. | Red Hat's 2026 [golden-path articles](https://developers.redhat.com/articles/2026/02/09/how-integrate-developer-hub-openshift-gitops) push repos directly and let Argo CD sync; there is no PR-based promotion. |
| [Roadie](https://roadie.io/docs/api/roadie-mcp/scaffolder/) | Hosted Backstage. Scaffolder actions can run inside your network through the Roadie Agent and Broker. Argo CD integration, plus a Scaffolder MCP server that finds, validates and runs templates and reports task status. | No change tracker or promotion view. |
| [Spotify Portal](https://portal.spotify.com/) | Managed Backstage with an in-portal template editor and dry-run. Exposes Catalog, Soundcheck and Scaffolder over MCP. Fleetshift opens PRs across many repos (GitHub, Azure DevOps) and tracks their activity. | No GitOps-specific features documented. |
| [Tanzu Developer Portal](https://techdocs.broadcom.com/us/en/vmware-tanzu/standalone-components/tanzu-application-platform/1-12/tap/release-notes.html) | Ships inside Tanzu Application Platform 1.12 (LTS). | Nothing PR- or GitOps-specific found. |

## GitOps-native promotion tools

For environment-to-environment moves, Kargo and GitOps Promoter are the open-source tools that treat a promotion as a PR. Octopus and Harness do the same commercially, and Codefresh's promotions have been switched off.

| Tool | How a promotion becomes a PR | What its UI shows | Status, Oct 2026 |
| --- | --- | --- | --- |
| [Kargo](https://docs.kargo.io/user-guide/reference-docs/promotion-steps) | A Warehouse spots a new image, chart or commit and creates Freight. Each Stage's promotion runs `git-open-pr`, then `git-wait-for-pr` (polling or webhook), then optionally [`git-merge-pr`](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/git-merge-pr) (synchronous; no merge queues), then `argocd-update`. Verification runs Argo Rollouts analysis. Rollback means promoting older Freight. | Stage graph, step-by-step state with inline logs, PR deep links, Freight diffs. | v1.12.0 (30 Sep 2026), Apache-2.0. Its only engine-specific steps are `argocd-update` and `argocd-wait`; there is no Flux step. No Backstage plugin. |
| [GitOps Promoter](https://github.com/argoproj-labs/gitops-promoter) | A hydrator (Argo CD Source Hydrator or your own) writes rendered manifests to `<env>-next` branches. One PR per environment stays open from `<env>-next` into `<env>`. Gates are commit statuses (Argo CD health, dependents, soak time, schedule, CEL, HTTP) that show as PR checks; auto-merge is on by default. | Standalone dashboard, plus an Argo CD UI extension that links each environment to its PR. | v0.42.1 (24 Sep 2026); the README says "experimental, please use with caution". Supports GitHub, GitLab, Gitea/Forgejo, Bitbucket Cloud and Azure DevOps. |
| [Octopus Deploy, Argo CD steps](https://octopus.com/docs/argo-cd/steps) | "Update Argo CD Application Image Tags" and "Update Argo CD Application Manifests" commit directly or open a PR, chosen per environment. PRs work on GitHub, GitLab and Azure DevOps. Steps can wait for the PR to merge (2026.2+) and for the app to be healthy (2026.1+). | Project dashboard with each environment's release and live Argo CD health. | Commercial. [Platform Hub](https://octopus.com/docs/platform-hub/templates) adds Git-backed process templates and Rego policies as golden paths. |
| [Harness GitOps PR pipelines](https://developer.harness.io/continuous-delivery/use-gitops/pr-pipelines/pr-pipelines-basics) | Update Release Repo raises the PR, then Merge PR or wait for an external merge. Revert PR handles rollback. | Pipeline run view and GitOps Applications. | Commercial, Argo CD-based. |
| [Codefresh GitOps](https://codefresh.io/docs/docs/promotions/promotion-policy/) | Promotion policies could commit, open a PR, or take no action. | — | "Promotions has been disabled and turned off" for runtimes after 0.24.0. Treat it as sunset. |
| [Weave GitOps Enterprise](https://github.com/weaveworks/weave-gitops-enterprise) | UI templates opened PRs; pipelines promoted by PR. | — | The repo went public after Weaveworks closed, with no maintained releases. Treat it as orphaned. |

## Platforms whose UI writes to Git

What matters most is what a UI click does: open a PR, commit straight to Git, or apply through an API. Only the PR family keeps review in the forge. Rows below are grouped in that order.

| Platform | A UI click becomes | Engine | How it fits the loop | Latest |
| --- | --- | --- | --- | --- |
| [Plural](https://docs.plural.sh/plural-features/pr-automation) | PR | Own pull agent | Platform engineers write `PrAutomation` resources: typed form fields, Liquid templates, YAML overlays and regex edits. The console renders the form and lists the PRs. Pipelines raise promotion PRs, Flows give per-PR [preview environments](https://docs.plural.sh/plural-features/flows/preview-environments), and merges can wait on ServiceNow approval. Works with GitHub, GitLab, Bitbucket and Azure DevOps. | Console v0.12.37 (14 Aug 2026) |
| [Kratix SKE](https://docs.kratix.io/ske/releases/backstage) | PR (Backstage PR mode); direct commit (state store) | Flux or Argo CD | Since ske-backend v0.21.0 (30 Jul 2026), templates write the request to a repo and open a PR. The pending request shows as a catalog Component linking to its PR, and Update or Delete raise PRs too. | SKE v0.60.0 (29 Sep 2026) |
| [Konflux](https://konflux-ci.dev/docs/building/component-nudges/) | PR, CI side only | Tekton, Pipelines as Code | Onboarding opens a PR that adds `.tekton/` pipelines. "Nudge" PRs bump image digests in dependent repos. No deployment view. | Pre-release v0.2.2-rc.12 |
| [Porter](https://docs.porter.run/preview-environments/overview) | PR to add the workflow file, then API | Own | Per-PR preview environments. | Commercial |
| [Devtron](https://docs.devtron.ai/docs/user-guide/app-management/configurations/gitops) | Direct commit | Argo CD or Flux | One repo per app with values per environment. Review happens in Devtron's own config drafts and approvals (enterprise), not in PRs. | v2.2.0 (Jul 2026) |
| [Akamai App Platform](https://github.com/linode/apl-api) (formerly Otomi) | Direct commit | Argo CD | "Every api deployment will result in a commit to the values repo." An admin-curated catalog of Helm charts acts as the golden paths. | apl-core v6.4.0 (30 Sep 2026) |
| [Mia-Platform Console](https://docs.mia-platform.eu/docs/products/console/set-up-infrastructure/enhanced-project-workflow) | Direct commit of generated manifests | Argo CD or Flux | Project state lives in Console; manifests for each environment are written to Git at deploy time. | v15.2.0 (21 Sep 2026) |
| [Cyclops](https://github.com/cyclops-ui/cyclops/blob/main/web/docs/installation/git-write.md) | Direct commit (opt-in) | Argo CD | Forms generated from Helm charts; two Backstage plugins. Activity is slowing. | v0.21.1 (Jun 2025) |
| [Northflank](https://northflank.com/docs/v1/application/infrastructure-as-code/gitops-on-northflank) | Direct commit (two-way template sync) | Own | Release flows can trigger on PRs. | Commercial |
| [KubeVela + VelaUX](https://kubevela.io/docs/end-user/gitops/fluxcd/) | API apply | KubeVela, plus a Flux addon | VelaUX keeps state in its own database. Its last release was Mar 2025. | KubeVela v1.11.0 (Jul 2026) |
| [Radius](https://docs.radapp.io/guides/tooling/dashboard/overview/) | None; read-only dashboard built on Backstage | Radius + Flux controller | Recipes are the golden paths. | v0.59.0 (Jun 2026) |
| [Humanitec](https://developer.humanitec.com/platform-orchestrator/docs/deploy/deployments/) | API (current orchestrator) | Terraform/OpenTofu runners | Its Backstage plugin was archived in Mar 2026; the older v1 had a GitOps mode. | SaaS |
| Qovery, Kubero, Kusion | API apply | Own | Qovery and Kubero create a preview environment per PR. | — |

Direct commit keeps Git as the audit log but moves review into each platform's own UI, so CODEOWNERS and branch protection no longer gate the change.

## Non-Backstage portals

Port and Cortex can run the whole loop from their own UI. Both are Argo CD-centric, though, and treat PRs and deployments as separate entities that you have to link yourself.

| Portal | Create the PR | See PR status | See deploy status | Notes |
| --- | --- | --- | --- | --- |
| [Port](https://docs.port.io/workflows/overview/) | [Workflows](https://www.port.io/blog/port-workflows) (GA 28 Aug 2026) have [GitHub nodes](https://docs.port.io/workflows/build-workflows/action-nodes/integration-actions/git/github/) to open, update, review and merge PRs, but only between existing branches. File edits come from a dispatched GitHub workflow running `gh pr create`, or from a coding agent. | GitHub, GitLab and Azure DevOps integrations ingest PRs as catalog entities through webhooks. | The [Argo CD integration](https://docs.port.io/build-your-software-catalog/sync-data-to-catalog/argocd/) ingests apps with `syncStatus`, `healthStatus` and deployment history. No first-party Flux integration found. | Free tier up to 15 seats. Guides cover promotion and rollback. AI agents call allowlisted actions, each set to run automatically or after approval. |
| [Cortex](https://docs.cortex.io/workflows/blocks) | GitHub blocks create a branch, create or update a file, open a PR and merge. GitLab and Azure DevOps have file blocks but no PR block. The Scaffolder can open PRs on GitHub, GitLab, Bitbucket and Azure DevOps. | Homepage shows "Your open PRs or MRs" and requested reviews, refreshed every 2 minutes. | Argo CD notifications arrive as deploy events; there is no live sync or health state. | The Cortex MCP server (GA Oct 2025) is read-only. |
| [OpsLevel](https://docs.opslevel.com/docs/getting-started-with-custom-actions) | Actions are webhooks behind forms; service templates create repos. No PR template. | No native PR inbox found. | An Argo CD PostSync hook reports deploy events. | MCP server since Jun 2025. |
| [Atlassian Compass](https://www.atlassian.com/blog/company-news/the-next-chapter-for-compass) | — | — | — | Being phased out in favour of DX Fabric, announced 13 Apr 2026. |

## Closing the loop: status in the PR, diffs and previews

The cheapest way to show PR and deployment status together is to write deployment status into the PR itself. Both engines now do this, and diff bots cover the pre-merge side.

| Need | Argo CD | Flux |
| --- | --- | --- |
| Status written back to the PR or commit | [Notifications, GitHub service](https://argo-cd.readthedocs.io/en/latest/operator-manual/notifications/services/github/): commit status, Deployments, check runs, and PR comments updated in place via `commentTag`. Other forges need generic webhooks. | [notification-controller](https://fluxcd.io/flux/components/notification/providers/): commit status on GitHub, GitLab, Gitea, Bitbucket and Azure DevOps for any Flux kind. [Flux 2.8](https://fluxcd.io/blog/2026/02/flux-v2.8.0/) (24 Feb 2026) added PR and MR comment providers: `githubpullrequestcomment`, `gitlabmergerequestcomment`, `giteapullrequestcomment`. |
| Diff in the PR before merge | [kubechecks](https://github.com/zapier/kubechecks) posts diffs, lint and policy results as PR or MR comments (GitHub, GitLab). [argocd-diff-preview](https://github.com/dag-andersen/argocd-diff-preview) renders both branches through a real Argo CD. | [konflate](https://github.com/home-operations/konflate) gives a rendered-diff review and replaces the deprecated flux-local. |
| Preview environment per PR | [ApplicationSet PR generator](https://argo-cd.readthedocs.io/en/latest/operator-manual/applicationset/Generators-Pull-Request/) for GitHub, GitLab, Gitea, Bitbucket and Azure DevOps. Argo CD 3.5 adds an ApplicationSet UI with a preview-apps tab. | [Flux Operator ResourceSetInputProvider](https://fluxoperator.dev/docs/resourcesets/github-pull-requests/): label-gated, removed when the PR merges or closes, with status posted back as a PR comment and a commit check. |
| Image bump PRs | [Image Updater](https://github.com/argoproj-labs/argocd-image-updater/releases) can open PRs or MRs instead of committing, since v1.2 (GitHub, GitLab). | Image automation pushes to a branch, and a [GitHub Action](https://fluxcd.io/flux/use-cases/gh-actions-auto-pr/) opens the PR. Renovate works with both engines. |
| Promotion PRs without a platform | [Telefonistka](https://github.com/commercetools/telefonistka) opens promotion PRs between directories and posts Argo CD diffs (GitHub only). | Template-generated PRs. Kargo's Git steps work, but it has no Flux sync step. |
| Engine UI | Argo CD 3.5; the 3.6 RC adds a Resources Explorer. The [Source Hydrator](https://argo-cd.readthedocs.io/en/latest/user-guide/source-hydrator/) (beta) pushes to a staging branch but "will not create a PR". | [Flux Web UI](https://fluxoperator.dev/web-ui/) in Flux Operator (AGPL-3.0): dashboards, GitOps graph, history, logs, OIDC SSO, and reconcile, suspend and resume guarded by RBAC. It shows no PRs. |

Other UIs only show status. GitLab's [environment Kubernetes dashboard](https://docs.gitlab.com/ci/environments/kubernetes_dashboard/) shows Flux badges and can reconcile (17.3+) and suspend or resume (17.5+). Rancher Fleet, Capacitor, Headlamp's Flux plugin and Weave GitOps OSS (now a community project) are status-only.

## Cloud providers and AI agents

Cloud consoles show status but don't run a PR loop. Only Azure opens a PR from its portal, and that flow deploys through CI, not GitOps.

| Provider | What developers get | PR-driven? |
| --- | --- | --- |
| Azure | [AKS Automated Deployments](https://learn.microsoft.com/en-us/azure/aks/automated-deployments) generates a Dockerfile, manifests and a pipeline, then opens a PR on GitHub or Azure DevOps. Merging it runs CI that builds and deploys. [Fleet Manager](https://learn.microsoft.com/en-us/azure/kubernetes-fleet/howto-automated-deployments) has a similar flow (preview, GitHub only). The Flux extension has a GitOps blade showing compliance; the Argo CD extension is in public preview, with a portal view announced in May 2026. [AKS desktop](https://learn.microsoft.com/en-us/azure/aks/aks-desktop-overview), built on Headlamp, deploys apps directly through "Projects". | Onboarding PR only; deployment is a CI push. |
| AWS | [EKS Capabilities](https://aws.amazon.com/about-aws/whats-new/2025/11/amazon-eks-capabilities) (30 Nov 2025) gives managed Argo CD, ACK and kro, with the hosted Argo CD UI. [AWS Proton](https://docs.aws.amazon.com/proton/latest/userguide/Welcome.html) reaches end of support on 7 Oct 2026. [appmod-blueprints](https://github.com/aws-samples/appmod-blueprints) is a reference IDP built from Backstage, Argo CD, Argo Workflows and Kargo. | No |
| Google Cloud | The [Config Sync dashboard](https://docs.cloud.google.com/kubernetes-engine/enterprise/config-sync/docs/how-to/use-config-sync-dashboard) shows reconciliation and sync status; fleet packages roll out from Git. | No |

AI agents are converging on the PR as their approval boundary:

- Backstage [MCP actions](https://backstage.io/docs/ai/mcp-actions) include `execute-template` (v1.50), so agents run the same golden-path templates as people.
- [Akuity](https://docs.akuity.io/changelog/cloud/) can make code changes and open PRs (2 Apr 2026). Its [Agentic Control Plane](https://akuity.io/blog/getting-started-agentic-control-plane) (17 Sep 2026) is an MCP server over Argo CD and Kargo, with read-only, read-write and full tiers.
- [Port AI agents](https://docs.port.io/agent-management/custom-agents/build-an-ai-agent/) call allowlisted actions, each set to run automatically or after approval.
- The [Flux Operator MCP](https://fluxcd.io/blog/2025/05/ai-assisted-gitops/) and [Argo CD MCP](https://github.com/argoproj-labs/mcp-for-argocd) servers act on the cluster or API, not Git, and both offer read-only modes.

## Reference architecture

The open-source build puts Backstage in front of the forge and the GitOps engine, with one custom component doing the joining.

_Diagram (see the living doc), as text:_

- **Forward path:** Golden-path form (Backstage scaffolder edits values YAML, opens the PR) → PR in GitOps repo (diff preview and policy results as PR checks) → Review and merge (CODEOWNERS, branch protection) → Argo CD or Flux (syncs the merged revision, writes status to the PR).
- **Promotion loop:** the engine step feeds a promotion PR for the next environment (Kargo or GitOps Promoter) back into the PR step.
- **Feedback path:** PR opened/checks, merge SHA, and revision/health events → Change tracker (custom Backstage backend plugin; joins template run, PR, merge SHA, synced revision and health per change; fed by forge webhooks plus Argo CD notifications or Flux alerts) → Entity page (PR status, sync, health, promotions).

Read the top row left to right for the forward path and the arrows down for the feedback path. Every forward step exists today; the tracker turns separate PR, merge and sync events into one status per change.

- **Argo CD variant.** `publish:github:pull-request` with `update: true` and Roadie's merge action. The community Argo CD plugin, with Rollouts. Argo CD notifications into Backstage and onto the PR as commit status, deployments or comments. argocd-diff-preview or kubechecks for pre-merge diffs. Kargo or GitOps Promoter for promotion, and the ApplicationSet PR generator for previews.
- **Flux variant.** The same portal pieces with the community Flux plugin. Flux commit status and PR comment providers (2.8+), konflate for rendered diffs, and Flux Operator `ResourceSetInputProvider` for previews. Promotion as template-generated PRs: Kargo has no Flux sync step, and GitOps Promoter's built-in health gate reads Argo CD.

## OpenChoreo gap analysis

OpenChoreo already has the hard parts of a PR-driven flow: immutable releases, per-environment bindings, offline generation with `occ`, Flux layouts, and workflows that open PRs. The gap is on the developer side: portal actions that open PRs, and a portal view of PR and Flux state.

Today the portal, `occ` and the MCP servers all go through the control-plane API. The [architecture docs](https://openchoreo.dev/docs/overview/architecture/) describe running it "imperatively via the UI, CLI, and MCPs, or declaratively with Git as the source of truth". The Git path runs through Workflows: [build-and-release](https://openchoreo.dev/docs/platform-engineer-guide/gitops/automations/build-and-release-workflows/) opens a PR with the Workload, ComponentRelease and ReleaseBinding, and [bulk promote](https://openchoreo.dev/docs/platform-engineer-guide/gitops/automations/bulk-promote/) opens a PR with generated ReleaseBindings. Latest release: v1.3.1 (30 Sep 2026).

| Capability | OpenChoreo today | Gap | Pattern to borrow |
| --- | --- | --- | --- |
| Create a PR to deploy | Portal [scaffolder actions](https://openchoreo.dev/docs/platform-engineer-guide/backstage-scaffolder-templates/) such as `openchoreo:component:create` call the API. The build-and-release workflow opens a release PR. | No portal action ends in a PR. | Kratix SKE PR mode: the template writes the request, opens a PR, and publishes a pending Component that links to it. |
| See the PR's status | Not shown in the portal. | Developers leave the portal to follow the PR. | SKE's pending-request entity; Plural's PR list per catalog item; forge PR cards on the component page. |
| Deployment status | The [Deploy tab](https://openchoreo.dev/docs/developer-guide/deploying-applications/deploy-and-promote/) shows Ready, NotReady or Failed per environment, with image, release name and endpoints. | No Flux revision or Kustomization health, and no link from a binding's status back to the PR that changed it. | Flux 2.8 commit status and PR comment providers on the GitOps repo; the Argo CD-to-Backstage notifications pattern, fed by Flux alerts instead. |
| Day-2 changes via PR | Workflows and `occ` only. | No form-driven PR that edits Workload or ReleaseBinding overrides. | Plural `PrAutomation` updates (YAML overlays, regex); `fetch:plain:file` + Roadie merge + `publish:github:pull-request` with `update: true`. |
| Promote via PR | The Promote button calls the API; bulk promote opens a PR. | No single-component promotion PR from the portal, and no gates. | GitOps Promoter: one PR per environment, gates as commit statuses. Kargo: open PR, wait for merge, then verify. |
| Preview environments | None. | No per-PR environment. | Flux Operator `ResourceSetInputProvider` creating a temporary Environment and ReleaseBinding per labelled PR. |
| Mixed mode | The portal applies through the API while Flux reconciles Git. | A portal write to a Flux-owned resource would be reverted on the next sync. | SKE's Hybrid mode: read live status from the cluster, write changes to Git. |

Where PR creation could live:

- **Backstage backend (scaffolder actions).** Fastest to ship, reusing `publish:*:pull-request`. But `occ` and MCP users would not share the path.
- **Control-plane API, in a GitOps mode.** The API accepts the same intent and returns a pending change backed by a PR, so portal, `occ` and MCP share one path. It needs SCM credentials and webhook intake in the control plane.
- **Workflow plane.** It already opens release and bulk-promote PRs and suits long waits, such as wait for merge then verify. It adds latency to small edits.

A split worth prototyping: the API records intent as a pending-change resource, the workflow plane renders files with `occ` and opens the PR, and the portal shows the pending change, its PR checks and Flux health on the component page. The closest existing thread is [discussion #1279](https://github.com/openchoreo/openchoreo/discussions/1279), GitOps reference implementations (WIP since Dec 2025).

Two other threads touch this. [#3572](https://github.com/openchoreo/openchoreo/discussions/3572) (deployment tracks) reserves `spec.gitBranch` for later Git wiring. [#3347](https://github.com/openchoreo/openchoreo/discussions/3347) (project release lifecycle) works through field-ownership conflicts between controllers and Flux under server-side apply, which is the same problem as mixed mode.

## Open questions

Each of these changes the design, so settle them before building.

- [ ] **Who authors the PR.** The developer's own SCM identity (Backstage v1.55 added `requireScmUserCredentials`) or a bot or GitHub App? This drives the audit trail, CODEOWNERS behaviour and rate limits.
- [ ] **Where PR creation lives.** Backstage backend, control-plane API or workflow plane (see the gap analysis).
- [ ] **Which forges, in what order.** Backstage's Bitbucket actions can't update an open PR, Gitea has no PR action, and Azure DevOps needs the community module.
- [ ] **How a merge maps to a running release.** PR, merge SHA, Flux `lastAppliedRevision` (or Argo CD sync revision), ReleaseBinding status: the chain needs a stable key, such as the ComponentRelease name in the branch or PR body.
- [ ] **Webhooks or polling for PR state.** Webhooks need org-admin rights to install. Polling needs rate-limit care; SKE runs its rejected-request sweep every 900 s for that reason.
- [ ] **Which approval gates.** CODEOWNERS and branch protection alone, or platform gates (soak time, health of the previous environment) published as commit statuses?
- [ ] **Mixed mode.** Can a project be imperative and GitOps at once? If not, how is the mode chosen per namespace or project, and how are portal writes blocked where Flux owns the resource?
- [ ] **PR granularity and repo layout.** One PR per change, or batched as bulk promote does? Mono-repo or repo per project, and who owns which paths?
- [ ] **Preview environments.** Model them as a temporary Environment plus ReleaseBinding? Who pays for them, and when are they cleaned up?
- [ ] **Rollback.** A revert PR, or a promotion PR that points at an older ComponentRelease?
- [ ] **AI agents.** Should agent-initiated changes always take the same PR path, with the same reviewers?

## Sources

Pages opened during research, 4–5 Oct 2026. Links inside the tables above are sources too.

**Backstage**

- [Backstage v1.50.0 release notes](https://backstage.io/docs/releases/v1.50.0/) — `execute-template` action
- [Backstage MCP actions](https://backstage.io/docs/ai/mcp-actions)
- [`publish:github:pull-request` source](https://github.com/backstage/backstage/blob/master/plugins/scaffolder-backend-module-github/src/actions/githubPullRequest.ts) — `update` input, no labels or auto-merge
- [Roadie scaffolder utils](https://github.com/RoadieHQ/roadie-backstage-plugins/blob/main/plugins/scaffolder-actions/scaffolder-backend-module-utils/README.md)
- [Community Flux plugin README](https://github.com/backstage/community-plugins/blob/main/workspaces/flux/plugins/flux/README.md)
- [Argo CD notifications into Backstage](https://github.com/backstage/community-plugins/blob/main/workspaces/argocd/docs/notifications-integration.md)

**Portals and distributions**

- [Harness GitOps PR pipelines](https://developer.harness.io/continuous-delivery/use-gitops/pr-pipelines/pr-pipelines-basics) and [linked workflows](https://developer.harness.io/internal-developer-portal/use-idp/self-service-workflows/link-workflows-to-entities)
- [What's new in Red Hat Developer Hub 1.9](https://developers.redhat.com/blog/2026/03/13/whats-new-red-hat-developer-hub-19)
- [Port Workflows GA](https://www.port.io/blog/port-workflows), [GitHub workflow nodes](https://docs.port.io/workflows/build-workflows/action-nodes/integration-actions/git/github/), [Argo CD integration](https://docs.port.io/build-your-software-catalog/sync-data-to-catalog/argocd/)
- [Cortex workflow blocks](https://docs.cortex.io/workflows/blocks)
- [The next chapter for Compass](https://www.atlassian.com/blog/company-news/the-next-chapter-for-compass)
- [SKE Backstage release notes](https://docs.kratix.io/ske/releases/backstage) — PR delivery mode, v0.21.0
- [Plural PR automation](https://docs.plural.sh/plural-features/pr-automation) and [preview environments](https://docs.plural.sh/plural-features/flows/preview-environments)
- [Akamai App Platform API (apl-api)](https://github.com/linode/apl-api)

**Promotion and GitOps engines**

- [Kargo promotion steps](https://docs.kargo.io/user-guide/reference-docs/promotion-steps), [`git-merge-pr`](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/git-merge-pr), [v1.12.0 release](https://github.com/akuity/kargo/releases/tag/v1.12.0)
- [GitOps Promoter README](https://github.com/argoproj-labs/gitops-promoter) and [getting started](https://argo-gitops-promoter.readthedocs.io/en/latest/getting-started/)
- [Octopus Argo CD steps](https://octopus.com/docs/argo-cd/steps)
- [Codefresh promotion policy](https://codefresh.io/docs/docs/promotions/promotion-policy/) — "Promotions has been disabled" banner
- [Argo CD Source Hydrator](https://argo-cd.readthedocs.io/en/latest/user-guide/source-hydrator/) and [GitHub notifications](https://argo-cd.readthedocs.io/en/latest/operator-manual/notifications/services/github/)
- [Argo CD Image Updater releases](https://github.com/argoproj-labs/argocd-image-updater/releases)
- [Announcing Flux 2.8 GA](https://fluxcd.io/blog/2026/02/flux-v2.8.0/), [Flux Web UI](https://fluxoperator.dev/web-ui/), [Flux Operator PR previews](https://fluxoperator.dev/docs/resourcesets/github-pull-requests/)
- [kubechecks](https://github.com/zapier/kubechecks), [argocd-diff-preview](https://github.com/dag-andersen/argocd-diff-preview)
- [GitLab Kubernetes dashboard](https://docs.gitlab.com/ci/environments/kubernetes_dashboard/)

**Cloud providers and AI**

- [AKS Automated Deployments](https://learn.microsoft.com/en-us/azure/aks/automated-deployments), [AKS desktop overview](https://learn.microsoft.com/en-us/azure/aks/aks-desktop-overview)
- [Amazon EKS Capabilities](https://aws.amazon.com/about-aws/whats-new/2025/11/amazon-eks-capabilities), [AWS Proton end of support](https://docs.aws.amazon.com/proton/latest/userguide/Welcome.html)
- [Akuity cloud changelog](https://docs.akuity.io/changelog/cloud/), [Akuity Agentic Control Plane](https://akuity.io/blog/getting-started-agentic-control-plane)

**OpenChoreo**

- [Architecture](https://openchoreo.dev/docs/overview/architecture/), [scaffolder templates](https://openchoreo.dev/docs/platform-engineer-guide/backstage-scaffolder-templates/), [deploy and promote](https://openchoreo.dev/docs/developer-guide/deploying-applications/deploy-and-promote/)
- [Build-and-release workflows](https://openchoreo.dev/docs/platform-engineer-guide/gitops/automations/build-and-release-workflows/), [bulk promote](https://openchoreo.dev/docs/platform-engineer-guide/gitops/automations/bulk-promote/), [changelog](https://openchoreo.dev/docs/changelog/)
- [Lessons from Designing OpenChoreo for GitOps](https://medium.com/@vajiraprabuddhaka/lessons-from-designing-openchoreo-for-gitops-6280ac5ce759) (Apr 2026)
- Discussions [#1279](https://github.com/openchoreo/openchoreo/discussions/1279), [#3572](https://github.com/openchoreo/openchoreo/discussions/3572), [#3347](https://github.com/openchoreo/openchoreo/discussions/3347)