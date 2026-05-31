# Runbook 34: Collabora Online

In-browser document editing integrated with Nextcloud.

| | |
|---|---|
| **Difficulty** | Intermediate |
| **Time Estimate** | 45–60 minutes |
| **Runs On** | k3s (cluster) |
| **Depends On** | Runbook 13 (Nextcloud), Runbook 6 (Traefik) |

Collabora Online is an open-source office suite (Writer, Calc, Impress) that runs as a CODE server alongside Nextcloud. Nextcloud acts as the client — you open a document in Nextcloud Files and editing happens in a browser tab powered by the CODE server. ARM64 ✅ (`collabora/code` ships multiarch). See the [Collabora docs](https://sdk.collaboraonline.com/docs/installation/CODE_Docker_image.html) for full reference.

!!! note "This extends Runbook 13"
    Collabora Online does not run standalone. It must be connected to an existing Nextcloud instance. Complete Runbook 13 before this runbook.

## Step 1: Deploy Collabora CODE Server

Create `homelab-manifests/apps/collabora/deployment.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: collabora
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: collabora
  namespace: collabora
spec:
  replicas: 1
  selector:
    matchLabels:
      app: collabora
  template:
    metadata:
      labels:
        app: collabora
    spec:
      containers:
        - name: collabora
          image: collabora/code:latest
          ports:
            - containerPort: 9980
          env:
            - name: aliasgroup1
              value: https://cloud.yourdomain.com:443
            - name: DONT_GEN_SSL_CERT
              value: "1"
            - name: extra_params
              value: "--o:ssl.enable=false --o:ssl.termination=true"
          securityContext:
            capabilities:
              add:
                - MKNOD
```

The `aliasgroup1` env var whitelists the Nextcloud URL. `DONT_GEN_SSL_CERT` and `ssl.termination=true` tell CODE that TLS termination happens at Traefik — CODE speaks plain HTTP internally.

Create `homelab-manifests/apps/collabora/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: collabora
  namespace: collabora
spec:
  selector:
    app: collabora
  ports:
    - port: 9980
      targetPort: 9980
```

## Step 2: IngressRoute

Collabora requires specific headers and WebSocket support. Create `homelab-manifests/apps/collabora/ingressroute.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: collabora
  namespace: collabora
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`office.yourdomain.com`)
      kind: Rule
      services:
        - name: collabora
          port: 9980
  tls:
    certResolver: cloudflare
    domains:
      - main: office.yourdomain.com
```

!!! note "No Authelia middleware"
    Collabora communicates directly with Nextcloud using token-based auth. Do not put Authelia ForwardAuth in front of Collabora — it will break the Nextcloud integration.

Commit all manifests to `homelab-manifests/apps/collabora/` and let ArgoCD sync.

## Step 3: Connect Collabora to Nextcloud

In your Nextcloud instance:

1. Go to **Apps → Search → "Nextcloud Office"** and enable the **Nextcloud Office** app (this is the Collabora integration).
2. Go to **Admin Settings → Nextcloud Office**.
3. Select **Use your own server**.
4. Enter the CODE server URL: `https://office.yourdomain.com`.
5. Click **Save**.

Nextcloud will contact the CODE server and display a green checkmark when connectivity is confirmed.

## Step 4: Verify Document Editing

1. In Nextcloud Files, upload a `.docx`, `.xlsx`, or `.pptx` file.
2. Click on the file — it should open in the Collabora editor within the browser tab.
3. Make an edit and verify changes are saved back to Nextcloud.

## Verification

- [ ] Collabora pod Running:

    ```bash
    kubectl get pods -n collabora
    ```

- [ ] `https://office.yourdomain.com` returns a 200 response (the CODE server root):

    ```bash
    curl -s -o /dev/null -w "%{http_code}" https://office.yourdomain.com
    ```

- [ ] Nextcloud Office admin settings show a green connected status.
- [ ] A test document opens and edits save correctly.
