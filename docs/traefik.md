# Traefik

Foundational layer for clean URLs and automatic HTTPS via Let's Encrypt: traffic flows MetalLB → Traefik (Gateway API provider) → `HTTPRoute`s, with TLS terminated once by a cert-manager wildcard certificate on the `Gateway`.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 2–3 hours |
| **Runs On** | k3s cluster |
| **Depends On** | Kubernetes (k3s + MetalLB) |

!!! note "Why the Gateway API, not IngressRoute"
    The cluster uses the Kubernetes **Gateway API**, not Traefik's proprietary `IngressRoute`. The Gateway API splits ownership: the platform owns the `Gateway` (listeners + TLS), each app owns its `HTTPRoute`. TLS is handled once by a cert-manager wildcard cert on the Gateway, so apps attach a route and configure no TLS at all. See [Deploying an App](apps-deploy-pattern.md) for the per-app side.

## Domain Strategy

Two options: local-only with self-signed certs, or a real domain (~$10/yr) with Let's Encrypt DNS-01. **Recommended: real domain** — annual cost is trivial and DNS-01 is a real-world skill.

## Step 1: Cloudflare API Token

Host the domain on Cloudflare, then create an API token under **My Profile → API Tokens** with the **"Edit zone DNS"** template, scoped to your zone:

- **Permissions:** Zone → DNS → Edit
- **Zone Resources:** Include → Specific zone → `yourdomain.com`

cert-manager uses this token to solve the DNS-01 challenge for the wildcard cert. Save it to your password manager; you'll seal it in Step 4.

## Step 2: Install the Gateway API CRDs

The Gateway API CRDs are a separate upstream project (`kubernetes-sigs/gateway-api`), not bundled with Traefik. Install them **first** — at ArgoCD sync wave `-2` — so they exist before Traefik tries to use the `GatewayClass`/`Gateway` kinds:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/standard-install.yaml
```

In GitOps this is an `Application` pointing at `config/crd/standard` of that repo, pinned to the release tag.

## Step 3: Install Traefik with the Gateway provider

```bash
helm repo add traefik https://traefik.github.io/charts
kubectl create namespace traefik   # skip if created during Kubernetes setup
```

`homelab-manifests/apps/traefik/values.yaml`:

```yaml
ports:
  web:
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true

service:
  type: LoadBalancer
  # Kubernetes deprecated spec.loadBalancerIP in 1.24+; pin the MetalLB IP via annotation.
  annotations:
    metallb.universe.tf/loadBalancerIPs: 10.0.20.200

providers:
  kubernetesGateway:
    enabled: true

gatewayClass:
  enabled: true
  name: traefik

# Disable the chart's default Gateway; we manage our own in manifests/gateway.yaml.
gateway:
  enabled: false
```

```bash
helm upgrade --install traefik traefik/traefik \
  --namespace traefik --values values.yaml
```

Note what's *gone* versus the legacy setup: no `certificatesresolvers` ACME arguments, no `acme.json` persistence, no `CF_DNS_API_TOKEN` env. Certificates are cert-manager's job now (Step 5), not Traefik's.

!!! note "Source of truth"
    This `values.yaml` lives at `homelab-manifests/apps/traefik/values.yaml` with a pinned chart version, applied by ArgoCD via the multi-source `$values` pattern. Edit the repo, not a local copy.

## Step 4: Seal the Cloudflare token

cert-manager reads the token from a Secret named `cloudflare-api-token` (key `token`) in the `cert-manager` namespace. Seal it so it's safe in Git:

```bash
kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=token=<YOUR_CF_TOKEN> \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > cf-token-sealed.yaml
```

Commit it under `homelab-manifests/infrastructure/cert-manager/manifests/`.

## Step 5: cert-manager + wildcard certificate

Install cert-manager, define two `ClusterIssuer`s (staging first to avoid burning Let's Encrypt rate limits, then prod), and issue one `*.yourdomain.com` wildcard `Certificate` into the `traefik` namespace.

```bash
helm upgrade --install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version vX.Y.Z \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true
```

Pin `--version` to a current release on [cert-manager.io](https://cert-manager.io/docs/installation/helm/).

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: cloudflare-staging
spec:
  acme:
    email: acme@yourdomain.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: cloudflare-staging-issuer-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: token
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: cloudflare-prod
spec:
  acme:
    email: acme@yourdomain.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: cloudflare-prod-issuer-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: token
```

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: yourdomain-wildcard
  namespace: traefik
spec:
  secretName: yourdomain-tls          # the Secret the Gateway references
  issuerRef:
    name: cloudflare-prod             # use cloudflare-staging while testing
    kind: ClusterIssuer
  dnsNames:
    - "*.yourdomain.com"
    - "yourdomain.com"
```

!!! tip "Verify against staging first"
    Point the Certificate's `issuerRef` at `cloudflare-staging` until you confirm the DNS-01 flow issues a cert, then switch to `cloudflare-prod` and let it re-issue. Prod Let's Encrypt caps duplicate certs at 5/week per registered domain.

## Step 6: The Gateway

The `Gateway` is where TLS lives — two listeners: HTTP (redirected to HTTPS by Traefik) and HTTPS, terminating TLS with the wildcard secret. `allowedRoutes.namespaces.from: All` lets any namespace attach a route.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: traefik
  namespace: traefik
spec:
  gatewayClassName: traefik
  listeners:
    - name: web
      protocol: HTTP
      port: 8000
      allowedRoutes:
        namespaces:
          from: All
    - name: websecure
      protocol: HTTPS
      port: 8443
      hostname: "*.yourdomain.com"
      tls:
        mode: Terminate
        certificateRefs:
          - name: yourdomain-tls
            namespace: traefik
      allowedRoutes:
        namespaces:
          from: All
```

## Step 7: Internal DNS

On your UDM, add a host record for `*.yourdomain.com → 10.0.20.200` to avoid hairpin NAT.

## Step 8: Sample HTTPRoute

Every service attaches like this — no TLS config, the Gateway handles it:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nextcloud
  namespace: nextcloud
spec:
  parentRefs:
    - name: traefik
      namespace: traefik
      sectionName: websecure
  hostnames:
    - nextcloud.yourdomain.com
  rules:
    - backendRefs:
        - name: nextcloud
          port: 80
```

Apps that need Authelia ForwardAuth attach a Traefik `Middleware` via an `ExtensionRef` filter — see [Deploying an App](apps-deploy-pattern.md).

!!! warning "Never expose the Traefik dashboard without auth"
    It leaks routing details that aid attackers.

## Verification

- [ ] Traefik LoadBalancer has the MetalLB IP:

    ```bash
    kubectl get svc -n traefik
    # Expected: traefik LoadBalancer ... 10.0.20.200 ...
    ```

- [ ] The GatewayClass is accepted and the Gateway is programmed:

    ```bash
    kubectl get gatewayclass traefik
    kubectl get gateway -n traefik traefik
    # Expected: PROGRAMMED=True
    ```

- [ ] The wildcard certificate is issued:

    ```bash
    kubectl get certificate -n traefik
    # Expected: READY=True for yourdomain-wildcard
    ```

- [ ] HTTPS works with a real Let's Encrypt cert (after attaching an HTTPRoute):

    ```bash
    curl -vI https://argocd.yourdomain.com 2>&1 | grep -E 'issuer|subject'
    # Expected: issuer references Let's Encrypt, not self-signed
    ```

- [ ] HTTP redirects to HTTPS (the `web` port's `http.redirections` in `values.yaml`).
