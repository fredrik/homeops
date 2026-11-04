# Working in this repo

This is Fredrik's homeops repo: GitOps for the Grove Kubernetes cluster
(Talos, ArgoCD). Read this before doing anything.

## How Fredrik works with agents

- Fredrik is learning by doing. The agent is the driver: it writes the
  manifests and commits at each logical step. Fredrik pushes and runs the
  cluster-facing commands (kubectl checks, rollout restarts, anything
  interesting to watch). Hand him the exact commands and the exact expected
  outcome.
- Work one phase at a time. Walk him through what the phase does and why
  before handing over the commands. Do not dump all phases at once.
- Be precise about expected states. "OutOfSync, then Synced within about a
  minute" is good. Promising "Synced" right after a push is wrong.
- He dislikes scope creep. Deliver the phase that was asked for and note
  tempting adjacent work as a follow-up instead of doing it.
- He makes his own commits between agent steps. Always read a file before
  rewriting it, and prefer targeted edits over whole-file rewrites.
- Commit messages end with
  `Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>`.

## Rules for the cluster

- Every cluster change goes into this repo and through ArgoCD. Never
  `kubectl apply`. Direct kubectl is for reading, `rollout restart` after a
  ConfigMap change, and the occasional `annotate` for an ArgoCD refresh.
- Read-only kubectl from the agent is fine and encouraged. Kube context is
  `admin@grove`.
- Verify any kustomization locally with `kubectl kustomize <dir>` before
  committing it.
- New components are onboarded by adding `infra/argocd/app-<name>.yaml` and
  listing it in `infra/argocd/kustomization.yaml`. Use sync waves to order
  components that depend on each other's CRDs.
- Write Gateway API HTTPRoutes fully explicit (`group`, `kind`, `weight`, the
  default `PathPrefix /` match). ArgoCD reports drift on bare ones.
  `infra/argocd/httproute.yaml` is the template.
- Secrets are never committed. Until a secrets tool exists, Fredrik creates
  them by hand. Never ask him to paste a credential into the chat.

## Gotchas that have bitten

- ArgoCD syncing a ConfigMap restarts nothing. argocd-server and the
  application-controller read their keys into env only at pod start. Check the
  key is live in the cluster before any `rollout restart`.
- A sync wave blocks on health of everything in earlier waves. A resource that
  can never become Healthy stalls the operation forever. Terminate, fix, push.
- Push-to-sync lag is the 60s poll plus up to 3 minutes of repo-server
  revision cache. Not a fault.
- `argocd --core` needs the kubeconfig context namespace set to `argocd`.
  Hard refresh without the CLI:
  `kubectl -n argocd annotate application <app> argocd.argoproj.io/refresh=hard`.

## Handover prompts

Phase-specific context for a fresh agent lives in `.agents/prompts/`. Those
files assume this document has been read.
