# Runbook 20: Homepage

A self-hosted dashboard for all cluster services, with live widget integrations.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 45–60 minutes |
| **Runs On** | k3s (cluster) |
| **Depends On** | Runbook 6 (Traefik), Runbook 18 (Authelia — ForwardAuth) |

Homepage ([gethomepage.dev](https://gethomepage.dev)) is a highly customisable start page. In this stack it aggregates status widgets for every deployed service, is protected by Authelia ForwardAuth (no login page of its own), and uses the Kubernetes downward API to auto-discover service health.

See the [official documentation](https://gethomepage.dev) for full widget and configuration reference.

!!! note "ForwardAuth — not OIDC"
    Homepage has no login page. It is protected by the Authelia ForwardAuth middleware (see Runbook 18). Do **not** attempt to configure it as an OIDC client.

## Step 1: RBAC

Homepage queries the Kubernetes API to discover pods, nodes, and HTTPRoutes for its Kubernetes widget. Create a ServiceAccount, ClusterRole, and ClusterRoleBinding.

Create `homelab-manifests/apps/homepage/rbac.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: homepage
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: homepage
  namespace: homepage
  labels:
    app: homepage
automountServiceAccountToken: true
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: homepage
rules:
  - apiGroups: [""]
    resources: ["namespaces", "pods", "nodes"]
    verbs: ["get", "list"]
  - apiGroups: ["extensions", "networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list"]
  - apiGroups: ["traefik.io"]
    resources: ["ingressroutes"]
    verbs: ["get", "list"]
  - apiGroups: ["metrics.k8s.io"]
    resources: ["nodes", "pods"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: homepage
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: homepage
subjects:
  - kind: ServiceAccount
    name: homepage
    namespace: homepage
```

## Step 2: ConfigMap

Homepage reads all configuration from YAML files mounted at `/app/config`. Deliver them via a ConfigMap. Customise the service URLs and icons to match your deployment.

Create `homelab-manifests/apps/homepage/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: homepage
  namespace: homepage
data:
  settings.yaml: |
    title: Homelab
    theme: dark
    color: slate
    headerStyle: clean
    target: _self
    language: en
    quicklaunch:
      searchDescriptions: false
      hideInternetSearch: true
      hideVisitURL: false
    layout:
      Infrastructure:
        style: row
        columns: 4
      Apps:
        style: row
        columns: 4
      Media:
        style: row
        columns: 4

  widgets.yaml: |
    - kubernetes:
        cluster:
          show: true
          cpu: true
          memory: true
          showLabel: true
          label: homelab
        nodes:
          show: true
          cpu: true
          memory: true
          showLabel: true

    - datetime:
        text_size: xl
        format:
          dateStyle: long
          timeStyle: short
          hourCycle: h23

  services.yaml: |
    - Infrastructure:
        - Traefik:
            href: https://traefik.yourdomain.com
            description: Reverse proxy
            icon: traefik.png
            widget:
              type: traefik
              url: https://traefik.yourdomain.com

        - ArgoCD:
            href: https://argocd.yourdomain.com
            description: GitOps
            icon: argocd.png

        - Authelia:
            href: https://auth.yourdomain.com
            description: SSO
            icon: authelia.png

        - Grafana:
            href: https://grafana.yourdomain.com
            description: Metrics
            icon: grafana.png
            widget:
              type: grafana
              url: https://grafana.yourdomain.com
              username: admin
              password: "{{HOMEPAGE_VAR_GRAFANA_PASS}}"

        - Prometheus:
            href: https://prometheus.yourdomain.com
            description: Alerting
            icon: prometheus.png

        - Forgejo:
            href: https://git.yourdomain.com
            description: Git
            icon: gitea.png
            widget:
              type: gitea
              url: https://git.yourdomain.com
              key: "{{HOMEPAGE_VAR_FORGEJO_TOKEN}}"

        - Woodpecker:
            href: https://ci.yourdomain.com
            description: CI/CD
            icon: woodpecker-ci.png

        - Vaultwarden:
            href: https://vault.yourdomain.com
            description: Passwords
            icon: bitwarden.png

    - Apps:
        - Nextcloud:
            href: https://cloud.yourdomain.com
            description: Files
            icon: nextcloud.png
            widget:
              type: nextcloud
              url: https://cloud.yourdomain.com
              username: admin
              password: "{{HOMEPAGE_VAR_NC_PASS}}"

        - Paperless:
            href: https://paperless.yourdomain.com
            description: Documents
            icon: paperless-ngx.png
            widget:
              type: paperlessngx
              url: https://paperless.yourdomain.com
              key: "{{HOMEPAGE_VAR_PAPERLESS_KEY}}"

        - ntfy:
            href: https://ntfy.yourdomain.com
            description: Notifications
            icon: ntfy.png

        - lldap:
            href: https://lldap.yourdomain.com
            description: User directory
            icon: ldap.png

    - Media:
        - Immich:
            href: https://photos.yourdomain.com
            description: Photos
            icon: immich.png
            widget:
              type: immich
              url: https://photos.yourdomain.com
              key: "{{HOMEPAGE_VAR_IMMICH_KEY}}"

  bookmarks.yaml: |
    - Quick links:
        - GitHub:
            - href: https://github.com
              icon: github.png
        - Cloudflare:
            - href: https://dash.cloudflare.com
              icon: cloudflare.png
        - UniFi:
            - href: https://unifi.ui.com
              icon: unifi.png

  kubernetes.yaml: |
    mode: cluster

  docker.yaml: |
    {}

  custom.js: ""

  custom.css: ""
```

### Protecting sensitive values

Instead of hardcoding API keys in the ConfigMap, use `HOMEPAGE_VAR_XXX` environment variables so secrets stay out of Git. The ConfigMap references `{{HOMEPAGE_VAR_GRAFANA_PASS}}` etc. — supply the actual values via a SealedSecret in Step 3.

## Step 3: SealedSecret for API Keys

```bash
kubectl create secret generic homepage-env \
  --namespace homepage \
  --from-literal=HOMEPAGE_VAR_GRAFANA_PASS=<grafana_admin_pass> \
  --from-literal=HOMEPAGE_VAR_NC_PASS=<nextcloud_admin_pass> \
  --from-literal=HOMEPAGE_VAR_PAPERLESS_KEY=<paperless_api_key> \
  --from-literal=HOMEPAGE_VAR_FORGEJO_TOKEN=<forgejo_api_token> \
  --from-literal=HOMEPAGE_VAR_IMMICH_KEY=<immich_api_key> \
  --dry-run=client -o yaml \
  | kubeseal --controller-name=sealed-secrets-controller \
             --controller-namespace=sealed-secrets \
             --format yaml \
  > homepage-env-sealed.yaml
```

Commit to `homelab-manifests/apps/homepage/`.

## Step 4: Deployment

Create `homelab-manifests/apps/homepage/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: homepage
  namespace: homepage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: homepage
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: homepage
    spec:
      serviceAccountName: homepage
      automountServiceAccountToken: true
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        runAsNonRoot: true
      containers:
        - name: homepage
          image: ghcr.io/gethomepage/homepage:latest
          ports:
            - name: http
              containerPort: 3000
          env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: HOMEPAGE_ALLOWED_HOSTS
              value: "homepage.yourdomain.com,$(POD_IP)"
          envFrom:
            - secretRef:
                name: homepage-env
          livenessProbe:
            httpGet:
              path: /api/healthcheck
              port: http
            initialDelaySeconds: 5
          volumeMounts:
            - name: config
              mountPath: /app/config
            - name: logs
              mountPath: /app/config/logs
      volumes:
        - name: config
          configMap:
            name: homepage
        - name: logs
          emptyDir: {}
```

Create `homelab-manifests/apps/homepage/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: homepage
  namespace: homepage
spec:
  selector:
    app: homepage
  ports:
    - port: 3000
      targetPort: 3000
```

## Step 5: HTTPRoute (with Authelia ForwardAuth)

Homepage has no built-in auth, so Authelia gates it via ForwardAuth. Under the Gateway API the `Middleware` lives in the app's namespace and the `HTTPRoute` references it with an `ExtensionRef` filter (see [Deploying an App](apps-deploy-pattern.md)). TLS is handled by the Gateway's wildcard cert. Create `homelab-manifests/apps/homepage/httproute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: authelia-forwardauth
  namespace: homepage
spec:
  forwardAuth:
    address: http://authelia.authelia.svc.cluster.local/api/authz/forward-auth
    trustForwardHeader: true
    authResponseHeaders: [Remote-User, Remote-Groups, Remote-Name, Remote-Email]
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: homepage
  namespace: homepage
spec:
  parentRefs:
    - name: traefik
      namespace: traefik
      sectionName: websecure
  hostnames:
    - homepage.yourdomain.com
  rules:
    - filters:
        - type: ExtensionRef
          extensionRef:
            group: traefik.io
            kind: Middleware
            name: authelia-forwardauth
      backendRefs:
        - name: homepage
          port: 3000
```

!!! note "ForwardAuth prerequisites"
    Per [Deploying an App](apps-deploy-pattern.md): Traefik's Kubernetes IngressRoute CRD provider must stay enabled for `ExtensionRef` middlewares to resolve, and the exact Authelia `forward-auth` address/headers are version-specific. No ForwardAuth app is deployed yet — confirm against Authelia's Traefik integration docs at deploy.

Commit all manifests to `homelab-manifests/apps/homepage/` and let ArgoCD sync.

## Step 6: Add API Tokens

Each widget integration needs an API key from the corresponding service. Retrieve or generate:

| Service | Where to get the key |
|---------|---------------------|
| Grafana | Profile → API Keys → Add |
| Nextcloud | User Settings → Security → App passwords |
| Paperless | Admin → API Token |
| Forgejo | User Settings → Applications → Generate Token |
| Immich | User Settings → API Keys |

Update the SealedSecret with the retrieved values and commit.

## Customisation

- **Icons** — Homepage resolves icon names against the [Dashboard Icons](https://github.com/walkxcode/dashboard-icons) set automatically. Use the icon name without `.png` to get auto-theming, or a full URL for custom icons.
- **Bookmarks** — Add `bookmarks.yaml` entries to create a pinned link panel alongside services.
- **Additional widgets** — Homepage has built-in widgets for AdGuard (`type: adguard`), Sonarr, Radarr, qBittorrent, Home Assistant, and many others — add them as you deploy each service in subsequent runbooks.

## Verification

- [ ] Homepage pod Running:

    ```bash
    kubectl get pods -n homepage
    ```

- [ ] `https://homepage.yourdomain.com` redirects to Authelia login for unauthenticated requests.
- [ ] After login, dashboard loads with the service grid.
- [ ] Kubernetes widget shows cluster CPU/memory and node list.
- [ ] Widget integrations that have API keys show live data.
- [ ] Verify the RBAC grants work (no 403 in pod logs):

    ```bash
    kubectl logs -n homepage deploy/homepage
    ```
