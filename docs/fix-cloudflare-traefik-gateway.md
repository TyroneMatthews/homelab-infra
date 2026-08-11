# fix/cloudflare-traefik-gateway

## Branch
- Branch name: `fix/cloudflare-traefik-gateway`

## Purpose
Convert the current Cloudflare Tunnel setup into a modern Traefik Gateway API flow.

## Summary
This branch moves Vaultwarden from the existing Cilium Gateway API path into a Traefik-managed Gateway API flow, with Cloudflare Tunnel terminating at Traefik.

## Changes made
- Enabled `providers.kubernetesGateway.enabled: true` in `infrastructure/controllers/traefik-values.yaml`.
- Kept `providers.kubernetesCRD` and `providers.kubernetesIngress` enabled to preserve existing Traefik compatibility.
- Left `gateway.enabled: false` and created a managed `gatewayClass.name: traefik`.
- Updated `apps/base/cloudflare-tunnel/configmap.yaml` to target Traefik at `http://traefik.traefik.svc.cluster.local:80`.
- Changed `apps/base/vaultwarden/gateway.yaml` from `gatewayClassName: cilium` in `gateway-system` to `gatewayClassName: traefik` in the `vaultwarden` namespace.
- Fixed `apps/base/vaultwarden/httproute.yaml` to route to `vaultwarden-svc`.

## Production overlay
- `apps/production/vaultwarden/kustomization.yaml` currently imports `../../base/vaultwarden`.
- That base includes the `gateway.yaml` and `httproute.yaml` objects, so the Traefik Gateway API configuration is already part of the production overlay.
- The legacy `apps/production/vaultwarden/ingressroute.yaml` remains commented out because this deployment uses Cloudflare Tunnel + Gateway API instead of Traefik IngressRoute.

## Validation
- `kustomize build apps/base/vaultwarden` ✅
- `kustomize build apps/base/cloudflare-tunnel` ✅
- `kustomize build infrastructure/controllers` ✅

## Notes
- The Tunnel origin should be HTTP to Traefik, not HTTPS to the old Cilium gateway service.
- Vaultwarden TLS termination is handled by the Traefik Gateway HTTPS listener using `vaultwarden-tls-cert`.

## Troubleshooting guide

### 1. Verify Traefik values and Gateway entry points
- Check `infrastructure/controllers/traefik-values.yaml`.
- Confirm `providers.kubernetesGateway.enabled: true`.
- Confirm the chart uses `ports.web` and `ports.websecure`, not `ports.http` or `ports.https`:
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
- If Traefik reports missing entry points for port 80/443, the values file is the primary suspect.

### 2. Confirm Gateway API CRDs
- Run:
```bash
kubectl api-resources --api-group=gateway.networking.k8s.io
kubectl get crd gatewayclasses.gateway.networking.k8s.io gateways.gateway.networking.k8s.io httproutes.gateway.networking.k8s.io tlsroutes.gateway.networking.k8s.io referencegrants.gateway.networking.k8s.io grpcroutes.gateway.networking.k8s.io -o jsonpath='{range .items[*]}{.metadata.name} {range .spec.versions[*]}{.name}:{.served}:{.storage} {end}\n{end}'
```
- Make sure the cluster has supported `v1` versions for:
  - `GatewayClass`
  - `Gateway`
  - `HTTPRoute`
  - `TLSRoute`
  - `BackendTLSPolicy`
  - `ReferenceGrant`
  - `GRPCRoute`
- If versions are missing, install or upgrade the Gateway API CRDs before debugging Traefik.

### 3. Inspect the Gateway and HTTPRoute definitions
- Run:
```bash
kubectl get gateway,httproute,tlsroute -A
kubectl describe gateway -n vaultwarden public-gateway
kubectl describe httproute -n vaultwarden vaultwarden
```
- Confirm the Gateway listeners are:
  - `protocol: HTTP`, `port: 80`, `name: http`
  - `protocol: HTTPS`, `port: 443`, `name: https`
- Confirm `HTTPRoute` parentRefs match `public-gateway` and `sectionName: https`.
- Confirm the `backendRef` points to `vaultwarden` service port `80`.

### 4. Check Traefik logs for Gateway provider startup
- Run:
```bash
kubectl logs -n traefik deployment/traefik | grep -i gateway
```
- Look for:
  - `Starting provider *gateway.Provider`
  - `Creating in-cluster Provider client`
  - `cannot find entryPoint for Gateway`
- If the provider starts successfully and the Gateway is accepted, Traefik is configured correctly.

### 5. Render the Helm manifest for Traefik
- Run:
```bash
helm template traefik traefik/traefik --version 41.0.2 -f infrastructure/controllers/traefik-values.yaml --namespace traefik
```
- Confirm the rendered Service ports are only `80/web` and `443/websecure`.
- Confirm the rendered Deployment container ports are only `80/web` and `443/websecure` on the Traefik container.
- If duplicates remain, the values file still contains invalid keys.

### 6. Reconcile or restart Traefik
- If using Flux:
```bash
flux reconcile helmrelease traefik -n traefik
flux get helmreleases -n traefik
kubectl describe hr -n traefik traefik | tail -n 40
```
- If using kubectl directly:
```bash
kubectl rollout restart deployment/traefik -n traefik
```

### 7. Validate the Tunnel route
- Check Cloudflare Tunnel config in `apps/base/cloudflare-tunnel/configmap.yaml`:
```yaml
ingress:
  - hostname: vaultwarden.tmatthews.casa
    service: http://traefik.traefik.svc.cluster.local:80
  - service: http_status:404
```
- Confirm the `cloudflared` deployment is running and connected.
- Use a debug pod to test internal connectivity if the tunnel pod has no shell:
```bash
kubectl run -n cloudflared debug --rm -i --tty --image=alpine -- ash
apk add --no-cache curl
curl -v -H 'Host: vaultwarden.tmatthews.casa' http://traefik.traefik.svc.cluster.local:80/
```

### 8. External validation
- Test the public hostname via Cloudflare:
```bash
curl -v https://vaultwarden.tmatthews.casa/
```
- If the hostname is not reachable, confirm Cloudflare Tunnel DNS and tunnel route configuration on the Cloudflare dashboard.

## Next steps
- Deploy the branch and confirm the Traefik Gateway status object is accepted.
- Verify `Gateway` and `HTTPRoute` status conditions in the `vaultwarden` namespace.
- Confirm Cloudflare Tunnel can reach Traefik successfully and route `vaultwarden.tmatthews.casa`.
- After validation, remove any unused Cilium Gateway objects if still present.

## PR checklist
- [ ] Confirm `providers.kubernetesGateway.enabled` is set in `infrastructure/controllers/traefik-values.yaml`.
- [ ] Confirm `apps/base/vaultwarden/gateway.yaml` uses `gatewayClassName: traefik` in namespace `vaultwarden`.
- [ ] Confirm `apps/base/vaultwarden/httproute.yaml` routes to `vaultwarden-svc`.
- [ ] Confirm `apps/base/cloudflare-tunnel/configmap.yaml` targets Traefik at `http://traefik.traefik.svc.cluster.local:80`.
- [ ] Run `kustomize build apps/base/vaultwarden` and `kustomize build apps/base/cloudflare-tunnel`.
- [ ] Optional: review `apps/production/vaultwarden/kustomization.yaml` to ensure the new objects are included.
- [ ] Open PR from `fix/cloudflare-traefik-gateway` and include this doc in the description.
