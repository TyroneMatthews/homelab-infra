# Cloudflare Tunnel, Traefik, and Vaultwarden Runbook

Date: 2026-08-19
Repository: homelab-infra
Hostname: vaultwarden.tmatthews.casa

## Architecture

Cloudflare Tunnel connects to the internal Traefik Service over HTTPS. Traefik's Gateway API listener terminates TLS and routes the hostname to the Vaultwarden Service. The Vaultwarden Service exposes port 80 and targets the application container on port 8080.

```text
Cloudflare Tunnel
  -> https://traefik.traefik.svc.cluster.local:443
  -> Traefik Gateway public-gateway, HTTPS listener
  -> Service vaultwarden:80
  -> Vaultwarden pod:8080
```

## Recent causes and fixes

### 1. Invalid Traefik Helm port keys

Traefik Helm chart 41.0.2 expects `ports.web` and `ports.websecure`. Using `ports.http` or `ports.https` can produce missing entrypoints or duplicate Service and Deployment ports during an upgrade.

Use:

```yaml
ports:
  web:
    port: 80
    expose:
      default: true
    exposedPort: 80
  websecure:
    asDefault: true
    port: 443
    expose:
      default: true
    exposedPort: 443
```

### 2. HTTPRoute backend name mismatch

The live Helm-managed Service is named `vaultwarden`, not `vaultwarden-svc`. The route must reference the Service port, which is 80. Port 8080 is the container targetPort and must not be used in the HTTPRoute.

```yaml
backendRefs:
  - name: vaultwarden
    kind: Service
    port: 80
```

### 3. Cloudflare origin hostname mismatch

The tunnel connects to Traefik using the internal Service DNS name, but the Gateway and certificate use `vaultwarden.tmatthews.casa`. Without explicit origin settings, HTTPS origin validation or host-based routing can fail and produce a 502.

```yaml
service: https://traefik.traefik.svc.cluster.local:443
originRequest:
  originServerName: vaultwarden.tmatthews.casa
  httpHostHeader: vaultwarden.tmatthews.casa
```

### 4. Restart command used the wrong namespace

`cloudflared` is deployed in the `cloudflared` namespace. The namespace is required:

```bash
kubectl rollout restart deployment/cloudflared -n cloudflared
kubectl rollout status deployment/cloudflared -n cloudflared --timeout=120s
```

### 5. Restricted PodSecurity warning

The cluster reported missing restricted-profile settings for the tunnel container. Add these settings to the container:

```yaml
securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault
```

## Recovery and verification commands

Run from the repository root.

### Inspect the active path

```bash
kubectl get pods -n cloudflared -o wide
kubectl get deployment cloudflared -n cloudflared
kubectl get svc -n traefik -o wide
kubectl get gateway,httproute -n vaultwarden -o wide
kubectl get endpointslice -n vaultwarden \\
  -l kubernetes.io/service-name=vaultwarden -o wide
```

### Check Gateway API status

```bash
kubectl describe gateway public-gateway -n vaultwarden
kubectl describe httproute vaultwarden -n vaultwarden
kubectl describe httproute vaultwarden-http-redirect -n vaultwarden
```

Expected conditions include `Accepted=True` and `ResolvedRefs=True`. The Gateway should also show `Programmed=True` and address `192.168.40.15` in this environment.

### Check logs

```bash
kubectl logs -n cloudflared deployment/cloudflared --since=20m \\
  | grep -Ei 'ERR|error|502|503|origin|certificate|tls|connection refused'

kubectl logs -n traefik deployment/traefik --since=20m \\
  | grep -Ei 'error|gateway|vaultwarden|502|tls|http'
```

### Test the origin directly

```bash
curl -sS -D - -o /dev/null \\
  -H 'Host: vaultwarden.tmatthews.casa' \\
  http://192.168.40.15/

curl -k -sS -D - -o /dev/null \\
  --resolve vaultwarden.tmatthews.casa:443:192.168.40.15 \\
  https://vaultwarden.tmatthews.casa/
```

Expected results: HTTP returns `301` to HTTPS; direct HTTPS returns `200` from Vaultwarden.

### Test through Cloudflare

```bash
curl -sS -D - -o /dev/null \\
  https://vaultwarden.tmatthews.casa/
```

Expected result: `HTTP/2 200`. A 502 means Cloudflare cannot successfully complete the origin request; check tunnel logs first, then test direct Traefik HTTPS.

### Reconcile Flux after committing changes

```bash
flux reconcile kustomization apps -n flux-system --with-source
flux reconcile helmrelease traefik -n traefik
flux reconcile helmrelease vaultwarden -n vaultwarden
flux get kustomizations -A
flux get helmreleases -A
```

### Validate manifests before pushing

```bash
kustomize build apps/production >/tmp/apps-production.yaml
kubectl apply --dry-run=server -k apps/production
 git diff --check
```

Remove the leading space before `git` if copying the command exactly from a shell that rejects it.

## Known-good final state

- Cloudflare Tunnel replicas: 2/2 Ready.
- Traefik Gateway provider started.
- `public-gateway`: Accepted and Programmed.
- Vaultwarden HTTPRoute: Accepted and ResolvedRefs.
- Vaultwarden Service: `vaultwarden:80 -> pod:8080`.
- Public hostname: HTTP/2 200.
- Cloudflare tunnel origin: internal Traefik HTTPS with Vaultwarden SNI and Host header.
