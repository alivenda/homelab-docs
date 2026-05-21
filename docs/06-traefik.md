# Runbook 6: Traefik Reverse Proxy with HTTPS

Foundational layer: clean URLs, automatic HTTPS via Let's Encrypt, and a single entry point for all services.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 1–2 hours |
| **Runs On** | k3s cluster |
| **Depends On** | Runbook 5 |

## Domain Strategy

Two options: local-only with self-signed certs, or a real domain (~$10/yr) with Let's Encrypt DNS-01. **Recommended: real domain** — annual cost is trivial and DNS-01 is a real-world skill.

## Step 1: Cloudflare API Token

Buy/host domain on Cloudflare, then create an API token under **My Profile → API Tokens** with the **"Edit zone DNS"** template, scoped to your domain only:

- **Permissions:** Zone → DNS → Edit
- **Zone Resources:** Include → Specific zone → `yourdomain.com`

Save the token to Vaultwarden. You'll also seal it as a Kubernetes secret in Step 3.

## Step 2: Install Traefik via Helm

```bash
helm repo add traefik https://traefik.github.io/charts
kubectl create namespace traefik   # skip if you created it in R5 Step 13
```

`values.yaml`:

```yaml
ports:
  web:
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true
  websecure:
    tls:
      enabled: true
service:
  type: LoadBalancer
  # Kubernetes deprecated spec.loadBalancerIP in 1.24+.
  # Pin the MetalLB IP via an annotation instead.
  annotations:
    metallb.universe.tf/loadBalancerIPs: 10.0.0.70
additionalArguments:
  - "--certificatesresolvers.cloudflare.acme.dnschallenge=true"
  - "--certificatesresolvers.cloudflare.acme.dnschallenge.provider=cloudflare"
  - "--certificatesresolvers.cloudflare.acme.email=<YOUR_EMAIL>"
  - "--certificatesresolvers.cloudflare.acme.storage=/data/acme.json"
persistence:
  enabled: true
  storageClass: nfs-storage
  size: 128Mi
env:
  - name: CF_DNS_API_TOKEN
    valueFrom:
      secretKeyRef:
        name: cloudflare-api-token
        key: token
```

## Step 3: Create the Cloudflare Token Secret and Install Traefik

You have two paths for the token Secret. Prefer the SealedSecret path so future cluster rebuilds reconcile from Git.

=== "SealedSecret (recommended)"
    If you sealed the token in [R5 Step 13](05-kubernetes.md#step-13-encrypt-your-first-secret), commit `cf-token-sealed.yaml` to `homelab-manifests/apps/traefik/`. ArgoCD applies it, the Sealed Secrets controller materializes the Secret, Traefik reads it.

    ```bash
    helm install traefik traefik/traefik \
      --namespace traefik --values values.yaml
    ```

=== "Bootstrap (imperative)"
    Works the first time but the Secret is not in Git, so ArgoCD can't reconcile it on a rebuild.

    ```bash
    kubectl create secret generic cloudflare-api-token \
      --from-literal=token=<YOUR_CF_TOKEN> \
      --namespace traefik

    helm install traefik traefik/traefik \
      --namespace traefik --values values.yaml
    ```

## Step 4: Internal DNS

On your UDM, add a host record for `*.yourdomain.com → 10.0.0.70` to avoid hairpin NAT.

## Step 5: Sample IngressRoute

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nextcloud
  namespace: nextcloud   # match the app's namespace (per R0 namespace strategy)
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`nextcloud.yourdomain.com`)
      kind: Rule
      services:
        - name: nextcloud
          port: 80
  tls:
    certResolver: cloudflare
    domains:
      - main: nextcloud.yourdomain.com
```

!!! warning
    Never expose the Traefik dashboard without basic auth. It exposes routing details that aid attackers.

!!! tip
    From this point forward, every service runbook adds an IngressRoute and skips NodePorts.

!!! tip "cert-manager alternative"
    Traefik's built-in ACME client (used above) is sufficient for this guide. If you later swap to a different ingress (`nginx-ingress`, Istio, etc.) or want a single cert lifecycle across multiple controllers, install [cert-manager](https://cert-manager.io) — the k8s-native standard for certificate management, works with the same Cloudflare DNS-01 you set up here:

    ```bash
    helm install cert-manager oci://quay.io/jetstack/charts/cert-manager \
      --namespace cert-manager --create-namespace \
      --set crds.enabled=true
    ```

## Verification

- [ ] Traefik LoadBalancer has the MetalLB IP:

    ```bash
    kubectl get svc -n traefik
    # Expected: traefik LoadBalancer ... 10.0.0.70 ...
    ```

- [ ] `acme.json` is being populated:

    ```bash
    kubectl exec -n traefik deploy/traefik -- ls -la /data/acme.json
    # Expected: file exists, non-zero size
    ```

- [ ] HTTPS works with a real Let's Encrypt cert (after adding an IngressRoute):

    ```bash
    curl -vI https://argocd.yourdomain.com 2>&1 | grep -E 'issuer|subject'
    # Expected: issuer references Let's Encrypt, not self-signed
    ```

- [ ] HTTP redirects to HTTPS automatically (the `web` entryPoint's `http.redirections.entryPoint` in `values.yaml`).
