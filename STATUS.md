# Project status

Where the Grove cluster and this repo stand, what is built, and what the
candidate next steps are. Updated 2026-11-07. Verified against the live cluster
with read-only kubectl on that date.

## In one paragraph

Grove is a nine-node Talos cluster managed entirely from this repo through
ArgoCD. The platform layer is in place: ArgoCD manages itself, MetalLB hands out
LAN addresses, Traefik implements the Gateway API, and cert-manager issues a
production Let's Encrypt wildcard for `*.fandango.cloud` through Porkbun DNS-01.
The ArgoCD UI is served over HTTPS at https://argocd.fandango.cloud with HTTP
redirected. All seven Applications are Synced and Healthy at the current main.
Nothing else runs on the cluster yet: no storage, no secrets tooling, no
observability, no workloads. The next phase is not yet chosen.

## What exists

### Cluster

| | | |---|---| | Talos | v1.11.3 | | Kubernetes | v1.34.1 | | Nodes |
talos-control-{1..3}, talos-worker-{1..6} | | CNI | flannel | | Storage | none,
no StorageClass | | kube context | admin@grove |

### Platform components, in sync-wave order

| Wave | Component | Version | Shape | Namespace | |---|---|---|---|---| | - |
ArgoCD | v3.5.7 | upstream install.yaml + ConfigMap patches, self-managed |
argocd | | 1 | Gateway API CRDs | v1.6.3 standard | upstream manifest | - | | 2
| MetalLB | v0.16.2 | upstream manifest + L2 pool 10.9.8.60-69 | metallb-system
| | 3 | Traefik | chart 42.3.0 (Traefik v3.7.17) | Helm, inline values | traefik
| | 4 | cert-manager | chart v1.21.3 | Helm, inline values | cert-manager | | 5
| Porkbun DNS-01 webhook | chart 2.0.8 | Helm, inline values | cert-manager | |
6 | cert-manager config | - | ClusterIssuers staging and prod, wildcard
Certificate | traefik |

### Networking and TLS

- Gateway `traefik-gateway` in namespace `traefik` (the chart does not name it
  `traefik`), pinned to 10.9.8.60. Listeners `web` (80) and `websecure` (443)
  for `*.fandango.cloud`.
- TLS terminates at the Gateway with Secret `fandango-cloud-tls`, issued by
  ClusterIssuer `letsencrypt-prod`. Certificate is Ready. The
  `letsencrypt-staging` issuer exists for experiments.
- Strict SNI is set on Traefik's default TLSOption.
- One hostname routed: argocd.fandango.cloud, with an HTTPRoute on `websecure`
  and a 301 redirect route on `web`. Both routes live in `infra/argocd/` and are
  the templates for new ones.
- Public DNS for `*.fandango.cloud` is at Porkbun and resolves to the LAN
  address, so the UI is reachable only from the LAN. Stale records from the
  retired AKS cluster have been cleaned out of the zone.

### Hand-made objects not in Git

- Secret `porkbun-api-credentials` in namespace `cert-manager`, the Porkbun API
  key pair used by the DNS-01 webhook. Documented in
  `infra/argocd/app-porkbun-webhook.yaml`.

### Repo conventions

Described in AGENTS.md: every change through ArgoCD, every version pinned,
explicit HTTPRoutes pinned with `sectionName`, header comments explaining the
why, Fredrik pushes and runs cluster commands. `scripts/refresh-argocd`
hard-refreshes the app-of-apps after a push.

## What has been done, in order

1. **ArgoCD bootstrap** (2026-11-04). Installed from the upstream manifest with
   server-side apply, then adopted by its own Application. Automated sync with
   prune and selfHeal, git polled every minute.
1. **Gateway API, MetalLB, Traefik** (2026-11-04 and 05). Flannel kept; Cilium
   deferred on purpose. ArgoCD UI exposed over plain HTTP via HTTPRoute.
   ServerSideDiff enabled on all Applications after ArgoCD reported drift on
   defaulted HTTPRoute fields.
1. **Strict SNI** on the Traefik default TLSOption.
1. **cert-manager and TLS** (2026-11-07, four phases). cert-manager with Gateway
   API support, Porkbun DNS-01 webhook, Let's Encrypt issuers and the wildcard
   Certificate (staging first, then production), the `websecure` listener, and
   HTTPS routing with HTTP redirect.
1. **Housekeeping.** AGENTS.md rewritten with verified cluster facts, repo
   layout, gotchas and follow-ups.

## Known issues and open decisions

- **Child Application health is not checked.** ArgoCD has no built-in health
  check for `argoproj.io/Application`, so the sync waves on the child
  Applications only order their creation, not their readiness. The documented
  Lua health customization in argocd-cm would fix this.
- **`argocd-initial-admin-secret` still exists** in the argocd namespace. The
  admin password was changed at bootstrap, so the Secret is dead weight and
  should be deleted.
- **Everything is in the default AppProject** with wildcard source, destination
  and cluster-resource permissions. A dedicated infra project is the usual
  practice.
- **No Application carries the resources finalizer**, so removing an app from
  Git orphans its resources instead of deleting them. Deliberate choice or
  oversight, undecided.
- **Strict SNI is unverified against the Gateway.** Traefik's Gateway provider
  should pick up the default TLSOption, but the docs do not say so explicitly.
  Check that a handshake to 10.9.8.60:443 without SNI fails while one with
  `argocd.fandango.cloud` succeeds.
- **Planning format is undecided.** One proposal is a root ROADMAP.md (Now /
  Next / Later / Done) with a Planning rule in AGENTS.md replacing the
  handover-prompt convention. AGENTS.md still describes handover prompts. No
  ROADMAP.md has been written.
- **Repetition in `infra/argocd/app-*.yaml`.** The syncPolicy block is copied
  per Application. Known, deliberately left for a dedicated phase.

## What is possibly next

Nothing is committed to. The candidates, roughly in the order they unblock each
other:

1. **Secrets tooling** (SOPS with ksops, Sealed Secrets, or External Secrets
   Operator). The Porkbun credential is the only hand-made Secret today, but
   every credentialed component after this will add one.
1. **Storage.** A StorageClass is the prerequisite for anything stateful.
   Candidates range from local-path-provisioner as a first step to Proxmox CSI,
   Longhorn or Rook, plus the snapshot controller.
1. **Observability.** kube-prometheus-stack and Loki.
1. **Backups.** Velero or Volsync, once there is state to back up.
1. **Repo hygiene.** Renovate for version bumps, CI running kustomize build and
   kubeconform, the syncOptions dedupe, a dedicated AppProject, the Application
   health check above, a GitHub webhook to argocd-server to replace the poll.
1. **Networking refinements.** A gateway-wide HTTP to HTTPS redirect instead of
   one per hostname (needs an `infra/traefik` folder). Split-horizon DNS so LAN
   clients resolve to 10.9.8.60 and external clients to the public address.
1. **Larger rebuilds, deliberately later.** Cilium replacing flannel (would make
   MetalLB redundant), Istio ambient as a second GatewayClass, Talos upgrade
   controller, Kyverno, a pull-through registry cache, OIDC identity, and then
   actual workloads.

A contingency: the Porkbun cert-manager webhook is a small third-party project.
If it goes stale, the healthier alternative is the DNSimple webhook, which would
mean moving the fandango.cloud nameservers.
