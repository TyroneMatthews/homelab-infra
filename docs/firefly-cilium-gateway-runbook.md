# Firefly Cilium Gateway Runbook

## Purpose

This runbook documents the Firefly internal-only deployment built on Cilium Gateway API. Traefik continues to serve existing workloads such as Vaultwarden. Firefly and the Firefly Data Importer use the shared Cilium Gateway and are not configured in the Cloudflare Tunnel.

## Architecture

```text
Internal DNS
  firefly.tmatthews.casa             -> 192.168.40.1
  firefly-importer.tmatthews.casa    -> 192.168.40.1
             |
             v
Cilium LoadBalancer Service
  cilium-gateway-internal-gateway
             |
             v
Cilium Gateway API
  gateway-system/internal-gateway
  HTTPS :443, wildcard TLS termination
             |
             +--> firefly/firefly HTTPRoute
             |      -> firefly-iii Service :80
             |
             +--> firefly/firefly-importer HTTPRoute
                    -> firefly-importer Service :80
```

The Gateway address is allocated by `CiliumLoadBalancerIPPool/vlan40-service-pool`, which currently covers `192.168.40.1` through `192.168.40.30`. The address is private LAN space. Kubernetes displays it under `EXTERNAL-IP` because the Service type is `LoadBalancer`; it is not public by itself.

## Repository Layout

- `infrastructure/base/cilium-gateway/`: `gateway-system`, the shared Cilium Gateway, and the Cilium LoadBalancer IP pool.
- `infrastructure/configs/traefik/referencegrant-cilium-gateway.yaml`: permits the shared Gateway to use the wildcard Secret stored in `traefik`.
- `apps/base/firefly/helmrelease.yaml`: Firefly III application HelmRelease and database/application settings.
- `apps/base/firefly/httproute.yaml`: Firefly route to `firefly-iii:80`.
- `apps/base/firefly/importer-helmrelease.yaml`: importer HelmRelease and internal HTTPRoute.
- `apps/base/firefly/importer-sessions-pvc.yaml`: RWX NFS storage for importer Laravel sessions.
- `apps/base/firefly/kustomization.yaml`: applies the `firefly` namespace transformer, including encrypted Secrets.

## Prerequisites

- Cilium is installed and healthy.
- Cilium Gateway API is enabled.
- Gateway API CRDs are installed.
- Cilium LB-IPAM is enabled.
- The `ugreen-nfs-sc` StorageClass is available for importer sessions.
- The wildcard certificate Secret `traefik/wildcard-tmatthews-casa-tls` exists.
- Internal DNS resolves both Firefly names to `192.168.40.1`.
- Flux has SOPS decryption configured for encrypted application Secrets.

## Initial Deployment

### 1. Enable the Cilium GatewayClass

When Traefik already owns a GatewayClass, Cilium's default `auto` behavior may skip creating its own class. Cilium must be installed with explicit GatewayClass creation:

```bash
helm upgrade cilium cilium/cilium \
  --version 1.18.12 \
  --namespace kube-system \
  --reuse-values \
  --set-string gatewayAPI.gatewayClass.create=true \
  --wait
```

For a rebuild, preserve this value in the Cilium install configuration:

```yaml
gatewayAPI:
  enabled: true
  gatewayClass:
    create: "true"
```

Verify:

```bash
kubectl get gatewayclass cilium -o wide
```

Expected controller:

```text
io.cilium/gateway-controller
```

### 2. Apply or reconcile infrastructure

The shared Gateway base is included by `infrastructure/configs/kustomization.yaml`. Flux normally applies it through the infrastructure configuration Kustomization. For a direct test:

```bash
kubectl apply -k infrastructure/base/cilium-gateway
```

The wildcard Secret remains in `traefik`. The `ReferenceGrant` in `infrastructure/configs/traefik` authorizes only `gateway-system/internal-gateway` to reference that Secret. Do not issue a second wildcard certificate unless there is a deliberate reason to duplicate it.

### 3. Reconcile applications

```bash
flux reconcile kustomization apps --with-source
flux reconcile helmrelease firefly-iii -n firefly --with-source
flux reconcile helmrelease firefly-importer -n firefly --with-source
```

The Firefly base Kustomization includes `namespace: firefly`. This is required because the encrypted Secret resources do not contain `metadata.namespace`; `namespace.yaml` only creates the namespace and does not assign resource namespaces.

## Internal Access

Create internal DNS records:

```text
firefly.tmatthews.casa          A 192.168.40.1
firefly-importer.tmatthews.casa A 192.168.40.1
```

Access:

```text
https://firefly.tmatthews.casa
https://firefly-importer.tmatthews.casa
```

For a temporary client test, add to `/etc/hosts`:

```text
192.168.40.1 firefly.tmatthews.casa firefly-importer.tmatthews.casa
```

The hostname is required for wildcard TLS SNI and HTTPRoute hostname matching. Testing only by IP will not exercise the configured route correctly.

Firefly is not in `apps/base/cloudflare-tunnel/configmap.yaml`. Keep it out of Cloudflare Tunnel if it must remain internal-only.

## Normal Health Checks

```bash
kubectl get gatewayclass cilium -o wide
kubectl get gateway -n gateway-system internal-gateway -o wide
kubectl get svc -n gateway-system cilium-gateway-internal-gateway -o wide
kubectl get httproute -n firefly firefly firefly-importer
kubectl describe httproute -n firefly firefly
kubectl describe httproute -n firefly firefly-importer
kubectl get pods,svc,endpointslice -n firefly -o wide
kubectl get pvc -n firefly firefly-importer-sessions
```

Expected conditions:

```text
Gateway:       PROGRAMMED=True
HTTPRoutes:    Accepted=True, ResolvedRefs=True
Importer pod:  Running and Ready 1/1
Session PVC:   Bound, RWX, ugreen-nfs-sc
```

End-to-end tests from a running Firefly pod:

```bash
kubectl exec -n firefly deploy/firefly-iii -- \
  curl -skS --connect-timeout 10 \
  --resolve firefly.tmatthews.casa:443:10.97.146.45 \
  -D - https://firefly.tmatthews.casa/ -o /dev/null

kubectl exec -n firefly deploy/firefly-iii -- \
  curl -skS --connect-timeout 10 \
  --resolve firefly-importer.tmatthews.casa:443:10.97.146.45 \
  -D - https://firefly-importer.tmatthews.casa/ -o /dev/null
```

A `302` to `/login` or `/token` is a successful routing result. `server: envoy` confirms the request passed through Cilium Gateway.

## OAuth Configuration

The importer uses two related URLs:

```yaml
fireflyiii:
  url: https://firefly.tmatthews.casa
  vanityUrl: https://firefly.tmatthews.casa
```

Both must point to Firefly's browser-reachable hostname. The importer itself is reached at `firefly-importer.tmatthews.casa`, and its callback is automatically:

```text
https://firefly-importer.tmatthews.casa/callback
```

Create or edit the OAuth client in Firefly and set its redirect URI exactly to the callback above. Do not set the redirect URI to the Firefly hostname or to an internal Kubernetes Service name.

The importer currently leaves the access token empty so the user can complete the OAuth flow interactively. A preconfigured token can be supplied later through a Secret referenced by `fireflyiii.auth.existingSecret`.

## OAuth Session Persistence

The importer originally used pod-local file sessions. A pod replacement during the OAuth redirect caused the callback to find no stored `state`, producing:

```text
The "state" returned from your server doesn't match the state that was sent.
```

The final configuration persists Laravel file sessions on the RWX NFS PVC:

```yaml
SESSION_DRIVER: file
```

The PVC is mounted at:

```text
/var/www/html/storage/framework/sessions
```

The importer also uses a stable cookie configuration. After changing session settings, bump the `podAnnotations.config-revision` value so the Deployment rolls out and loads the new environment.

When retrying OAuth after a session change:

1. Close old importer tabs.
2. Delete cookies for `firefly.tmatthews.casa` and `firefly-importer.tmatthews.casa`.
3. Open `https://firefly-importer.tmatthews.casa/token`.
4. Enter the OAuth client ID again.
5. Complete authorization in Firefly.

## Troubleshooting

### `GatewayClass/cilium` is missing

Check:

```bash
kubectl get gatewayclass
kubectl -n kube-system get cm cilium-config -o yaml | grep -E 'enable-gateway-api|kube-proxy-replacement'
helm get values cilium -n kube-system --all -o yaml | grep -n -A8 'gatewayAPI:'
```

If `gatewayAPI.gatewayClass.create` is `auto`, upgrade Cilium with:

```bash
helm upgrade cilium cilium/cilium \
  --version 1.18.12 \
  --namespace kube-system \
  --reuse-values \
  --set-string gatewayAPI.gatewayClass.create=true \
  --wait
```

The chart requires the value to be passed as a string. Do not use `--set` with a boolean for this version.

### Gateway is `ListenersNotValid` or `RefNotPermitted`

Check the ReferenceGrant and wildcard Secret:

```bash
kubectl get referencegrant -n traefik wildcard-cilium-gateway
kubectl get secret -n traefik wildcard-tmatthews-casa-tls
kubectl describe gateway -n gateway-system internal-gateway
```

Expected listener conditions:

```text
ResolvedRefs=True
Accepted=True
Programmed=True
```

### HTTPRoute is `BackendNotFound`

Check the actual Helm-created Service names:

```bash
kubectl get svc -n firefly
kubectl get endpointslice -n firefly
```

The Firefly route must target:

```yaml
backendRefs:
  - name: firefly-iii
    port: 80
```

The importer route targets `firefly-importer:80`.

### `no healthy upstream`

Check whether the importer pod is running:

```bash
kubectl get pod -n firefly -l app.kubernetes.io/instance=firefly-importer
kubectl get endpointslice -n firefly -l kubernetes.io/service-name=firefly-importer
kubectl logs -n firefly deploy/firefly-importer --since=15m
```

A missing `firefly-importer-auth` Secret caused this during the initial deployment. The final manifest does not reference that Secret. If authentication is enabled later, create the Secret before reconciling.

### OAuth 404 at `/oauth/authorize`

The importer must redirect OAuth to Firefly, not itself:

```yaml
fireflyiii:
  url: https://firefly.tmatthews.casa
  vanityUrl: https://firefly.tmatthews.casa
```

If the browser shows:

```text
https://firefly-importer.tmatthews.casa/oauth/authorize
```

`vanityUrl` is wrong. Reconcile the HelmRelease and start a new flow.

### OAuth state mismatch

Check the importer session configuration and PVC:

```bash
kubectl exec -n firefly deploy/firefly-importer -- \
  php artisan config:show session | grep -E 'driver|cookie|domain|secure'
kubectl get pvc -n firefly firefly-importer-sessions
kubectl exec -n firefly deploy/firefly-importer -- \
  mount | grep storage/framework/sessions
```

The expected session driver is `file`, and the session directory must be mounted from the RWX PVC. If the pod was recently replaced, restart the OAuth flow after confirming the PVC is mounted.

### DNS or ping fails

Ping is not a valid Gateway test and may be blocked or routed unexpectedly. Test HTTPS with the correct hostname instead:

```bash
curl -vk https://firefly.tmatthews.casa/
curl -vk https://firefly-importer.tmatthews.casa/
```

If in-cluster HTTPS works but a workstation cannot connect, check LAN/VPN routing and firewall access from the client network to `192.168.40.1`.

## Recovery Checklist

```bash
kubectl get gatewayclass cilium
kubectl get gateway -n gateway-system internal-gateway -o wide
kubectl get svc -n gateway-system cilium-gateway-internal-gateway -o wide
kubectl get httproute -n firefly firefly firefly-importer
kubectl get pods,svc,endpointslice -n firefly
kubectl get pvc -n firefly firefly-importer-sessions
flux get kustomization apps
flux get helmrelease -n firefly firefly-iii firefly-importer
```

Then reconcile in order:

```bash
flux reconcile kustomization infra-configs --with-source
flux reconcile kustomization apps --with-source
flux reconcile helmrelease firefly-iii -n firefly --with-source
flux reconcile helmrelease firefly-importer -n firefly --with-source
```

Do not use plain `kubectl apply -k apps/production/firefly` for encrypted SOPS Secrets; local kubectl does not decrypt them. Use Flux for the production application overlay.

## Final State

The final working state is:

```text
Cilium GatewayClass       Accepted=True
internal-gateway          Programmed=True at 192.168.40.1
Wildcard TLS              traefik/wildcard-tmatthews-casa-tls
LB-IPAM                   vlan40-service-pool, 192.168.40.1-30
Firefly route             Accepted=True, ResolvedRefs=True
Importer route            Accepted=True, ResolvedRefs=True
Importer session storage  RWX NFS PVC firefly-importer-sessions
Cloudflare Tunnel         Vaultwarden only; Firefly remains internal-only
```
