# Repo layout: infra and apps, discovered by ApplicationSets

Written 2026-11-07. Status: agreed, not started.

## Why

Today every component is wired up by hand: a directory under `infra/`, an
`infra/argocd/app-<name>.yaml` Application, and a line in
`infra/argocd/kustomization.yaml`. Helm-based components have no directory
at all; their values sit inline in the Application. Three things are wrong
with that as the repo grows:

- `infra/` will stop being an accurate name the moment the first workload
  lands. A media server is not infrastructure.
- `infra/argocd/` is three things at once: the ArgoCD install, the
  app-of-apps, and the UI's routes. It should be about ArgoCD only.
- Components do not all have the same shape. A kustomize component is a
  directory; a Helm component is a block inside an Application. Anything
  that wants to treat components uniformly (discovery, patching, a
  template) cannot.

Fredrik enjoys kustomize and does not enjoy Helm. The layout should let
him touch a kustomization and a values file and nothing else.

## Target layout

```
infra/                      what the cluster needs before it can run anything
  argocd/                   install, ConfigMap patches, UI HTTPRoutes,
                            and the two ApplicationSets below
  gateway-api/
  metallb/
  traefik/                  kustomization with helmCharts + values.yaml
  cert-manager/             kustomization with helmCharts + values.yaml
  porkbun-webhook/          kustomization with helmCharts + values.yaml
  cert-manager-config/      ClusterIssuers and the wildcard Certificate
apps/                       what we actually run; one directory per app
docs/plans/                 plans like this one
scripts/
```

Rules the layout implies:

- Every directory under `infra/` and `apps/` is a kustomization and becomes
  exactly one ArgoCD Application, named after the directory.
- A directory owns everything about its component: its Namespace manifest,
  `namespace:` in the kustomization, its HTTPRoutes, its chart and values.
  Nothing about a component lives anywhere else.
- Charts are rendered by kustomize's `helmCharts` field, never by a `helm`
  source in an Application. Chart version is pinned in the kustomization,
  values live in `values.yaml` beside it.
- Single cluster, so no `clusters/<name>/` layer. Add one only when a
  second cluster exists.
- In prose, things under `infra/` are "components" and things under
  `apps/` are "apps". Both are Applications in ArgoCD's vocabulary.

## Decisions and why

### Two trees, two ApplicationSets

`infra/argocd/` gets two ApplicationSets, each with a git directory
generator: one over `infra/*`, one over `apps/*`. Each stamps out an
Application from a single template. Adding a component becomes "create a
directory". The `syncPolicy` repetition across the `app-*.yaml` files goes
away because the template is written once.

Two sets rather than one so the trees can later differ: separate ArgoCD
projects, different sync behaviour, a different branch for apps. Not
needed today, cheap to have from the start.

The `argocd` directory is itself picked up by the infra ApplicationSet, so
ArgoCD keeps managing itself the same way it does now. A fresh cluster is
bootstrapped once with `kubectl apply -k infra/argocd`, which installs
ArgoCD and the ApplicationSets, and nothing is applied by hand after that.

### Ordering by retry, not sync wave

The price of ApplicationSets: sync waves between components stop working.
A wave orders resources inside one sync operation. The current waves work
because the app-of-apps syncs the child Applications as its own resources.
ApplicationSet children are created by a controller, not synced by a
parent, so there is no operation for a wave to order.

Ordering comes from retries instead. The template sets
`syncPolicy.retry` with a generous limit and backoff. A component whose
CRDs do not exist yet fails its sync, backs off, and succeeds on a later
attempt once the component that ships the CRDs is in. On a fresh
bootstrap the UI is red for a few minutes and then goes green. On the
running cluster nothing changes because everything already exists.

Waves still work inside one directory. That is why `cert-manager-config`
stays its own directory rather than being merged into `cert-manager`: the
ClusterIssuers must wait for both cert-manager's CRDs and the Porkbun
webhook, and those are two other directories.

### Helm through kustomize `helmCharts`

ArgoCD already renders Helm sources with `helm template`; it never runs
`helm install`. Kustomize's `helmCharts` field does the same thing, so
nothing is lost on the rendering side and three things are gained:

- Every component has the same shape, which the directory generator needs.
- Values live in a file instead of nested six levels deep in an
  Application.
- The rendered chart output can be patched with ordinary kustomize
  patches. That is the tool we wanted when the Traefik chart would not
  render the HTTPS listener our way.

What it costs:

- `argocd-cm` needs `kustomize.buildOptions: --enable-helm`. Without it
  the repo-server refuses to render a kustomization with `helmCharts`.
- Helm hooks are not translated into ArgoCD hooks. A hook resource is
  rendered as a plain resource and applied once. cert-manager's
  `startupapicheck` is such a hook; disable it in values.
- `releaseName` must be set explicitly to the current Application name,
  because ArgoCD's native Helm source names the release after the
  Application and resource names derive from the release name. For
  `traefik` and `cert-manager` that equals the chart name anyway. For
  `porkbun-webhook` it does not: the chart is
  `cert-manager-webhook-porkbun`. Pinning `releaseName: porkbun-webhook`
  keeps every resource name the same so the migration is a no-op diff.
- `namespace` must be set in the `helmCharts` entry as well as in the
  kustomization, so templates that read `.Release.Namespace` render the
  right value.
- `includeCRDs: true` where the chart ships CRDs in its `crds/` directory
  (Traefik does; cert-manager renders CRDs from templates behind
  `crds.enabled`).

Helm is still underneath. Fredrik only ever touches a kustomization and a
values file.

### Every directory owns its Namespace

Each component ships a Namespace manifest and sets `namespace:` in its
kustomization, as `infra/argocd` already does. The ApplicationSet
template then needs no per-directory knowledge, `CreateNamespace=true`
disappears from the sync options, and `destination.namespace` in the
template can be the directory name because nothing relies on it.

### The UI's HTTPRoutes stay in `infra/argocd`

The route that exposes the ArgoCD UI is ArgoCD's concern, the same way a
component owns its Namespace. Every future app carries its own
`httproute.yaml`. The AGENTS.md wording that calls the argocd one "the
template" is what goes; the file stays.

### Deletion safety

The ApplicationSet controller adds the resources finalizer to the
Applications it generates. Deleting a generated Application, or the
ApplicationSet itself, would then cascade into deleting the component's
resources. With the `argocd` Application in that set, a mistake there
would delete ArgoCD. Set `syncPolicy.preserveResourcesOnDeletion: true`
on both ApplicationSets so removing a directory removes the Application
and orphans the resources. Cleaning up an orphaned component is then a
deliberate `kubectl delete namespace`, which is the right amount of
friction for a destructive step.

The current `app-*.yaml` Applications carry no finalizer (checked
2026-11-07). Pruning them during the migration orphans their resources,
and the ApplicationSet-generated Application of the same name adopts them
on its first sync.

## Migration phases

One push per phase. Each phase leaves the cluster Synced and Healthy
before the next starts. Exact commands and expected states are handed
over per phase when it is run, as usual.

1. **Enable Helm in kustomize.** Add `kustomize.buildOptions:
   --enable-helm` to `argocd-cm-patch.yaml`. Verify the key is live in
   the ConfigMap; restart `argocd-repo-server` if a `helmCharts`
   kustomization still fails to render afterwards. No component changes.

2. **Convert the Helm components, one at a time.** Order:
   `porkbun-webhook` (smallest, and the one with the release name trap),
   then `cert-manager`, then `traefik`. For each: create the directory
   with kustomization, `values.yaml` and Namespace manifest, verify with
   `kubectl kustomize --enable-helm`, and switch its existing Application
   from a `helm` source to a `path` source. Expected result per
   component: OutOfSync on the source change, then Synced with an empty
   or near-empty diff. A large diff means `releaseName` or `namespace` is
   wrong; stop and fix before syncing.

3. **Give every directory its Namespace.** Add Namespace manifests and
   `namespace:` to `gateway-api`, `metallb` and `cert-manager-config`
   where they apply, and drop `CreateNamespace=true` from the
   Applications that use it. Namespaces already exist, so the diff is the
   new Namespace resource coming under management.

4. **Swap the app-of-apps for the ApplicationSets.** In one commit: add
   `appset-infra.yaml` and `appset-apps.yaml` to `infra/argocd`, remove
   every `app-*.yaml` and `application.yaml`, and update the kustomization.
   Create `apps/.gitkeep` so the tree exists. Expected: the `argocd`
   Application prunes the old Applications and creates the
   ApplicationSets; the ApplicationSets create Applications with the
   same names; each adopts its existing resources and goes Synced. The
   `argocd` Application is briefly deleted and recreated by its own
   ApplicationSet, which is the same self-management loop as today. This
   is the one phase with a window where nothing manages the cluster.
   Nothing is deleted during it.

5. **Rewrite the docs.** AGENTS.md: replace the Repo layout section and
   the New components rule, rewrite the sync wave gotcha as a retry
   gotcha, drop the "template" wording about the HTTPRoute. STATUS.md:
   components table without waves, note the ApplicationSets.
   `scripts/refresh-argocd` keeps working unchanged.

## Out of scope

Secrets tooling, storage, observability, a `clusters/` layer, ArgoCD
projects per tree, progressive syncs. None of these are needed for the
layout and each is its own piece of work.
