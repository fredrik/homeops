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
- Fredrik pushes and runs every cluster-facing command. The agent never pushes.

### Detailed instructions for the agent

- Hand over exact commands with exact expected outcomes. "OutOfSync, then Synced
  within about a minute" is precise. "Synced" right after a push is wrong.
- One phase at a time. Explain what the phase does and why, then hand over the
  commands. Never dump all phases at once.
- No scope creep. Deliver the phase that was asked for. Tempting adjacent work
  goes in the Follow-ups section below.
- Commits also land between agent steps. Start every session with `git status`
  and `git log`. Read a file before changing it. Prefer targeted edits over
  whole-file rewrites; a heredoc overwrite once dropped a patch from a
  kustomization.
- Manifest header comments explain the why, not the what. They are read to
  learn, so keep them up. Commit subjects are imperative and name the version:
  "Install Traefik 42.3.0 as the Gateway API implementation".
- Run `uvx mdformat --wrap 80` on any Markdown file you edit.

## Repo layout

Everything lives under `infra/`. `infra/argocd/` is both the ArgoCD install
(upstream install.yaml pinned by version in the URL, plus ConfigMap patches) and
the app-of-apps: one `app-<name>.yaml` Application per component, listed in
`infra/argocd/kustomization.yaml`. ArgoCD manages itself through the `argocd`
Application.

Components come in two shapes:

- Plain manifests: `infra/<name>/` with a kustomization, usually an upstream
  install URL pinned to a version plus our own resources (gateway-api, metallb,
  cert-manager-config).
- Helm charts: no directory. The Application points at the chart repo with
  `targetRevision` pinned and values inline under `helm.valuesObject` (traefik,
  cert-manager, porkbun-webhook).

`scripts/refresh-argocd` hard-refreshes the `argocd` Application after a push so
Fredrik does not have to wait out the poll and cache.

## Rules for the cluster

- Read-only kubectl from the agent is fine and encouraged.
- Pass `--context admin@grove` on every kubectl call so a different default
  context is never hit by accident.
- Every cluster change goes into this repo and through ArgoCD. Never
  `kubectl apply`. Direct kubectl is for reading, `rollout restart` after a
  ConfigMap change, and `annotate` for an ArgoCD refresh.
- Pin every version: chart `targetRevision` and upstream manifest URLs.
- Verify any kustomization locally with `kubectl kustomize <dir>` before
  committing it.
- New components: add `infra/argocd/app-<name>.yaml`, list it in
  `infra/argocd/kustomization.yaml`, pick a sync wave, and copy the `syncPolicy`
  block from an existing Application (automated prune and selfHeal,
  ServerSideApply, ServerSideDiff, CreateNamespace when the chart does not
  create its namespace). The repetition is a known follow-up; do not fix it in
  passing.
- Write Gateway API HTTPRoutes fully explicit (`group`, `kind`, `weight`, the
  default `PathPrefix /` match) and pin them with `sectionName`. ArgoCD reports
  drift on bare ones, and a route without `sectionName` attaches to every
  listener. `infra/argocd/httproute.yaml` is the template;
  `httproute-redirect.yaml` is the matching HTTP to HTTPS redirect.
- Secrets are never committed. Until a secrets tool exists, Fredrik creates them
  by hand. Never ask him to paste a credential into the chat. Document every
  hand-made Secret (name, namespace, keys) in the header comment of the
  Application that needs it, as `app-porkbun-webhook.yaml` does.

## Gotchas that have bitten

- ArgoCD syncing a ConfigMap restarts nothing. argocd-server and the
  application-controller read their keys into env only at pod start. Check the
  key is live in the cluster before any `rollout restart`.
- A sync wave blocks on health of everything in earlier waves. A resource that
  can never become Healthy stalls the operation forever. Terminate, fix, push.
- Push-to-sync lag is the 60s poll plus up to 3 minutes of repo-server revision
  cache. Not a fault. `scripts/refresh-argocd` skips the wait for the
  app-of-apps. For any other Application:
  `kubectl -n argocd annotate application <app> argocd.argoproj.io/refresh=hard --overwrite`.
