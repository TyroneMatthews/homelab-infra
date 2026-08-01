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

## Next steps
- Deploy the branch and confirm the Traefik Gateway status object is accepted.
- Verify `Gateway` and `HTTPRoute` status conditions in `vaultwarden` namespace.
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
