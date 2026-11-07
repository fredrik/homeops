# Repo layout: infra and apps, discovered by ApplicationSets

Written 2026-11-07. Amended the same day after two independent reviews, both
checked against the live cluster, the three chart tarballs, the ArgoCD docs and
the v3.5.7 source. The amendments are summarised at the end. Status: agreed, not
started.

## Why

Today every component is wired up by hand: a directory under `infra/`, an
`infra/argocd/app-<name>.yaml` Application, and a line in
`infra/argocd/kustomization.yaml`. Helm-based components have no directory at
all; their values sit inline in the Application. Three things are wrong with
that as the repo grows:

- `infra/` will stop being an accurate name the moment the first workload lands.
  A media server is not infrastructure.
- `infra/argocd/` is three things at once: the ArgoCD install, the app-of-apps,
  and the UI's routes. It should be about ArgoCD only.
- Components do not all have the same shape. A kustomize component is a
  directory; a Helm component is a block inside an Application. Anything that
  wants to treat components uniformly (discovery, patching, a template) cannot.

Fredrik enjoys kustomize and does not enjoy Helm. The layout should let him
touch a kustomization and a values file and nothing else.

## Target layout

```
infra/                      what the cluster needs before it can run anything
  argocd/                   install, ConfigMap patches, UI HTTPRoutes, the
                            hand-written root Application, and the two
                            ApplicationSets below
  gateway-api/
  metallb/
  traefik/                  kustomization with helmCharts + values.yaml
  cert-manager/             kustomization with helmCharts + values.yaml
  porkbun-webhook/          kustomization with helmCharts + values.yaml
  cert-manager-config/      ClusterIssuers and the wildcard Certificate
apps/                       what we actually run; one directory per app
docs/plans/                 plans like this one
scripts/
.gitignore                  charts/ pulled by local kustomize builds
```

Rules the layout implies:

- Every directory under `infra/` and `apps/` is a kustomization and becomes
  exactly one ArgoCD Application, named after the directory. The one exception
  is `infra/argocd`, whose Application is hand-written; see "Two trees, two
  ApplicationSets" for why.
- A directory owns everything about its component: its Namespace manifest when
  it owns a namespace, its HTTPRoutes, its chart and values. Nothing about a
  component lives anywhere else.
- Charts are rendered by kustomize's `helmCharts` field, never by a `helm`
  source in an Application. Chart version is pinned in the kustomization, values
  live in `values.yaml` beside it.
- Single cluster, so no `clusters/<name>/` layer. Add one only when a second
  cluster exists.
- In prose, things under `infra/` are "components" and things under `apps/` are
  "apps". Both are Applications in ArgoCD's vocabulary.

## Decisions and why

### Two trees, two ApplicationSets

`infra/argocd/` gets two ApplicationSets, each with a git directory generator:
one over `infra/*`, one over `apps/*`. Each stamps out an Application from a
single template. Adding a component becomes "create a directory". The
`syncPolicy` repetition across the `app-*.yaml` files goes away because the
template is written once.

Two sets rather than one so the trees can later differ: separate ArgoCD
projects, different sync behaviour, a different branch for apps. Not needed
today, cheap to have from the start. One set with two `path` globs would be the
drier version; splitting later is a five-minute change.

The root Application `argocd` stays hand-written in `application.yaml` and the
infra generator excludes its directory:

```yaml
directories:
  - path: infra/*
  - path: infra/argocd
    exclude: true
```

The reason is garbage collection. A generated Application carries an
ownerReference to its ApplicationSet, and `preserveResourcesOnDeletion` keeps
the resources, not the Applications. If the infra set were ever pruned by a bad
commit, renamed (a rename is a delete plus a create) or deleted by hand,
Kubernetes would delete every generated Application, including the one that
manages ArgoCD. Nothing would then apply the fix commit, and the cluster would
need a manual bootstrap again. With the root Application outside the set, the
same mistake heals itself on the next push, because the root is still alive and
recreates the set. ArgoCD keeps managing itself exactly as it does now.

A fresh cluster is bootstrapped with
`kubectl apply --server-side -k infra/argocd`, run twice: the first pass
installs ArgoCD and fails on the ApplicationSets because their CRD is not
established yet, the second pass creates them. Nothing is applied by hand after
that.

### Ordering by retry, not sync wave

The price of ApplicationSets: sync waves between components stop working. A wave
orders resources inside one sync operation. The current waves work because the
app-of-apps syncs the child Applications as its own resources. ApplicationSet
children are created by a controller, not synced by a parent, so there is no
operation for a wave to order.

The loss is smaller than it looks. ArgoCD has no built-in health check for
`Application` resources, so today's waves only order the creation of the child
Applications and never wait for one to be ready. The children already converge
by retrying. The STATUS.md known issue about child Application health closes
with this migration, because nothing waits on a child any more.

Ordering comes from retries. The template sets `syncPolicy.retry`. A component
whose CRDs do not exist yet fails its sync, backs off, and succeeds on a later
attempt once the component that ships the CRDs is in. On a fresh bootstrap the
UI is red for a while and then goes green. On the running cluster nothing
changes because everything already exists.

Retry exhaustion is terminal, not cosmetic. Auto-sync refuses to retry a
revision whose last attempt failed; the controller logs "Skipping auto-sync:
failed previous sync attempt ... and will not retry". Once the budget runs out
the Application stays Failed until a new commit lands or someone syncs by hand.
A hard refresh does not help. The budget therefore has to cover the slowest
dependency chain on a fresh bootstrap:

```yaml
retry:
  limit: 10
  backoff:
    duration: 10s
    factor: 2
    maxDuration: 5m
```

That is roughly half an hour of attempts. A negative `limit` means unlimited,
which the docs allow; a finite one is kinder to the API server and the recovery
step is a manual sync or any new commit. This becomes the retry gotcha that
replaces the sync wave gotcha in AGENTS.md.

Waves still work inside one directory, and a CRD and its custom resource in the
same directory need no retry at all: gitops-engine skips the dry run for kinds
whose CRD is in the same sync. The retry is only for cross-directory
dependencies. That is why `cert-manager-config` stays its own directory rather
than being merged into `cert-manager`: the ClusterIssuers must wait for both
cert-manager's CRDs and the Porkbun webhook, and those are two other
directories.

### Helm through kustomize `helmCharts`

ArgoCD already renders Helm sources with `helm template`; it never runs
`helm install`. Kustomize's `helmCharts` field does the same thing, so nothing
is lost on the rendering side and three things are gained:

- Every component has the same shape, which the directory generator needs.
- Values live in a file instead of nested six levels deep in an Application.
- The rendered chart output can be patched with ordinary kustomize patches. That
  is the tool we wanted when the Traefik chart would not render the HTTPS
  listener our way.

The ArgoCD-native alternative, a multi-source Application with the chart from
its repo and values from this one, keeps the Helm tab in the UI and needs no
global flag, but it gives no post-render patching and a directory generator
could not stamp it out. For a kustomize-first repo `helmCharts` is the right
choice.

What it costs, and what to get exactly right:

- `argocd-cm` needs `kustomize.buildOptions: --enable-helm`. Without it the
  repo-server refuses to render a kustomization with `helmCharts`. The key is
  read by the application-controller, which sends it to the repo-server with
  every manifest request, so if a restart is ever needed it is of the
  application-controller, not the repo-server.
- `releaseName` must be set explicitly to the current Application name, because
  ArgoCD's native Helm source names the release after the Application and
  resource names derive from the release name. For `traefik` and `cert-manager`
  that equals the chart name anyway. For `porkbun-webhook` it does not: the
  chart is `cert-manager-webhook-porkbun`, and its resources are named
  `porkbun-webhook-cert-manager-webhook-porkbun` today. Pinning
  `releaseName: porkbun-webhook` keeps every resource name the same so the
  migration is a no-op diff.
- `namespace` must be set in the `helmCharts` entry, because that is what
  `.Release.Namespace` renders from. Traefik gives one more reason to get both
  exact: its `app.kubernetes.io/instance` label is the release name joined with
  the namespace (`traefik-traefik` today) and sits in the Deployment selector. A
  wrong value fails with "field is immutable" instead of silently replacing
  Traefik.
- `includeCRDs: true` where the chart ships CRDs in its `crds/` directory.
  Traefik does, and its Application manages 25 CRDs today; dropping the flag
  would prune every one of them, including the kind of the `default` TLSOption.
  cert-manager renders CRDs from templates behind `crds.enabled`, so the flag is
  irrelevant there.
- Helm hooks carry over. ArgoCD maps `helm.sh/hook`, `helm.sh/hook-weight` and
  `helm.sh/hook-delete-policy` on any rendered resource whatever the source
  type, so cert-manager's `startupapicheck` Job keeps running as a PostSync hook
  and cleaning itself up exactly as it does today. Disabling it is optional
  tidiness, not a requirement.
- ArgoCD passes the cluster's Kubernetes version and API versions to kustomize
  Helm renders by default (`--helm-kube-version` and `--helm-api-versions`), the
  same values the native Helm source gets, so a chart that gates templates on
  `.Capabilities` renders the same either way. A local
  `kubectl kustomize --enable-helm` passes neither unless given the same flags,
  so a local render can differ from the repo-server's. The diff in the UI is the
  authority.
- Kustomize pulls the chart into `charts/` beside the kustomization and silently
  reuses whatever is there without checking the version. A committed copy would
  pin a stale chart, so `.gitignore` lists `charts/` and the local copy is
  deleted after a version bump. The repo-server is unaffected: it cleans its
  checkout on every revision.
- A `helm` binary is needed locally for `kubectl kustomize --enable-helm`. Helm
  4.3.0 is installed on Fredrik's machine; the repo-server runs Helm 4.2.1 and
  kustomize 5.8.1, which kubectl 1.37.1 also bundles.

Helm is still underneath. Fredrik only ever touches a kustomization and a values
file.

### Namespaces: owned by one directory, set per chart

Each namespace is owned by exactly one directory, which ships its Namespace
manifest. `traefik` and `cert-manager` ship theirs. `porkbun-webhook` lives in
cert-manager's namespace and ships none, because two Applications managing one
resource would flip the tracking annotation on every sync. `metallb` gets its
Namespace from the upstream manifest. `gateway-api` and `cert-manager-config`
own no namespace. `CreateNamespace=true` disappears from the sync options
because no Application relies on it.

The top-level `namespace:` field in a kustomization is a transformer that
rewrites the namespace of every resource, including ones a chart places
elsewhere on purpose. Verified on the live cluster, ArgoCD tracks these in
`kube-system`: cert-manager's leader-election Roles and RoleBindings for the
controller and the cainjector, and the Porkbun webhook's RoleBinding to
`extension-apiserver-authentication-reader`. With `namespace: cert-manager` at
the top level they would render into `cert-manager` and ArgoCD would prune the
`kube-system` originals: cert-manager loses its leader election, the webhook
loses the right to read the API server's request-header CA. So:

- Never set the top-level `namespace:` on a Helm directory. Set `releaseName`
  and `namespace` inside the `helmCharts` entry instead.
- Reserve the top-level `namespace:` for directories where every resource
  belongs in that namespace. Today that is only `infra/argocd`, because the
  upstream install manifest carries no namespaces at all.
- For a future chart that leaves some resources without a namespace, use a
  `NamespaceTransformer` with `unsetOnly: true`, which fills in only the missing
  ones.

All three current charts write a namespace on every namespaced resource, so
nothing relies on the Application's `destination.namespace`. The template sets
it to the directory name, which is cosmetic: a resource that lands there is a
bug that fails loudly with "namespace not found".

A Helm directory then looks like this:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
# No top-level namespace: it would rewrite the kube-system RoleBinding this
# chart creates on purpose. The chart namespaces every resource itself.
helmCharts:
  - name: cert-manager-webhook-porkbun
    repo: https://porkbun.octabits.io
    version: 2.0.8
    # ArgoCD's native Helm source named the release after the Application.
    # Keeping that name keeps every resource name, so the migration is a
    # no-op diff.
    releaseName: porkbun-webhook
    namespace: cert-manager
    valuesFile: values.yaml
```

### The UI's HTTPRoutes stay in `infra/argocd`

The route that exposes the ArgoCD UI is ArgoCD's concern, the same way a
component owns its Namespace. Every future app carries its own `httproute.yaml`.
The AGENTS.md wording that calls the argocd one "the template" is what goes; the
file stays.

### Deletion safety

The ApplicationSet controller adds the resources finalizer to the Applications
it generates. Deleting a generated Application, or the ApplicationSet itself,
would then cascade into deleting the component's resources. Set
`syncPolicy.preserveResourcesOnDeletion: true` on both ApplicationSets so
removing a directory removes the Application and orphans the resources. Together
with the root Application living outside the set, no single commit can take
ArgoCD down.

Cleaning up an orphaned component is a deliberate step, and it is more than
`kubectl delete namespace`. Every component here has cluster-scoped resources
too: CRDs, ClusterRoles, the webhook's APIService, the GatewayClass, the
ClusterIssuers. Those are listed and deleted by hand, which is the right amount
of friction for a destructive step.

The current `app-*.yaml` Applications carry no finalizer and no owner (checked
2026-11-07). Pruning them during the migration orphans their resources, and the
ApplicationSet-generated Application of the same name adopts them on its first
sync, because the tracking annotation embeds only the Application name and the
names already equal the directory names.

### Discovery lag

The git directory generator re-lists the repo every three minutes by default,
and a hard refresh of the root Application does nothing for a new directory. A
new component therefore takes up to three minutes plus the repo-server cache to
appear as an Application. Annotating the sets triggers a reconcile and bypasses
the revision cache:

```
kubectl --context admin@grove -n argocd annotate applicationset infra apps \
  argocd.argoproj.io/application-set-refresh=true --overwrite
```

`scripts/refresh-argocd` gains this line. It is documented for webhooks, so it
is tested once during phase 3 before the script relies on it.

### The infra ApplicationSet

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: infra
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  syncPolicy:
    # No resources finalizer on generated Applications: removing a directory
    # removes the Application and orphans its resources.
    preserveResourcesOnDeletion: true
  generators:
    - git:
        repoURL: https://github.com/fredrik/homeops.git
        revision: main
        directories:
          - path: infra/*
          # The root Application lives here, hand-written, so that deleting
          # this set can never delete the thing that recreates it.
          - path: infra/argocd
            exclude: true
  template:
    metadata:
      name: "{{ .path.basename }}"
    spec:
      project: default
      source:
        repoURL: https://github.com/fredrik/homeops.git
        targetRevision: main
        path: "{{ .path.path }}"
      destination:
        server: https://kubernetes.default.svc
        # Only used for resources that carry no namespace. Every directory
        # namespaces its own resources, so a resource landing here is a bug
        # that fails loudly with "namespace not found".
        namespace: "{{ .path.basename }}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - ServerSideApply=true
          - ServerSideDiff=true
        retry:
          limit: 10
          backoff:
            duration: 10s
            factor: 2
            maxDuration: 5m
```

`goTemplate: true` with `missingkey=error` is what the docs recommend; the
legacy template engine is slated for deprecation. The apps set is the same with
`apps/*` and no exclusion.

## Migration phases

One push per phase unless stated. Each phase leaves the cluster Synced and
Healthy before the next starts. Exact commands and expected states are handed
over per phase when it is run, as usual.

- **Phase 0. Prerequisites.** Confirm `helm version` works locally (it does as
  of 2026-11-07). Add `.gitignore` with `charts/`. No cluster change.

- **Phase 1. Enable Helm in kustomize.** Add
  `kustomize.buildOptions: --enable-helm` to `argocd-cm-patch.yaml` and verify
  the key is live in the ConfigMap. This phase cannot be verified on its own;
  the first `helmCharts` render in phase 2 verifies it. Restart the
  application-controller only if that render fails with an `--enable-helm` error
  after a hard refresh. No component changes.

- **Phase 2. Convert the Helm components, one at a time, two pushes each.**
  Order: `porkbun-webhook` (smallest, and the one with the release name trap),
  then `cert-manager`, then `traefik`. For each: create the directory with
  kustomization, `values.yaml` and, for `traefik` and `cert-manager`, the
  Namespace manifest. Verify locally with `kubectl kustomize --enable-helm` and
  compare resource names and namespaces against what ArgoCD manages today.

  First push: switch the existing Application from a `helm` source to a `path`
  source, drop `CreateNamespace=true` where present, and remove the `automated`
  block in the same commit. Automated sync would apply a wrong render seconds
  after the push, and a wrong release name creates new Deployments and Services
  and prunes the old ones, so the first push must be observable. Expected result
  of a correct conversion: stays Synced, the revision column flips from the
  chart version to a git SHA, no OutOfSync at all. Anything else shows as a diff
  in the UI to be read and fixed before anyone presses Sync. The usual causes of
  a diff are `releaseName`, `namespace` in the `helmCharts` entry, a missing
  `includeCRDs`, or a `.Capabilities` difference between the local and the
  server render.

  Second push: restore `automated`.

- **Phase 3. Swap the app-of-apps for the ApplicationSets.** In one commit: add
  `appset-infra.yaml` and `appset-apps.yaml` to `infra/argocd`, remove the six
  `app-*.yaml`, keep `application.yaml`, and update the kustomization. Create
  `apps/.gitkeep` so the tree exists. Expected: the root Application prunes the
  six child Applications (resources are orphaned, not deleted, because no
  Application carries a finalizer) and creates the two ApplicationSets; the sets
  create six Applications with the same names; each adopts its existing
  resources on the first sync and goes Synced. Within one sync gitops-engine
  runs every prune before any create, so there is a window of a minute or so
  where the six Applications do not exist. The root Application never deletes
  itself, so the cluster is never without a manager. Applications are deleted
  and recreated during the swap. Resources are not. Test the ApplicationSet
  refresh annotation here.

- **Phase 4. Rewrite the docs and the script.** AGENTS.md: replace the Repo
  layout section and the New components rule, rewrite the sync wave gotcha as a
  retry gotcha including the exhaustion-is-terminal recovery step, add the
  discovery lag gotcha, drop the "template" wording about the HTTPRoute.
  STATUS.md: components table without waves, note the ApplicationSets, close the
  child-health known issue, remove the syncOptions repetition known issue.
  `scripts/refresh-argocd`: add the ApplicationSet refresh.

The earlier plan had a separate phase for giving every directory its Namespace.
With the corrected namespace rule only `traefik` and `cert-manager` ship one,
and both are created in phase 2, so that phase folded into it.

## Out of scope

Secrets tooling, storage, observability, a `clusters/` layer, ArgoCD projects
per tree, progressive syncs (beta, behind a controller flag). None of these are
needed for the layout and each is its own piece of work.

Follow-up made possible by the layout: `crds.keep: true` in the cert-manager
values is a Helm-ism ArgoCD ignores. Rendering through kustomize allows the
ArgoCD equivalent, a patch adding `argocd.argoproj.io/sync-options: Prune=false`
to the CRDs.

## Amendments after review, 2026-11-07

Two reviews, collated into the text above. Where they disagreed, the
disagreement was settled against the ArgoCD v3.5.7 source.

Design changes:

- Root Application stays hand-written and outside the infra set, with
  `exclude: true` on `infra/argocd`. This also removes the need for the
  `PruneLast=true` step one review proposed: the root never deletes itself, so
  prune-before-create no longer threatens it.
- No top-level `namespace:` on Helm directories. `releaseName` and `namespace`
  go in the `helmCharts` entry. One directory per namespace; `porkbun-webhook`
  ships no Namespace.
- Each Helm conversion is two pushes, the first with `automated` removed, so a
  wrong render is a diff in the UI rather than an applied change.
- Retry budget is finite and sized for a fresh bootstrap. Retry exhaustion is
  terminal; the recovery is a manual sync or a new commit.
- `goTemplate: true` with `missingkey=error`.
- The namespace phase folded into phase 2.

Text corrections:

- Helm hooks carry over through kustomize; the plan said they were lost.
- `kustomize.buildOptions` is read by the application-controller, not the
  repo-server, so that is the restart target if one is needed.
- ArgoCD passes the cluster's Kubernetes and API versions to kustomize Helm
  renders. One review said it did not; `util/kustomize/kustomize.go` at v3.5.7
  adds `--helm-kube-version` and `--helm-api-versions`, and the repo-server
  fills them from the cluster unless the source overrides them. Only a local
  render lacks them.
- The swap phase said nothing is deleted during the swap. Applications are;
  resources are not.
- Expected phase 2 outcome was "OutOfSync, then Synced"; that describes an
  imperfect conversion.
- Orphan cleanup includes cluster-scoped resources, not just a namespace.
- Bootstrap is server-side apply, run twice.
- Discovery lag of the directory generator is a new gotcha, and
  `scripts/refresh-argocd` gains the ApplicationSet refresh.
- Both reviews said no `helm` binary was installed; Helm 4.3.0 is present as of
  this amendment.

Open choices, each with the default this plan takes:

- Root Application and sets in `infra/argocd` (taken) or in a separate
  `bootstrap/` directory with `infra/argocd` generated like any other component.
  The latter costs a second kind of apply on a fresh cluster.
- Two ApplicationSets (taken) or one with two globs.
- Finite retry limit (taken) or unlimited.
- `preserveResourcesOnDeletion` on the apps set too (taken, and right for
  anything with a PVC) or cascade so that removing an app's directory removes
  its namespace.
