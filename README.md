# payments-gitops

GitOps repository for the **payments** domain. It is owned by the payments team and deployed by Argo CD; promotion between environments is handled by Kargo.

This is a demo domain. The workload is [podinfo](https://github.com/stefanprodan/podinfo), a small public web service, declared as a `WebApi` and reconciled by [webapi-operator](https://github.com/AaronStahley/webapi-operator).

## Applications

### payments-api

One `WebApi` deployed to three environments with Kustomize base + overlays:

| Environment | Namespace | Replicas | Source branch |
|---|---|---|---|
| dev | `payments-dev` | 1 | `stage/dev` |
| qa | `payments-qa` | 2 | `stage/qa` |
| prod | `payments-prod` | 3 | `stage/prod` |

- **Base** (`manifests/payments-api/base/`): the `WebApi` shared by every environment
- **Overlays** (`manifests/payments-api/overlays/{dev,qa,prod}/`): replicas, resource sizing, and a per-environment UI message and color

## How a change reaches production

1. A change to `manifests/payments-api/` on `main`, or a new podinfo release, becomes new Freight in the Kargo Warehouse (`kargo/warehouses/`).
2. The `dev` stage promotes automatically. It renders the manifests with the new image tag and commits them to the `stage/dev` branch.
3. Argo CD syncs `payments-api-dev` from `stage/dev`.
4. `qa` and `prod` promote on approval, using the same mechanism and the same Freight that passed the previous stage.

The `stage/*` branches are created by Kargo on the first promotion, so the per-environment Argo applications show as missing until then.

## Repository layout

```
apps/                      Argo CD Applications, one per environment
  kargo.yaml               Application that installs the Kargo project below
  {dev,qa,prod}/
kargo/                     Kargo project, warehouse and promotion stages
manifests/payments-api/    Kustomize base and overlays
```

## Boundaries

The `payments-project` AppProject (defined in [platform-gitops](https://github.com/AaronStahley/platform-gitops)) limits this domain to `payments-*` namespaces and to this repository. Namespaces are created by the platform team, not here.

## Related repositories

- [platform-gitops](https://github.com/AaronStahley/platform-gitops): platform services, AppProjects and namespaces
- [platform-bootstrap](https://github.com/AaronStahley/platform-bootstrap): cluster and Argo CD setup
- [webapi-operator](https://github.com/AaronStahley/webapi-operator): the operator behind `WebApi`
