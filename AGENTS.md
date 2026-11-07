# Working in this repo

This is Fredrik's homeops repo: GitOps for the Grove Kubernetes cluster (Talos,
ArgoCD, Gateway API via Traefik). This is a learning project just as much as it
is a doing project: optimise for understanding, explain the why, treat every
phase as a lesson as well as a change.

Cluster facts, what is built and what is next live in STATUS.md.

## How to work here

- Three obvious roles: decide what's next, write the code, run the code.
- Fredrik plans and decides the next task.
- The agent writes the manifests and commits at each logical step.
- Fredrik pushes and runs every cluster-facing command. The agent never pushes
  unless Fredrik says so for a given stretch of work.

### Detailed instructions for the agent

- Hand over exact commands with exact expected outcomes. "OutOfSync, then Synced
  within about a minute" is precise. "Synced" right after a push is wrong.
- One phase at a time. Explain what the phase does and why, then hand over the
  commands. Never dump all phases at once.
- No scope creep. Deliver the phase that was asked for. Tempting adjacent work
  gets suggested as follow-up work.
- Commits also land between agent steps. Start every session with `git status`
  and `git log`. Read a file before changing it. Prefer targeted edits over
  whole-file rewrites; a heredoc overwrite once dropped a patch from a
  kustomization.
- Manifest header comments explain the why, not the what. They are read to
  learn, so keep them up. Commit subjects are imperative and name the version:
  "Install Traefik 42.3.0 as the Gateway API implementation".
- Run `uvx --with mdformat-gfm mdformat --wrap 80` on any Markdown file you
  edit. Plain `mdformat` does not know tables and flattens them into one
  paragraph.

## Repo layout

```
bootstrap/   the root Application and two ApplicationSets; applied by hand once
infra/       platform components, one directory each, discovered by the infra set
  argocd/    upstream install.yaml pinned by version, ConfigMap patches, UI routes
apps/        workloads, one directory each, discovered by the apps set
scripts/     refresh-argocd
```

- Every directory under `infra/` and `apps/` is a kustomization and becomes
  exactly one ArgoCD Application named after the directory. The ApplicationSets
  in `bootstrap/` stamp them out from one template, so there is nothing to
  register: adding a component is creating a directory. `infra/argocd` is
  generated like the rest; ArgoCD manages its own install through it. Every
  generated Application carries the label `homeops.fandango.cloud/tree` set to
  `infra` or `apps`, so the two trees can be told apart in the UI and with
  `kubectl -l`.
- The only hand-written Application is `bootstrap`, which syncs `bootstrap/`
  including itself. It is hand-written so that deleting or renaming an
  ApplicationSet (which garbage-collects every Application it generated) heals
  on the next sync instead of needing a manual bootstrap.
- A directory owns everything about its component: its Namespace manifest when
  it owns a namespace, its HTTPRoutes, its chart and values. One directory per
  namespace; `porkbun-webhook` lives in cert-manager's namespace and ships no
  Namespace.
- Helm charts are rendered by kustomize's `helmCharts` field, never by a `helm`
  source in an Application. Pin `version`, set `releaseName` and `namespace` in
  the entry, keep values in `values.yaml` beside the kustomization, and add
  `includeCRDs: true` when the chart ships CRDs in `crds/`. Rendered output can
  be patched with ordinary kustomize patches.
- Never set the top-level `namespace:` in a Helm directory. It rewrites every
  resource, including the ones a chart places in `kube-system` on purpose, and
  ArgoCD would prune the originals. Reserve it for directories where every
  resource belongs in that namespace, like `infra/argocd`.
- In prose, things under `infra/` are "components" and things under `apps/` are
  "apps". Both are Applications in ArgoCD's vocabulary.
- A fresh cluster is bootstrapped with two commands and nothing after:
  `kubectl --context admin@grove apply --server-side -k infra/argocd`, then
  `kubectl --context admin@grove apply --server-side -k bootstrap`.

## Rules for the cluster

- Read-only kubectl from the agent is fine and encouraged.
- Pass `--context admin@grove` on every kubectl call so a different default
  context is never hit by accident.
- Every cluster change goes into this repo and through ArgoCD. Never
  `kubectl apply`, with the one exception of the two bootstrap commands above on
  a fresh cluster. Direct kubectl is for reading, `rollout restart` after a
  ConfigMap change, and `annotate` for an ArgoCD refresh.
- Pin every version: chart `version` in `helmCharts` and upstream manifest URLs.
- Verify any kustomization locally with `kubectl kustomize --enable-helm <dir>`
  before committing it. The local render passes no cluster version to Helm; add
  `--helm-kube-version` and `--helm-api-versions` for a faithful render of a
  chart that gates on `.Capabilities`. The diff in the ArgoCD UI is the
  authority.
- Kustomize pulls charts into `charts/` beside the kustomization and silently
  reuses whatever is there, version checked or not. `charts/` is gitignored;
  delete the local copy after a chart version bump.
- New components: create the directory with a kustomization. Sync policy comes
  from the ApplicationSet template (automated prune and selfHeal,
  ServerSideApply, ServerSideDiff, retry); do not add per-component overrides.
  Copy `infra/porkbun-webhook/` for a Helm chart, `infra/metallb/` for an
  upstream manifest.
- Cross-component ordering has no sync waves. A component that needs another
  one's CRDs fails its first syncs and succeeds on retry once they exist. Keep a
  CRD and its custom resources in one directory when you can; gitops-engine then
  needs no retry at all. `cert-manager-config` is separate from `cert-manager`
  because it also waits for the Porkbun webhook.
- Write Gateway API HTTPRoutes fully explicit (`group`, `kind`, `weight`, the
  default `PathPrefix /` match) and pin them with `sectionName`. ArgoCD reports
  drift on bare ones, and a route without `sectionName` attaches to every
  listener. `infra/argocd/httproute.yaml` and `httproute-redirect.yaml` are
  worked examples; every component carries its own.
- Secrets are never committed. Until a secrets tool exists, Fredrik creates them
  by hand. Never ask him to paste a credential into the chat. Document every
  hand-made Secret (name, namespace, keys) in the header comment of the
  kustomization that needs it, as `infra/porkbun-webhook/kustomization.yaml`
  does.
- Removing a directory removes its Application and orphans its resources
  (`preserveResourcesOnDeletion`). Cleaning up is deliberate and manual, and
  includes cluster-scoped resources: CRDs, ClusterRoles, APIServices,
  ClusterIssuers.

## Gotchas that have bitten

- ArgoCD syncing a ConfigMap restarts nothing. argocd-server and the
  application-controller read `argocd-cmd-params-cm` into env only at pod start.
  Check the key is live in the cluster before any `rollout restart`. Keys in
  `argocd-cm` such as `kustomize.buildOptions` are read live; no restart.
- Retry exhaustion is terminal. The template retries ten times with backoff
  capped at five minutes, about half an hour in all. After that ArgoCD logs
  "Skipping auto-sync: failed previous sync attempt" and the Application stays
  Failed until a new commit lands or someone syncs by hand. A hard refresh does
  not help.
- A new directory takes up to three minutes to become an Application: the git
  directory generator re-lists the repo on its own schedule, and a hard refresh
  of any Application does nothing for it. `scripts/refresh-argocd` annotates
  both sets, which the controller acts on within a second.
- Push-to-sync lag is the 60s poll plus up to 3 minutes of repo-server revision
  cache. Not a fault. `scripts/refresh-argocd [app ...]` skips the wait.
- Converting a chart from a `helm` source to `helmCharts` is only a no-op when
  `releaseName` equals the old Application name and `namespace` matches: the
  chart derives resource names from the release name, and Traefik's Deployment
  selector carries `instance: <release>-<namespace>`, which is immutable.
  Compare the local render's resource list against what ArgoCD manages before
  pushing; a server-side `kubectl diff` of the render with a scratch field
  manager is read-only and shows any real difference.
- Helm hooks carry over through kustomize. cert-manager's `startupapicheck` Job
  runs as a PostSync hook on every sync operation and deletes itself.
- The app-of-apps sync waves on the old `app-*.yaml` files only ever ordered the
  creation of child Applications, never their readiness: ArgoCD has no health
  check for `Application`. The retry block replaced them.
