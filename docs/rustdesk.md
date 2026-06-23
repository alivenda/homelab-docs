# RustDesk Server

Self-hosted relay server for RustDesk remote desktop — ID registration and traffic relay.

| | |
|---|---|
| **Difficulty** | Beginner–Intermediate |
| **Time Estimate** | 30–45 minutes |
| **Runs On** | k3s (cluster) — LoadBalancer service via MetalLB |
| **Depends On** | Kubernetes (k3s, MetalLB), Networking (Lab VLAN IP range) |

RustDesk ([rustdesk.com](https://rustdesk.com)) is an open-source remote desktop application. This runbook deploys the **server components** (`hbbs` for peer ID registration, `hbbr` for traffic relay) on the cluster. The actual RustDesk client apps are installed separately on the machines you want to connect to. ARM64 ✅ (`rustdesk/rustdesk-server` ships multiarch). See the [self-hosting docs](https://rustdesk.com/docs/en/self-host/rustdesk-server-oss/) for full reference.

**No Traefik / Authelia involvement:** RustDesk uses TCP/UDP ports directly, not HTTP. MetalLB assigns a dedicated IP to the RustDesk LoadBalancer service and ports are exposed as-is.

## Ports required

| Port | Protocol | Component | Purpose |
|------|----------|-----------|---------|
| 21115 | TCP | hbbs | NAT type test |
| 21116 | TCP + UDP | hbbs | ID registration and heartbeat |
| 21117 | TCP | hbbr | Relay traffic |
| 21118 | TCP | hbbs | Web clients (optional) |
| 21119 | TCP | hbbr | Web clients (optional) |

Ports 21118 and 21119 are only needed for browser-based clients. For desktop/mobile app only, 21115–21117 are sufficient.

## Step 1: Deploy hbbs and hbbr

Both components use the same image with different commands. They share a volume so the key pair persists.

Create `homelab-manifests/apps/rustdesk/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: rustdesk
```

Create `homelab-manifests/apps/rustdesk/pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rustdesk-data
  namespace: rustdesk
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: nfs-storage
  resources:
    requests:
      storage: 1Gi
```

Create `homelab-manifests/apps/rustdesk/deployment.yaml`. Both `hbbs` and `hbbr` run as containers in the same pod so they can share the key volume without multi-writer PVC complications:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rustdesk
  namespace: rustdesk
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rustdesk
  template:
    metadata:
      labels:
        app: rustdesk
    spec:
      containers:
        - name: hbbs
          image: rustdesk/rustdesk-server:latest
          command: ["hbbs"]
          args: ["-r", "10.0.20.50:21117"]
          ports:
            - containerPort: 21115
              protocol: TCP
            - containerPort: 21116
              protocol: TCP
            - containerPort: 21116
              protocol: UDP
            - containerPort: 21118
              protocol: TCP
          volumeMounts:
            - name: data
              mountPath: /root
        - name: hbbr
          image: rustdesk/rustdesk-server:latest
          command: ["hbbr"]
          ports:
            - containerPort: 21117
              protocol: TCP
            - containerPort: 21119
              protocol: TCP
          volumeMounts:
            - name: data
              mountPath: /root
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: rustdesk-data
```

Replace `10.0.20.50` in the `hbbs` args with the MetalLB IP you assign in Step 2.

## Step 2: LoadBalancer Service

A single Service selects the pod and exposes all ports on one MetalLB IP:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: rustdesk
  namespace: rustdesk
  annotations:
    metallb.universe.tf/loadBalancerIPs: 10.0.20.50
spec:
  type: LoadBalancer
  selector:
    app: rustdesk
  ports:
    - name: hbbs-nat
      port: 21115
      targetPort: 21115
      protocol: TCP
    - name: hbbs-id-tcp
      port: 21116
      targetPort: 21116
      protocol: TCP
    - name: hbbs-id-udp
      port: 21116
      targetPort: 21116
      protocol: UDP
    - name: hbbr-relay
      port: 21117
      targetPort: 21117
      protocol: TCP
    - name: hbbs-ws
      port: 21118
      targetPort: 21118
      protocol: TCP
    - name: hbbr-ws
      port: 21119
      targetPort: 21119
      protocol: TCP
```

Set `metallb.universe.tf/loadBalancerIPs` to an unused IP in your MetalLB pool.

Commit all manifests to `homelab-manifests/apps/rustdesk/` and let ArgoCD sync.

## Step 3: Retrieve the Public Key

On first startup, `hbbs` generates an Ed25519 key pair in the `/root` volume. Retrieve the public key — clients must have it to verify the server identity:

```bash
kubectl exec -n rustdesk deploy/rustdesk -c hbbs -- cat /root/id_ed25519.pub
```

Save the public key to Vaultwarden.

## Step 4: Configure RustDesk Clients

Install the RustDesk app on every machine you want to remotely access, and on your main machine for control. In each client:

1. Go to **Network → ID/Relay Server**.
2. Set **ID Server** to `<rustdesk-lb-ip>` (or Tailscale hostname if routing via Tailscale).
3. Set **Relay Server** to `<rustdesk-lb-ip>`.
4. Set **Key** to the public key from Step 3.

Each device shows a numeric ID — share the ID and a one-time password to allow connections.

## Access via Tailscale

If you want to access the RustDesk server from outside your LAN without opening firewall ports:

1. The cluster nodes are already on Tailscale (from Networking). The MetalLB IP `10.0.20.50` is on the Lab VLAN which is advertised via the Tailscale subnet router.
2. Set the RustDesk client ID/Relay server to the MetalLB IP — it will be reachable from anywhere on your Tailscale network.

## Verification

- [ ] RustDesk pod Running (both hbbs and hbbr containers):

    ```bash
    kubectl get pods -n rustdesk
    kubectl logs -n rustdesk deploy/rustdesk -c hbbs
    kubectl logs -n rustdesk deploy/rustdesk -c hbbr
    ```

- [ ] MetalLB has assigned the expected IP:

    ```bash
    kubectl get svc -n rustdesk
    ```

- [ ] Public key retrieved and saved to Vaultwarden.
- [ ] RustDesk client on main machine connects to a remote machine using the custom server.
