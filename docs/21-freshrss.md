# Runbook 21: FreshRSS

Self-hosted RSS aggregator with multi-user support and mobile sync APIs.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 30–45 minutes |
| **Runs On** | k3s (cluster) |
| **Depends On** | Runbook 5 (k3s), Runbook 6 (Traefik), Runbook 18 (Authelia) |

FreshRSS ([freshrss.org](https://freshrss.org)) is a fast, low-memory RSS/Atom aggregator. It exposes a Google Reader and Fever API so clients like Reeder, NetNewsWire, and Readrops can sync feeds and read state. ARM64 ✅ (`lscr.io/linuxserver/freshrss` ships multiarch). See the [FreshRSS documentation](https://freshrss.github.io/FreshRSS/) for full reference.

**Auth mode:** Authelia ForwardAuth passes a `Remote-User` header; FreshRSS auto-authenticates web users by that header. Mobile sync clients use FreshRSS's own API password and are unaffected by the ForwardAuth layer.

## Step 1: Deploy FreshRSS

Create `homelab-manifests/apps/freshrss/deployment.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: freshrss
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: freshrss
  namespace: freshrss
spec:
  replicas: 1
  selector:
    matchLabels:
      app: freshrss
  template:
    metadata:
      labels:
        app: freshrss
    spec:
      containers:
        - name: freshrss
          image: lscr.io/linuxserver/freshrss:latest
          ports:
            - containerPort: 80
          env:
            - name: PUID
              value: "1000"
            - name: PGID
              value: "1000"
            - name: TZ
              value: Europe/London
          volumeMounts:
            - name: config
              mountPath: /config
      volumes:
        - name: config
          persistentVolumeClaim:
            claimName: freshrss-config
```

Create `homelab-manifests/apps/freshrss/pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: freshrss-config
  namespace: freshrss
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 2Gi
```

Create `homelab-manifests/apps/freshrss/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: freshrss
  namespace: freshrss
spec:
  selector:
    app: freshrss
  ports:
    - port: 80
      targetPort: 80
```

## Step 2: IngressRoute

FreshRSS web UI is protected by Authelia ForwardAuth. The Authelia middleware passes `Remote-User`, `Remote-Email`, and `Remote-Name` headers to FreshRSS after authentication.

Create `homelab-manifests/apps/freshrss/ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: freshrss
  namespace: freshrss
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`rss.yourdomain.com`)
      kind: Rule
      middlewares:
        - name: authelia@kubernetescrd
          namespace: authelia
      services:
        - name: freshrss
          port: 80
  tls:
    certResolver: cloudflare
    domains:
      - main: rss.yourdomain.com
```

Commit all manifests to `homelab-manifests/apps/freshrss/` and let ArgoCD sync.

## Step 3: Initial Setup

Open `https://rss.yourdomain.com`. The setup wizard will prompt you:

1. **Database** — SQLite is the default and sufficient for personal use.
2. **Default user** — enter the username that Authelia will pass as `Remote-User`. This must exactly match your lldap username (the value Authelia sends in the `Remote-User` header).
3. **Authentication method** — choose **HTTP authentication (proxy)**.

!!! warning "Authentication method must be set during setup"
    Choosing HTTP authentication here tells FreshRSS to trust the `Remote-User` header. You cannot change this after the wizard without editing the config file.

After the wizard, FreshRSS will auto-authenticate any future web visit that has passed Authelia, skipping the FreshRSS login form entirely.

## Step 4: API Access for Mobile Clients

Mobile RSS clients (Reeder, NetNewsWire, Readrops, etc.) connect via the Google Reader or Fever API and do **not** go through Authelia — they authenticate directly against FreshRSS using its own API password.

In the FreshRSS web UI:

1. Go to **User profile → API management**.
2. Set an **API password** (different from any Authelia credentials).
3. Configure the mobile client with:
   - **Server URL**: `https://rss.yourdomain.com`
   - **Username**: your FreshRSS username
   - **Password**: the API password set above

Save the API password to Vaultwarden.

## Step 5: Add Feeds

In the FreshRSS web UI:

1. **Subscription tools → Add a subscription** and paste a feed URL.
2. Feeds refresh automatically on a cron schedule (default every hour for the LinuxServer image).
3. Organise feeds into categories from **Subscription management**.

## Verification

- [ ] FreshRSS pod Running:

    ```bash
    kubectl get pods -n freshrss
    ```

- [ ] `https://rss.yourdomain.com` redirects to Authelia login for unauthenticated requests.
- [ ] After Authelia login, FreshRSS loads without prompting for its own password.
- [ ] A test feed (e.g. `https://blog.cloudflare.com/rss/`) can be added and refreshed.
- [ ] Mobile client connects successfully using the API password.
- [ ] API password saved to Vaultwarden.
