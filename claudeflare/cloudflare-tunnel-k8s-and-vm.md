# Cloudflare Tunnel - Implementation Guide

Expose internal services to the public internet without a public IP.
Target environment: Linux (Fedora workstation, RHEL/Ubuntu VMs, Kubernetes cluster).

Sources: Cloudflare official documentation, September 2026.

---

## 1. How it works

`cloudflared` runs inside your network. It opens an **outbound** connection to
Cloudflare's edge. Cloudflare then forwards public requests back down that
connection.

Key consequences:

- No inbound firewall rule is needed.
- No port forwarding on your router.
- Your home or lab public IP stays hidden.
- Requests arrive over HTTPS, terminated at Cloudflare's edge.

**Required outbound port: `7844`.** Check this before anything else if your
firewall is restrictive.

### Two management models

| Model | Config lives in | Auth method | Best for |
|---|---|---|---|
| **Remotely-managed** | Cloudflare dashboard | Tunnel token | Kubernetes, containers, fleets |
| **Locally-managed** | `config.yml` on the host | `cert.pem` + credentials JSON | Single VM, GitOps-free setups |

Pick remotely-managed for Kubernetes. Pick locally-managed for a VM when you want
the config in a file you control.

---

## 2. Prerequisites (both cases)

1. A domain added to Cloudflare.
2. Domain nameservers pointed at Cloudflare.
3. Outbound access to Cloudflare on port `7844`.

Verify the port from the host:

```bash
nc -zv region1.v2.argotunnel.com 7844
```

---

## 3. Case A - Kubernetes service

This uses a **remotely-managed** tunnel. The token lives in a Kubernetes Secret.

### 3.1 Architecture

Run `cloudflared` as its own Deployment. Do not use a sidecar.

Reasons:

- You scale `cloudflared` independently of your apps.
- Each replica can reach every Service in the cluster.
- Replicas give high availability, not load balancing.

Cloudflare warns against autoscaling `cloudflared`. Removing a replica breaks
active user connections.

### 3.2 Create the tunnel

1. Cloudflare dashboard > **Networking** > **Tunnels**.
2. Select **Create a tunnel**.
3. Name it, for example `homelab-tunnel`.
4. Choose **Docker** as the platform.
5. Copy **only the token value**, not the full command. It looks like
   `eyJhIjoiNWFiNGU5Z...`.

### 3.3 Store the token as a Secret

```yaml
# tunnel-token.yaml
apiVersion: v1
kind: Secret
metadata:
  name: tunnel-token
  namespace: cloudflare
stringData:
  token: <YOUR_TUNNEL_TOKEN>
```

```bash
kubectl create namespace cloudflare
kubectl apply -f tunnel-token.yaml
```

If you run External Secrets Operator, pull the token from your secret store
instead. Note that `TUNNEL_TOKEN` is injected at pod creation. A rotated token
needs a rollout restart:

```bash
kubectl rollout restart deployment/cloudflared -n cloudflare
```

### 3.4 Deploy cloudflared

```yaml
# tunnel.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: cloudflare
spec:
  replicas: 2
  selector:
    matchLabels:
      pod: cloudflared
  template:
    metadata:
      labels:
        pod: cloudflared
    spec:
      securityContext:
        sysctls:
          # Allows ICMP (ping, traceroute) to resources behind cloudflared.
          - name: net.ipv4.ping_group_range
            value: "65532 65532"
      containers:
        - image: cloudflare/cloudflared:latest
          name: cloudflared
          env:
            - name: TUNNEL_TOKEN
              valueFrom:
                secretKeyRef:
                  name: tunnel-token
                  key: token
          command:
            - cloudflared
            - tunnel
            - --no-autoupdate
            - --loglevel
            - info
            - --output
            - json
            - --metrics
            - 0.0.0.0:2000
            - run
          livenessProbe:
            httpGet:
              # /ready returns 200 only when connected to Cloudflare's network.
              path: /ready
              port: 2000
            failureThreshold: 1
            initialDelaySeconds: 10
            periodSeconds: 10
```

```bash
kubectl apply -f tunnel.yaml
kubectl get pods -n cloudflare
```

Notes on the flags:

- `--no-autoupdate` - the container image controls the version, not the binary
  updater. Pin a digest in production instead of `latest`.
- `--output json` - structured logs, easy to ship to Loki or similar.
- `--metrics 0.0.0.0:2000` - exposes `/ready` and Prometheus metrics.

If pods keep restarting, check the order of the `command` arguments. Run
parameters must come before `run`.

### 3.5 Verify

```bash
kubectl logs -n cloudflare deployment/cloudflared
```

Expect a line containing `"message":"Starting tunnel"` with a `tunnelID`.

### 3.6 Route a service

1. Dashboard > **Networking** > **Tunnels** > select your tunnel.
2. **Routes** tab > **Add route** > **Published application**.
3. Hostname: `app.your-domain.com`.
4. Service: `http://my-service.my-namespace.svc.cluster.local:80`.
5. Select **Add route**.

Cloudflare creates the proxied CNAME record for you.

### 3.7 Restrict what the tunnel can reach

This step is not in the official guide, but it matters.

By default a `cloudflared` pod can reach **every Service in every namespace**.
If an attacker exploits one exposed app, the pod becomes a pivot into your
Vault, your database, your Grafana.

Add a NetworkPolicy that allows egress only to the services you publish, plus
DNS and the Cloudflare edge.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cloudflared-egress
  namespace: cloudflare
spec:
  podSelector:
    matchLabels:
      pod: cloudflared
  policyTypes:
    - Egress
  egress:
    # DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
    # The one app you publish
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: my-namespace
          podSelector:
            matchLabels:
              app: my-app
      ports:
        - protocol: TCP
          port: 80
    # Cloudflare edge
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
      ports:
        - protocol: TCP
          port: 7844
        - protocol: UDP
          port: 7844
```

Adjust the CIDR if you want to narrow it to Cloudflare IP ranges.

### 3.8 Optional - the Cloudflare Operator

`adyanth/cloudflare-operator` adds CRDs (`Tunnel`, `TunnelBinding`). It manages
the `cloudflared` Deployment, its ConfigMap, and DNS records automatically when
you expose a new Service.

Status: alpha. Useful if you add services often. Skip it for a single tunnel.

---

## 4. Case B - Service on a VM

This uses a **locally-managed** tunnel. Config lives in `config.yml` on the host.

### 4.1 Install cloudflared

**RHEL / Fedora / Rocky:**

```bash
curl -fsSl https://pkg.cloudflare.com/cloudflared.repo | sudo tee /etc/yum.repos.d/cloudflared.repo
sudo dnf install cloudflared
```

**Debian / Ubuntu:**

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg \
  | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null

echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main" \
  | sudo tee /etc/apt/sources.list.d/cloudflared.list

sudo apt-get update && sudo apt-get install cloudflared
```

Verify:

```bash
cloudflared --version
```

### 4.2 Authenticate

```bash
cloudflared tunnel login
```

This opens a browser. On a headless VM, copy the printed URL and open it
elsewhere. It writes `cert.pem` into `~/.cloudflared/`.

### 4.3 Create the tunnel

```bash
cloudflared tunnel create vm-tunnel
```

Output gives you:

- a tunnel UUID
- a credentials file at `~/.cloudflared/<UUID>.json`

List your tunnels:

```bash
cloudflared tunnel list
```

### 4.4 Move credentials to a system path

The service runs as root. Put the files where root can find them.

```bash
sudo mkdir -p /etc/cloudflared
sudo mv ~/.cloudflared/<UUID>.json /etc/cloudflared/
```

### 4.5 Write the config file

```yaml
# /etc/cloudflared/config.yml
tunnel: <UUID>
credentials-file: /etc/cloudflared/<UUID>.json

ingress:
  - hostname: app.your-domain.com
    service: http://localhost:8000
  - hostname: api.your-domain.com
    service: http://localhost:9000
  # Catch-all is mandatory. It must be last.
  - service: http_status:404
```

Validate the ingress rules before starting:

```bash
sudo cloudflared --config /etc/cloudflared/config.yml tunnel ingress validate
```

### 4.6 Create the DNS route

```bash
cloudflared tunnel route dns vm-tunnel app.your-domain.com
```

This creates a proxied CNAME pointing to `<UUID>.cfargotunnel.com`.

### 4.7 Test in the foreground first

```bash
cloudflared tunnel --config /etc/cloudflared/config.yml run vm-tunnel
```

Open `https://app.your-domain.com` in a browser. Stop with `Ctrl+C` once it
works.

### 4.8 Install as a systemd service

```bash
sudo cloudflared --config /etc/cloudflared/config.yml service install
sudo systemctl enable --now cloudflared
sudo systemctl status cloudflared
```

Watch the logs:

```bash
sudo journalctl -u cloudflared -f
```

**Important gotcha:** `sudo` sets `$HOME` to `/root`. If your config sits in
`/home/<USER>/.cloudflared/`, `cloudflared` will not find it. Always pass
`--config` explicitly, as shown above.

### 4.9 Optional - a custom unit file

If you need extra flags, edit the unit directly:

```ini
# /etc/systemd/system/cloudflared.service
[Unit]
Description=Cloudflare Tunnel
After=network-online.target
Wants=network-online.target

[Service]
TimeoutStartSec=0
Type=notify
ExecStart=/usr/bin/cloudflared --no-autoupdate --config /etc/cloudflared/config.yml tunnel run
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart cloudflared
```

---

## 5. Comparison

| Aspect | Kubernetes | VM |
|---|---|---|
| Management model | Remote (token) | Local (`config.yml`) |
| Config location | Cloudflare dashboard | `/etc/cloudflared/config.yml` |
| Credentials | Kubernetes Secret | `<UUID>.json` + `cert.pem` |
| High availability | `replicas: 2` | Single process, restart on failure |
| Adding a service | Add route in dashboard | Edit `ingress:`, restart service |
| Health check | `/ready` on port 2000 | `systemctl status` |
| Blast radius | Reaches all cluster Services | Reaches all localhost ports |

---

## 6. Security checklist

- [ ] Never commit the tunnel token or the credentials JSON to Git.
- [ ] Pin the container image by digest, not `latest`.
- [ ] Add a NetworkPolicy in Kubernetes. Limit egress to published services only.
- [ ] On a VM, bind your application to `127.0.0.1`, not `0.0.0.0`.
- [ ] Put a Cloudflare Access policy in front of anything that is not truly public.
- [ ] Set `chmod 600` on `/etc/cloudflared/<UUID>.json`.
- [ ] Scrape `cloudflared` metrics on port 2000 and alert on `up == 0`.

---

## 7. Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| Pod restarts in a loop | Wrong flag order in `command` | Run parameters must precede `run` |
| `error="connection refused"` | Origin service not listening | `curl http://localhost:<port>` from the host |
| Tunnel connects, 502 from browser | Wrong `service:` URL in ingress | Use the full `svc.cluster.local` name |
| Service starts, no config found | `sudo` changed `$HOME` to `/root` | Pass `--config` explicitly |
| Nothing connects at all | Port 7844 blocked outbound | `nc -zv region1.v2.argotunnel.com 7844` |
| DNS resolves but times out | CNAME not proxied | Check the orange cloud in Cloudflare DNS |

---

## 8. Reference links

- Kubernetes deployment guide:
  https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/deployment-guides/kubernetes/
- Locally-managed tunnel:
  https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/local-management/create-local-tunnel/
- Run as a service on Linux:
  https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/local-management/as-a-service/linux/
- Tunnel run parameters:
  https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/configure-tunnels/run-parameters/
- Cloudflare Operator (alpha):
  https://github.com/adyanth/cloudflare-operator
