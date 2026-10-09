# CLAUDE.md: payments-gitops

GitOps repo for the **payments** domain. It deploys podinfo as a `WebApi` named `payments-api` to `payments-dev`, `payments-qa` and `payments-prod`, and defines the Kargo project that promotes it.

## Layout

- `manifests/payments-api/base/webapi.yaml`: the `WebApi`. `spec.image.tag` is overwritten by Kargo during promotion.
- `manifests/payments-api/overlays/{dev,qa,prod}/`: replicas, resources and UI message/color per env.
- `apps/{dev,qa,prod}/`: one Argo CD Application per env, each syncing `stage/<env>`. These branches are written only by Kargo; never edit them by hand.
- `apps/kargo.yaml`, `kargo/`: Kargo Project, ProjectConfig, Warehouse (Git commits under `manifests/payments-api/` plus podinfo semver tags) and Stages. dev auto-promotes; qa and prod are promoted from the previous stage.
- Bounded by the `payments-project` AppProject in platform-gitops: `payments-*` namespaces and this repo only. Namespaces come from platform-gitops.

## Build, test, run

No build. Validate before pushing:

```bash
for e in dev qa prod; do kustomize build manifests/payments-api/overlays/$e >/dev/null; done
kustomize build kargo >/dev/null
```

Changes reach the cluster by pushing to `main` and promoting through Kargo (UI on the `kargo` namespace, or `kargo promote`). Check with `kubectl get webapi -n payments-dev`.

## The five-repo platform

| Repo | Role |
|---|---|
| `webapi-operator` | Go/kubebuilder operator. Owns the `WebApi` CRD (`platform.aaronstahley.dev/v1alpha1`) and reconciles it into a Deployment and Service. Ships a Helm chart and a GHCR image. |
| `platform-gitops` | Platform team repo. Argo CD app-of-apps (`apps/infrastructure`, `apps/domains`), AppProjects as tenant boundaries, namespaces, Kargo, operator instances per env, docs. |
| `platform-bootstrap` | Day-0: kind cluster, Argo CD install, credentials, root app-of-apps. `make all`. |
| `payments-gitops` | Domain repo: podinfo as a `WebApi`, Kustomize overlays per env, Kargo warehouse and stages. |
| `orders-gitops` | Same shape as payments, for the orders domain. |

Flow: a developer changes a `WebApi` in a domain repo on Git. Kargo turns the change into Freight and promotes dev -> qa -> prod by rendering into `stage/<env>` branches. Argo CD syncs each env from its branch, and the operator reconciles the `WebApi`.

## Conventions (all repos)

- Commit as `Aaron Stahley <AaronStahley@users.noreply.github.com>` (set via local `git config`).
- Keep commits small and meaningful.
- Never hand-edit generated files (deepcopy, CRDs, RBAC). In webapi-operator run `make manifests generate`, then refresh the Helm chart's CRD copy in `helm/webapi-operator/crds/`.
- Demo workload is podinfo: `ghcr.io/stefanprodan/podinfo` `6.15.0`, port `9898`, liveness `/healthz`, readiness `/readyz`.
- Domains are `payments` and `orders`. CRD group/version is `platform.aaronstahley.dev/v1alpha1`.
- These repos are public. No secrets, tokens or personal email addresses.
- Run tests and the build after changes, and only commit/push when they pass.
