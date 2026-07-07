# moathsalman.com — Static Site via Cloudflare Tunnel

## What this is

A static personal site (`moathsalman.com`) hosted on the `lemoncluster` Kubernetes
cluster (Harvester/HCI + FluxCD GitOps), made publicly reachable through a
**Cloudflare Tunnel** rather than through the cluster's normal ingress path.

## Why a tunnel, not the existing ingress

`lemoncluster`'s Istio ingress gateway sits on a private LAN IP
(`192.168.0.79`) — everything behind it is only reachable on the LAN/VPN.
The cluster's existing DNS/TLS automation is also intentionally scoped away
from this domain:

- `external-dns` is `pdns`-only, filtered to `devkuban.com` — it will not
  touch `moathsalman.com`.
- `cert-manager`'s only issuer is a Cloudflare DNS-01 issuer scoped to
  `lem.devkuban.com` — it cannot issue certs for this domain either.

Rather than widening either of those (which would affect other apps on the
cluster), a Cloudflare Tunnel gives `moathsalman.com` a fully independent,
public path to a Service, with **zero inbound firewall/port-forwarding
changes** on the home network. The tunnel is strictly outbound: a
`cloudflared` pod in-cluster dials out to Cloudflare's edge and registers
itself; Cloudflare then routes public HTTPS traffic for the domain back over
that existing outbound connection. Nothing external ever initiates a
connection into the network.

## Architecture

```
Browser (HTTPS)
      │
      ▼
Cloudflare Edge  ── Universal SSL cert, auto-issued, no action needed
      │  (tunnel connection — encrypted, outbound-initiated by cloudflared)
      ▼
cloudflared Deployment (2 replicas, moathsalman namespace)
      │  (plain HTTP, internal-only — never leaves the cluster network)
      ▼
nginx Service (ClusterIP) → nginx Deployment (nginxinc/nginx-unprivileged)
      │
      ▼
ConfigMap (index.html)
```

### Where TLS actually terminates

- **Browser ↔ Cloudflare edge**: real, browser-trusted TLS (Cloudflare
  Universal SSL, auto-issued once the zone is active).
- **Cloudflare edge ↔ cloudflared**: encrypted by the tunnel protocol itself
  (QUIC/HTTP2), authenticated via the tunnel token — independent of whatever
  protocol is used on the last leg.
- **cloudflared ↔ nginx Service**: plain HTTP, but this traffic never
  leaves the cluster's internal pod network, so no origin cert is needed.
  This is why `cert-manager` and `external-dns` are correctly **not**
  involved anywhere in this setup.

## Prerequisites

- A Flux GitOps cluster with the standard folder pattern:
  `infrastructures/lemoncluster/<app>/` + registration in
  `clusters/lemoncluster/infrastructures.yaml`.
- `sealed-secrets` controller already installed and reachable
  (controller name `sealed-secrets-controller`, namespace `sealed-secrets`).
- Local `kubeseal` CLI matching the controller's version.
- A domain you can move to Cloudflare's nameservers (registrar stays
  wherever it already is — only the nameservers change).

## Setup steps

### 1. Cloudflare (one-time, dashboard)

1. Add the domain as a zone in Cloudflare (Free plan is sufficient).
2. Update the domain's **registrar-level** nameservers (not the hosted
   zone's own NS records) to the two nameservers Cloudflare assigns.
   Wait for the zone to show **Active** (propagation: minutes to ~24h).
3. Under **SSL/TLS → Overview**, set encryption mode to **Full** (not
   *Full (strict)* — the origin has no cert to verify, and not *Flexible*
   — no reason to leave the edge-to-origin hop unencrypted-by-default).
4. Under **SSL/TLS → Edge Certificates**, enable **Always Use HTTPS**.
5. Go to **Zero Trust → Networks → Tunnels → Create a tunnel** (connector
   type: Cloudflared). Copy the generated **token** — do not run any of
   the install commands shown (those are for bare-metal/VM installs; this
   tunnel runs as a container instead).
6. Hold off on the tunnel's **public hostname** config until the workload
   is deployed and verified (step 3 below) — this avoids configuring a
   route to a Service that doesn't exist yet.

### 2. Manifests

All under `infrastructures/lemoncluster/moathsalman/`:

| File | Purpose |
|---|---|
| `namespace.yaml` | Dedicated namespace, isolates RBAC/NetworkPolicy/secrets scope |
| `configmap.yaml` | Holds `index.html` — simplest possible static-site delivery |
| `nginx-deployment.yaml` | `nginxinc/nginx-unprivileged` (non-root, port 8080), mounts the ConfigMap read-only |
| `nginx-service.yaml` | ClusterIP, port 80 → targetPort 8080. **Name must match exactly** what's configured as the tunnel's public-hostname target |
| `cloudflared-sealedsecret.yaml` | Tunnel token, sealed — never committed in plaintext |
| `cloudflared-deployment.yaml` | `cloudflare/cloudflared`, 2 replicas, `tunnel --no-autoupdate run`, token via `env.valueFrom.secretKeyRef`, no Service (outbound-only) |
| `networkpolicy.yaml` | Restricts cloudflared's **egress** to only DNS (kube-system, port 53) + the nginx Service (port 8080) |
| `kustomization.yaml` | Lists all of the above for Flux/Kustomize |

**Why `nginx-unprivileged`, not stock `nginx`:** stock nginx binds port 80,
which needs root inside the container. The unprivileged image listens on
8080 and runs non-root out of the box — simpler and safer.

**Why the NetworkPolicy matters:** cloudflared has open outbound internet
access by default. Without this policy, a compromised token/image could
let it pivot to *any* other pod on the cluster. Restricting egress to just
DNS + the one Service caps the blast radius to "can reach one internal
static-site Service," regardless of what happens to the tunnel itself.
Note the policy only restricts *cluster-internal* egress — the outbound
connection to Cloudflare's edge is a separate concern, governed by normal
node/cluster network routing, not by this NetworkPolicy.

**Sealing the token:**

```bash
kubectl create secret generic cloudflared-tunnel-token \
  --namespace=moathsalman \
  --from-literal=token='<TUNNEL_TOKEN>' \
  --dry-run=client -o yaml > /tmp/raw-secret.yaml

kubeseal --controller-name sealed-secrets-controller \
  --controller-namespace sealed-secrets \
  --format yaml \
  < /tmp/raw-secret.yaml > cloudflared-sealedsecret.yaml

rm /tmp/raw-secret.yaml
```

The intermediate raw secret file is never committed — only the encrypted
`SealedSecret` output is. SealedSecrets are name+namespace scoped by
default: this sealed blob can only decrypt into a Secret named
`cloudflared-tunnel-token` in the `moathsalman` namespace.

### 3. Flux wiring

Add a `Kustomization` to `clusters/lemoncluster/infrastructures.yaml`:

```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: moathsalman
  namespace: flux-system
spec:
  interval: 10m
  retryInterval: 1m
  timeout: 5m
  path: ./infrastructures/lemoncluster/moathsalman
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  dependsOn:
    - name: sealed-secrets
  wait: true
```

`dependsOn: [sealed-secrets]` ensures Flux won't attempt to apply this
Kustomization until the sealed-secrets controller itself reports Ready —
necessary since `cloudflared-deployment.yaml` references a Secret that
only exists once the controller decrypts the SealedSecret.

Commit and push. Flux reconciles automatically (or force it:
`flux reconcile kustomization moathsalman -n flux-system --with-source`).

### 4. Complete the tunnel's public hostname

Once the pods are confirmed running (`kubectl get all -n moathsalman`) and
the SealedSecret has decrypted (`kubectl get secret cloudflared-tunnel-token
-n moathsalman`), go back to **Zero Trust → Networks → Tunnels → [tunnel] →
Public Hostname** and add:

- **Hostname:** `moathsalman.com`
- **Service type:** HTTP
- **URL:** `nginx.moathsalman.svc.cluster.local:80`

Cloudflare auto-creates the corresponding DNS record — no manual A/CNAME
record needed. The Service name/namespace here must match the manifests
exactly, or requests will 502.

## Verification checklist

```bash
kubectl get all -n moathsalman                              # nginx 1/1, cloudflared 2/2, Service present
kubectl get secret cloudflared-tunnel-token -n moathsalman   # confirms SealedSecret decrypted
kubectl logs -n moathsalman deploy/cloudflared --tail=30     # look for a successful tunnel registration
```

Then load `https://moathsalman.com` in a browser and confirm it serves
over HTTPS.

## Issues encountered during setup

**Stale Bitnami Helm repository URL blocked `sealed-secrets` reconciliation.**
The cluster's `sealed-secrets` `HelmRepository` pointed at
`https://bitnami-labs.github.io/sealed-secrets`, which Bitnami migrated to
`https://bitnami.github.io/sealed-secrets` (org rename from `bitnami-labs`
to `bitnami`). The old URL now 404s, which left the `sealed-secrets`
Flux `Kustomization` permanently stuck in `HealthCheckFailed` — unrelated
to this project, but it directly blocked the `dependsOn` chain above and
had to be fixed first:

```bash
# infrastructures/lemoncluster/sealed-secrets/repository.yaml
- url: https://bitnami-labs.github.io/sealed-secrets
+ url: https://bitnami.github.io/sealed-secrets
```

## Follow-ups / hardening

- Rotate the tunnel token periodically (regenerate in Cloudflare, re-seal,
  redeploy) as routine hygiene.
- Consider pinning `cloudflared` and `nginx-unprivileged` to specific
  image tags instead of `latest`/`stable` for reproducibility.
- Bump `nginx` to 2 replicas if zero-downtime rolling updates become a
  priority (currently 1, which is fine for a low-traffic static site).