# Handover: cert-manager and TLS on Grove

You are picking up the next phase of Fredrik's homeops work: install cert-manager
and put real TLS on the Traefik gateway for `*.fandango.cloud`. Everything below
was true on 2026-11-04 evening. Verify anything cheap to verify before relying on it.

## How to work with Fredrik

Read `AGENTS.md` at the repo root first. It covers working style, the
GitOps rules, and the gotchas. This file only adds what is specific to the
cert-manager phase.

## Cluster facts

- Talos v1.11.3, Kubernetes v1.34.1, 3 control planes + 6 workers on 10.9.8.21-36,
  flannel CNI, kube context `admin@grove`. No storage class.
- Talos enforces the `baseline` Pod Security level by default. MetalLB's
  namespace carries `privileged` labels from upstream. cert-manager runs as
  non-root and is fine under baseline.
- ArgoCD v3.5.7 self-managed from `infra/argocd/` (kustomize, upstream
  install.yaml pinned by URL). Application `argocd` has automated sync, prune,
  selfHeal, `ServerSideApply=true`, `ServerSideDiff=true`. It also contains the
  child Applications, so adding `app-<name>.yaml` to `infra/argocd/` and listing
  it in the kustomization is how a new component is onboarded (mini app-of-apps).
- Sync waves on the Applications: argocd 0, gateway-api 1, metallb 2, traefik 3.
  Use 4 for cert-manager and 5 for anything that needs cert-manager's CRDs.
- ArgoCD polls Git every 60s (argocd-cm `timeout.reconciliation`). Effective
  push-to-sync lag is up to that plus the repo-server's 3-minute revision cache.
- Gateway API v1.6.3 standard CRDs via `infra/gateway-api/`.
- MetalLB v0.16.2, L2 pool 10.9.8.60-69 via `infra/metallb/`.
- Traefik Helm chart 42.3.0 (Traefik v3.7.17) via `infra/argocd/app-traefik.yaml`
  (Helm source, values inline, no repo folder). Gateway API provider on, legacy
  Ingress provider off. The chart's Gateway is named `traefik-gateway` in
  namespace `traefik` (not `traefik`). One listener today: `web`, HTTP, port
  8000, `namespacePolicy.from: All`. Service pinned to 10.9.8.60 via
  `metallb.io/loadBalancerIPs`. The chart already exposes 443 -> 8443 on the
  Service; the `websecure` listener is commented out in chart defaults because
  it requires `certificateRefs`.
- ArgoCD UI is live at http://argocd.fandango.cloud via
  `infra/argocd/httproute.yaml`. argocd-server runs with `server.insecure: "true"`
  (patch in `infra/argocd/argocd-cmd-params-cm-patch.yaml`), so Traefik
  terminates client connections and forwards plain HTTP. Port-forward still works.

## DNS facts (checked with dig)

- `fandango.cloud` nameservers are Porkbun.
- `argocd.fandango.cloud` resolves publicly to 10.9.8.60, a private LAN IP.
  Let's Encrypt can never reach the gateway over HTTP, so HTTP-01 is out.
  DNS-01 is the only ACME path.
- The apex `fandango.cloud` and a public wildcard `*.fandango.cloud` already
  resolve to 89.167.82.199. Something else lives there. Do not touch those
  records. DNS-01 only needs `_acme-challenge` TXT records and is unaffected.
- Porkbun is not a built-in cert-manager DNS-01 provider.

## Decisions to make with Fredrik before writing anything

Ask these up front, with your recommendation first.

1. DNS-01 strategy for Porkbun. Options, roughly in order of effort:
   - Community webhook solver `mdonoughe/porkbun-webhook` (Helm chart, release
     0.1.5 from 2024-10-11, 45 stars, no activity since). Least work, but an
     unmaintained dependency in the ACME path. Needs a Porkbun API key + secret
     and API access enabled on the domain in Porkbun's UI.
   - CNAME delegation: `_acme-challenge.fandango.cloud CNAME` to a zone on a
     built-in provider such as Cloudflare, with `cnameStrategy: Follow` on the
     solver. Robust, no third-party webhook, but needs a Cloudflare zone he owns.
   - Move fandango.cloud nameservers to Cloudflare. Most robust, biggest change,
     and the existing apex/wildcard records must be recreated there.
   - Private CA via a cert-manager CA ClusterIssuer, no Let's Encrypt. Zero DNS
     work, but the CA must be trusted on every device. Valid for a homelab.
2. Wildcard `*.fandango.cloud` (one cert, one Secret in the traefik namespace,
   recommended) versus per-hostname certificates.
3. Let's Encrypt staging first, then flip to production once a staging cert is
   Ready. Recommend yes. Production has a 5 duplicate certs/week limit.
4. Secret handling. The DNS provider credential must never be committed. For
   now Fredrik creates the Secret by hand with kubectl in the cert-manager
   namespace. Never ask him to paste the key into the chat, and never print it.
   Note SOPS / External Secrets as a later follow-up, do not build it now.

## Proposed design (adjust to the decisions above)

Phase 1: cert-manager
- `infra/argocd/app-cert-manager.yaml`: Helm source `https://charts.jetstack.io`,
  chart `cert-manager`, version `v1.21.3` (latest as of 2026-10-23; v1.20.5 is
  the maintenance line). Namespace `cert-manager`, `CreateNamespace=true`,
  `ServerSideApply=true`, `ServerSideDiff=true`, sync-wave 4. Values:
  `crds.enabled: true` and `config.enableGatewayAPI: true` (the latter lets
  cert-manager see Gateway API objects; harmless even if you use explicit
  Certificate resources).
- Verify: three Deployments ready in `cert-manager`, CRDs present
  (`kubectl get crd | grep cert-manager.io`), Application Synced and Healthy.
  The first sync may sit in Progressing for a while, as gateway-api did, while
  ArgoCD rebuilds its API cache for the new CRDs.

Phase 2: issuers and the certificate
- New folder, for example `infra/cert-manager-config/`, with its own Application
  at sync-wave 5 (CRDs must exist before these are applied; same reason metallb's
  pool sits in a later wave). Contents: `ClusterIssuer letsencrypt-staging`,
  `ClusterIssuer letsencrypt-prod`, and a `Certificate` in namespace `traefik`
  with `secretName: fandango-cloud-tls`, `dnsNames: ["*.fandango.cloud"]`,
  issuerRef pointing at staging first. If a webhook solver is chosen, its Helm
  Application also goes here at wave 4 or 5 as appropriate.
- Verify: `kubectl -n traefik get certificate,certificaterequest,order,challenge`
  and `kubectl -n traefik describe challenge` while it is pending. Expected
  end state: Certificate Ready=True and Secret `fandango-cloud-tls` present with
  `tls.crt`/`tls.key`.

Phase 3: websecure listener on the Gateway
- In `infra/argocd/app-traefik.yaml` add `gateway.listeners.websecure` with
  port 8443, protocol HTTPS, hostname `*.fandango.cloud`,
  `namespacePolicy.from: All`, `certificateRefs: [{kind: Secret, name: fandango-cloud-tls}]`.
  The chart fails to render an HTTPS listener without certificateRefs, so do
  this only after the Secret exists.
- Verify: `kubectl -n traefik get gateway traefik-gateway -o yaml` shows the
  websecure listener with ResolvedRefs=True and Programmed=True.
- `curl -vk https://argocd.fandango.cloud/` shows a certificate for
  `*.fandango.cloud`. With staging the issuer says "(STAGING) Let's Encrypt" and
  browsers reject it; that is the expected intermediate state.

Phase 4: route over HTTPS and redirect HTTP
- Change `infra/argocd/httproute.yaml` to attach to the websecure listener
  (parentRef with `sectionName: websecure`). Add a second HTTPRoute attached to
  `sectionName: web` whose only rule is a `RequestRedirect` filter with
  `scheme: https`, `statusCode: 301`, no backendRefs.
- Keep every HTTPRoute fully explicit: `group`, `kind` on parentRefs and
  backendRefs, `weight: 1`, and the default `PathPrefix /` match. ArgoCD 3.5.7
  reports drift on HTTPRoutes that omit these, even with ServerSideDiff. The
  existing `infra/argocd/httproute.yaml` is the template.
- Flip the Certificate's issuerRef to `letsencrypt-prod` in its own commit.
- Verify: `curl -sI http://argocd.fandango.cloud/` is a 301 to https,
  `curl -sI https://argocd.fandango.cloud/` is 200 with no `-k`, browser is happy.

## Follow-ups already noted elsewhere, not for this phase

- Deduplicate the repeated syncOptions (global `controller.diff.server.side`
  for ServerSideDiff; a kustomize JSON6902 patch on all Applications or an
  ApplicationSet over `infra/*` for ServerSideApply).
- GitHub webhook to argocd-server so pushes sync in seconds.
- Secrets management (SOPS or External Secrets).
- Storage. Cilium. Istio as a second GatewayClass.
