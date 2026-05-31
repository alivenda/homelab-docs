# Runbook 37: Ollama + Open WebUI

Local LLM inference on the NAS with a browser chat interface.

| | |
|---|---|
| **Difficulty** | Beginner |
| **Time Estimate** | 30–45 minutes |
| **Runs On** | UGREEN DXP6800 Pro NAS (Docker Compose — not k3s) |
| **Depends On** | NAS RAM upgrade to 16 GB, Runbook 6 (Traefik reverse proxy for HTTPS) |

!!! warning "Defer until NAS RAM upgrade"
    Ollama and the models it runs are RAM-hungry. The NAS currently has 8 GB DDR5, which leaves insufficient headroom after Plex and Immich are running. Upgrade to 16 GB DDR5 SO-DIMM first (see [personal-cloud-stack.md](personal-cloud-stack.md#nas-upgrade-path)) — then deploy this runbook.

    With 16 GB: Plex + Immich use roughly 4–6 GB combined, leaving 10+ GB for Ollama. Models like `llama3.2:3b` (2 GB) or `gemma2:9b` (5 GB) are practical. The NAS uses x86 CPU inference — expect 10–30 tokens/second for 3B models, slower for larger ones.

[Ollama](https://ollama.com) serves local language models via an OpenAI-compatible API. [Open WebUI](https://openwebui.com) provides a chat interface similar to ChatGPT that connects to Ollama. ARM64 note: this runbook targets the x86 NAS. See the [Ollama Docker docs](https://github.com/ollama/ollama) and [Open WebUI docs](https://docs.openwebui.com) for full reference.

## Step 1: Docker Compose on NAS

Create a `docker-compose.yml` on the NAS (via the UGREEN NAS Manager Docker interface or SSH):

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - 11434:11434
    volumes:
      - ollama-models:/root/.ollama
    environment:
      - OLLAMA_HOST=0.0.0.0

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - 3000:8080
    volumes:
      - open-webui-data:/app/backend/data
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
      - WEBUI_SECRET_KEY=<generate_with_openssl_rand_hex_32>
      - ENABLE_SIGNUP=false
      - ENABLE_OAUTH_SIGNUP=true
      - OAUTH_CLIENT_ID=open-webui
      - OAUTH_CLIENT_SECRET=<oidc_client_secret_plaintext>
      - OPENID_PROVIDER_URL=https://auth.yourdomain.com/.well-known/openid-configuration
      - OAUTH_PROVIDER_NAME=Authelia
      - OAUTH_SCOPES=openid profile email
      - OAUTH_MERGE_ACCOUNTS_BY_EMAIL=true
    depends_on:
      - ollama

volumes:
  ollama-models:
  open-webui-data:
```

## Step 2: Register the OIDC Client in Authelia

In `homelab-manifests/apps/authelia/values.yaml`, add under `configMap.identity_providers.oidc.clients`:

```yaml
- client_id: open-webui
  client_name: Open WebUI
  client_secret:
    value: <HASHED_SECRET>   # authelia crypto hash generate pbkdf2 --variant sha512
  redirect_uris:
    - https://ai.yourdomain.com/oauth/oidc/callback
  scopes: [openid, profile, email]
  token_endpoint_auth_method: client_secret_basic
  grant_types: [authorization_code]
  response_types: [code]
```

Commit and upgrade Authelia.

## Step 3: Enable ExternalName routing in Traefik

Traefik blocks ExternalName Services by default. Enable it in your Traefik Helm values (`homelab-manifests/apps/traefik/values.yaml`):

```yaml
providers:
  kubernetesIngress:
    allowExternalNameServices: true
  kubernetesCRD:
    allowExternalNameServices: true
```

Commit and let ArgoCD sync before creating the IngressRoute.

## Step 4: Expose via Traefik

The NAS services need to be reachable through Traefik (running on the cluster). Add a ServersTransport and IngressRoute pointing to the NAS IP.

Create `homelab-manifests/apps/nas-services/open-webui.yaml`:

```yaml
apiVersion: traefik.io/v1alpha1
kind: ServersTransport
metadata:
  name: nas-transport
  namespace: default
spec:
  insecureSkipVerify: true
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: open-webui
  namespace: default
spec:
  entryPoints: [websecure]
  routes:
    - match: Host(`ai.yourdomain.com`)
      kind: Rule
      services:
        - name: open-webui-nas
          port: 3000
          kind: Service
  tls:
    certResolver: cloudflare
    domains:
      - main: ai.yourdomain.com
---
apiVersion: v1
kind: Service
metadata:
  name: open-webui-nas
  namespace: default
spec:
  type: ExternalName
  externalName: <nas-ip>
  ports:
    - port: 3000
```

## Step 5: Pull Models

Once Ollama is running, pull models via Docker exec on the NAS. A good starting set:

```bash
# Small, fast — good for quick questions (2 GB RAM)
docker exec ollama ollama pull llama3.2:3b

# Medium, balanced quality/speed (5 GB RAM)
docker exec ollama ollama pull gemma2:9b

# Coding-focused (4 GB RAM)
docker exec ollama ollama pull qwen2.5-coder:7b
```

Find all available models at [ollama.com/library](https://ollama.com/library). Check the model page for RAM requirements — a 7B model needs roughly 5 GB of RAM.

## Step 6: First Login

Open `https://ai.yourdomain.com`. Click **Continue with Authelia** to log in. The first user to log in becomes admin.

Select a model from the dropdown at the top of the chat interface and start a conversation.

## Verification

- [ ] Ollama container running on NAS:

    ```bash
    docker ps | grep ollama
    ```

- [ ] Ollama API reachable:

    ```bash
    curl http://<nas-ip>:11434/api/tags
    ```

- [ ] At least one model pulled:

    ```bash
    docker exec ollama ollama list
    ```

- [ ] `https://ai.yourdomain.com` loads and OIDC login works.
- [ ] Chat response received from a local model.
- [ ] `WEBUI_SECRET_KEY` and OIDC client secret saved to Vaultwarden.
