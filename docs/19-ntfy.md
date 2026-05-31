# Runbook 19: ntfy

Self-hosted push notifications for the entire homelab stack.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 30–45 minutes |
| **Runs On** | k3s (cluster) |
| **Depends On** | Runbook 5 (k3s), Runbook 6 (Traefik) |

ntfy is a pub/sub notification service — services publish messages to named topics and your phone (or any HTTP client) subscribes to receive them. Every other service in this stack can send notifications through it: Woodpecker pipeline results, Restic backup alerts, Prometheus alertmanager, Home Assistant automations, Paperless document ingestion, Uptime Kuma alerts.

The ARM64 image (`binwiederhier/ntfy`) ships official multiarch builds. See the [ntfy self-hosting docs](https://docs.ntfy.sh/install/) and [config reference](https://docs.ntfy.sh/config/) for full reference.

## Step 1: Create the ConfigMap

ntfy reads its configuration from `/etc/ntfy/server.yml`. In k3s, deliver this via a ConfigMap.

Create `homelab-manifests/apps/ntfy/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ntfy-config
  namespace: ntfy
data:
  server.yml: |
    base-url: "https://ntfy.yourdomain.com"
    listen-http: ":80"
    cache-file: "/var/cache/ntfy/cache.db"
    cache-duration: "12h"
    auth-file: "/var/lib/ntfy/user.db"
    auth-default-access: "deny-all"
    behind-proxy: true
    attachment-cache-dir: "/var/cache/ntfy/attachments"
    attachment-file-size-limit: "15m"
    attachment-total-size-limit: "1g"
```

`auth-default-access: deny-all` makes this a private server — every subscriber and publisher must authenticate. This is the correct setting for a homelab instance.

## Step 2: Deploy as a StatefulSet

A StatefulSet ensures the cache and auth databases survive pod restarts.

Create `homelab-manifests/apps/ntfy/statefulset.yaml`:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ntfy
  namespace: ntfy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ntfy
  serviceName: ntfy
  template:
    metadata:
      labels:
        app: ntfy
    spec:
      containers:
        - name: ntfy
          image: binwiederhier/ntfy:latest
          args: [serve]
          ports:
            - containerPort: 80
          volumeMounts:
            - name: config
              mountPath: /etc/ntfy
            - name: cache
              mountPath: /var/cache/ntfy
            - name: data
              mountPath: /var/lib/ntfy
      volumes:
        - name: config
          configMap:
            name: ntfy-config
  volumeClaimTemplates:
    - metadata:
        name: cache
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: nfs-storage
        resources:
          requests:
            storage: 2Gi
    - metadata:
        name: data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: nfs-storage
        resources:
          requests:
            storage: 1Gi
```

Create `homelab-manifests/apps/ntfy/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ntfy
  namespace: ntfy
spec:
  selector:
    app: ntfy
  ports:
    - port: 80
      targetPort: 80
```

## Step 3: IngressRoute

Create `homelab-manifests/apps/ntfy/ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: ntfy
  namespace: ntfy
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`ntfy.yourdomain.com`)
      kind: Rule
      services:
        - name: ntfy
          port: 80
  tls:
    certResolver: cloudflare
    domains:
      - main: ntfy.yourdomain.com
```

!!! note "No Authelia middleware on ntfy"
    ntfy handles its own authentication via the `auth-file` database. Do not put the Authelia ForwardAuth middleware in front of ntfy — it would break the ntfy Android/iOS apps and HTTP publishing integrations that send auth credentials directly.

Commit all files to `homelab-manifests/apps/ntfy/` and let ArgoCD sync.

## Step 4: Create Users and Topics

Once the pod is running, exec into it to create your admin user and subscribe to topics:

```bash
kubectl exec -it -n ntfy statefulset/ntfy -- sh
```

Inside the container:

```bash
# Create admin user (saves to /var/lib/ntfy/user.db)
ntfy user add --role=admin <YOUR_USERNAME>

# Verify
ntfy user list
```

Exit the shell. Save the credentials to Vaultwarden.

## Step 5: Mobile App Setup

Install the [ntfy Android app](https://play.google.com/store/apps/details?id=io.xtph.ntfy) or [iOS app](https://apps.apple.com/app/ntfy/id1625396347).

In the app settings:
1. Add your server: `https://ntfy.yourdomain.com`
2. Enter your username and password.
3. Subscribe to a topic, e.g. `homelab-alerts`.

Test with a curl publish from a cluster node:

```bash
curl -u username:password \
  -d "ntfy is working" \
  https://ntfy.yourdomain.com/homelab-alerts
```

The notification should arrive on your phone within seconds.

## Integrating Other Services

### Woodpecker CI

In Woodpecker's pipeline YAML:

```yaml
- name: notify
  image: plugins/webhook
  settings:
    urls: https://ntfy.yourdomain.com/woodpecker
    content_type: application/json
    template: |
      {"topic":"woodpecker","message":"Build {{build.status}}: {{repo.name}} #{{build.number}}"}
```

Or use the ntfy plugin directly if available for your runner architecture.

### Restic (via backup script)

In your backup script (`homelab-manifests/apps/backups/`), add after a successful or failed backup run:

```bash
curl -s -u username:password \
  -H "Title: Restic backup" \
  -d "Backup of $VOLUME completed" \
  https://ntfy.yourdomain.com/homelab-alerts
```

### Prometheus Alertmanager

In `alertmanager.yml`:

```yaml
receivers:
  - name: ntfy
    webhook_configs:
      - url: https://ntfy.yourdomain.com/homelab-alerts
        http_config:
          basic_auth:
            username: <USERNAME>
            password: <PASSWORD>
```

### Home Assistant

In your HA `configuration.yaml` or via the UI:

```yaml
notify:
  - name: ntfy
    platform: rest
    resource: https://ntfy.yourdomain.com/homelab-alerts
    method: POST_JSON
    headers:
      Authorization: Basic <BASE64_USER_PASS>
    data:
      message: "{{ message }}"
```

## Verification

- [ ] ntfy pod Running:

    ```bash
    kubectl get pods -n ntfy
    ```

- [ ] `https://ntfy.yourdomain.com` loads the ntfy web UI.
- [ ] A test notification published via curl arrives on the mobile app.
- [ ] Unauthenticated publish attempt is rejected (HTTP 401):

    ```bash
    curl -d "test" https://ntfy.yourdomain.com/homelab-alerts
    # Expect: 401 Unauthorized
    ```

- [ ] Credentials saved to Vaultwarden.
