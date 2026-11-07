# Working in this repo

This is Fredrik's homeops repo: GitOps for the Grove Kubernetes cluster
(Talos, ArgoCD, Gateway API via Traefik). Read this before doing anything.

## How Fredrik works with agents

- Fredrik is learning by doing. The agent is the driver: it writes the
  manifests and commits at each logical step. Fredrik pushes and runs the
  cluster-facing commands (kubectl checks, rollout restarts, anything
  interesting to watch). The agent never pushes.
- Hand him the exact commands and the exact expected outcome. Be precise
  about states. "OutOfSync, then Synced within about a minute" is good.
  Promising "Synced" right after a push is wrong.
- Work one phase at a time. Explain what the phase does and why before
  handing over the commands. Do not dump all phases at once.
- He dislikes scope creep. Deliver the phase that was asked for and add
  tempting adjacent work to the Follow-ups section below instead of doing it.
- He also commits between agent steps. Start a session with `git status` and
  `git log`, always read a file before changing it, and prefer targeted
  edits over whole-file rewrites. A heredoc overwrite once dropped a patch
  from a kustomization.
- Manifests carry header comments explaining the why, not the what. Keep
  that up; he reads them to learn. Commit subjects are imperative and name
  the version, e.g. "Install Traefik 42.3.0 as the Gateway API
  implementation".
- Commit messages end with
  `Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>`.

## Cluster facts (verified 2026-11-07)

- Talos v1.11.3, Kubernetes v1.34.1. Nodes `talos-control-{1..3}` and
  `talos-worker-{1..6}`. CNI is flannel. No StorageClass yet.
- Kube context is `admin@grove`. Pass `--context admin@grove` on every
  kubectl call so a different default context is never hit by accident.
- MetalLB L2 pool 10.9.8.60-69. Traefik's LoadBalancer Service is pinned
  to 10.9.8.60.
- The one Gateway is `traefik-gateway` in namespace `traefik` (the chart
  does not name it `traefik`). Listeners: `web` on 80 and `websecure` on
  443 for `*.fandango.cloud`, terminating TLS with Secret
  `fandango-cloud-tls` in the traefik namespace. cert-manager issues that
  wildcard from Let's Encrypt production via Porkbun DNS-01.
- Public DNS for `*.fandango.cloud` is at Porkbun. The ArgoCD UI is
  https://argocd.fandango.cloud.

## Repo layout

Everything lives under `infra/`. `infra/argocd/` is both the ArgoCD install
(upstream install.yaml pinned by version in the URL, plus ConfigMap patches)
and the app-of-apps: one `app-<name>.yaml` Application per component,
listed in `infra/argocd/kustomization.yaml`. ArgoCD manages itself through
the `argocd` Application.

Components come in two shapes:

- Plain manifests: `infra/<name>/` with a kustomization, usually an
  upstream install URL pinned to a version plus our own resources
  (gateway-api, metallb, cert-manager-config).
- Helm charts: no directory. The Application points at the chart repo with
  `targetRevision` pinned and values inline under `helm.valuesObject`
  (traefik, cert-manager, porkbun-webhook).

Sync waves in use: 1 gateway-api, 2 metallb, 3 traefik, 4 cert-manager,
5 porkbun-webhook, 6 cert-manager-config. A new component goes after the
wave that provides the CRDs it needs.

`scripts/refresh-argocd` hard-refreshes the `argocd` Application after a
push so Fredrik does not have to wait out the poll and cache.

## Rules for the cluster

- Every cluster change goes into this repo and through ArgoCD. Never
  `kubectl apply`. Direct kubectl is for reading, `rollout restart` after a
  ConfigMap change, and `annotate` for an ArgoCD refresh.
- Read-only kubectl from the agent is fine and encouraged.
- Pin every version: chart `targetRevision` and upstream manifest URLs.
- Verify any kustomization locally with `kubectl kustomize <dir>` before
  committing it.
- New components: add `infra/argocd/app-<name>.yaml`, list it in
  `infra/argocd/kustomization.yaml`, pick a sync wave, and copy the
  `syncPolicy` block from an existing Application (automated prune and
  selfHeal, ServerSideApply, ServerSideDiff, CreateNamespace when the
  chart does not create its namespace). The repetition is a known
  follow-up; do not fix it in passing.
- Write Gateway API HTTPRoutes fully explicit (`group`, `kind`, `weight`,
  the default `PathPrefix /` match) and pin them with `sectionName`. ArgoCD
  reports drift on bare ones, and a route without `sectionName` attaches to
  every listener. `infra/argocd/httproute.yaml` is the template;
  `httproute-redirect.yaml` is the matching HTTP to HTTPS redirect.
- Secrets are never committed. Until a secrets tool exists, Fredrik creates
  them by hand. Never ask him to paste a credential into the chat. Document
  every hand-made Secret (name, namespace, keys) in the header comment of
  the Application that needs it, as `app-porkbun-webhook.yaml` does.

## Gotchas that have bitten

- ArgoCD syncing a ConfigMap restarts nothing. argocd-server and the
  application-controller read their keys into env only at pod start. Check
  the key is live in the cluster before any `rollout restart`.
- A sync wave blocks on health of everything in earlier waves. A resource
  that can never become Healthy stalls the operation forever. Terminate,
  fix, push.
- Push-to-sync lag is the 60s poll plus up to 3 minutes of repo-server
  revision cache. Not a fault. `scripts/refresh-argocd` skips the wait for
  the app-of-apps. For any other Application:
  `kubectl -n argocd annotate application <app> argocd.argoproj.io/refresh=hard --overwrite`.
- `argocd --core` needs the kubeconfig context namespace set to `argocd`.
  Prefer plain kubectl.
- cert-manager v1.21 chart: `config.enableGatewayAPI` is deprecated, use
  `config.gatewayAPI.enabled`.
- The Traefik chart refuses to render an HTTPS listener without
  `certificateRefs`, so the certificate Secret must exist before the
  `websecure` listener goes in.

## Follow-ups

Noted, not built. Propose one as its own phase when it fits; never bolt it
onto another phase.

- Deduplicate the per-Application `syncOptions`: ServerSideDiff globally
  via `controller.diff.server.side` in argocd-cmd-params-cm, ServerSideApply
  via a kustomize JSON6902 patch on `kind: Application` or an
  ApplicationSet over `infra/*`.
- One gateway-wide HTTP to HTTPS redirect route instead of a redirect route
  per app.
- A secrets tool (SOPS or External Secrets) so Secrets stop being hand-made.
- Split-horizon DNS for `*.fandango.cloud`: LAN clients resolve to
  10.9.8.60, external clients to the public IP.
- A StorageClass. Cilium as CNI and Istio are on the roadmap but
  deliberately deferred.

## Handover prompts

When a phase is paused mid-way, write what a fresh agent needs into
`.agents/prompts/<phase>.md` (it may assume this file has been read) and
delete it in the commit that lands the phase. None exist right now.
