# homelab-infra

# 🏡 Home Infrastructure & GitOps Cluster

An automated, declarative home lab environment powered by Kubernetes and GitOps.

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.35.3-blue?logo=kubernetes)
![OS](https://img.shields.io/badge/OS-Talos_Linux-orange?logo=linux)
![CNI](https://img.shields.io/badge/CNI-Cilium-purple?logo=cilium)




## 💻 Hardware & Network

### Physical Nodes
| Hostname | Role | Specs | Network | Notes |
| :--- | :--- | :--- | :--- | :--- |
| `node-01` | Control Plane / Worker | Mac Mini (10GbE) | 10.0.30.x | |
| `node-02` | Control Plane / Worker | Mac Mini (10GbE) | 10.0.30.x | |
| `node-03` | Control Plane / Worker | Mac Mini (10GbE) | 10.0.30.x | |
| `ugreen` | NAS / Storage | DXP4800 Pro (10GbE) | 10.0.90.x | NFS share |

### Network Segmentation

| VLAN ID | Subnet / CIDR | Purpose | Routing Type | Notes |
| :--- | :--- | :--- | :--- | :--- |
| `10` | `10.0.10.0/24` | Main / Trusted | Routed | Laptops, desktops, trusted mobile devices. |
| `20` | `10.0.20.0/24` | Media | Routed | Streaming sticks, TVs, game consoles. |
| `30` | `10.0.30.0/24` | K8s Hosts / Nodes | Routed | Physical bare-metal OS IPs of the mini PCs. |
| `35` | `N/A (L2 Only)` | Pod Network | Non-Routed | L2 VLAN for pod traffic to separate it from node traffic. |
| `38` | `10.0.38.0/24` | BGP Peering | Routed (Transit) | BGP peering transit subnet connecting nodes to router. |
| `40` | `10.0.40.0/24` | Service IPs via BGP | BGP Advertised | Data Plane: Ingress and External Service VIPs. |
| `50` | `10.0.50.0/23` | IoT / Smart Home | Routed | Spans 10.0.50.1 to 10.0.51.254. |
| `60` | `10.0.60.0/24` | Guest Wi-Fi | Routed | Isolated internet-only access. |
| `75` | `10.0.75.0/24` | L2 Storage Data | Non-Routed | High-speed NFS/iSCSI storage traffic (Jumbo Frames / MTU 9000). |
| `80` | `10.0.80.0/24` | DMZ / Ingress | Routed | Public-facing reverse proxies and edge tunnels. |
| `90` | `10.0.90.0/24` | Infra Management | Routed | Switches, RouterOS management, and NAS Web UI. |
| `91` | `10.0.91.0/24` | Infra Services | Routed | Technitium DNS, Omada Controller, and core services. |

## 🛠️ Tech Stack

* **Operating System:** [Talos Linux](https://www.talos.dev/) (Immutable, API-driven)
* **Container Networking (CNI):** [Cilium](https://cilium.io/) with BGP Control Plane
* **Ingress Controller:** [Traefik](https://traefik.io/) + [cert-manager](https://cert-manager.io/) (Let's Encrypt DNS-01)
* **DNS Infrastructure:** [Technitium DNS](https://technitium.com/dns/) (Internal split-horizon)
* **GitOps Engine:** [FluxCD](https://fluxcd.io/)


## 📂 Repository Structure

```text

Repository based on Flux documentation https://fluxcd.io/flux/guides/repository-structure/#repository-structure

.
├── clusters/                                   # Flux cluster entry point & root manifests
│   └── production/
│       ├── apps.yaml                           # Kustomization pointing to application workloads
│       ├── gateway-api.yaml                    # Gateway API Custom Resource Definitions & setup
│       ├── infrastructure.yaml                 # Kustomization pointing to core infra controllers
│       ├── kustomization.yaml                  # Root kustomization for the production cluster
│       └── flux-system/                        # Flux controller components & Git repository sync config
│           ├── gotk-components.yaml            # Flux core controllers (source, kustomize, helm, notification)
│           ├── gotk-sync.yaml                  # GitRepository and Kustomization defining the root sync
│           └── kustomization.yaml              # Kustomization for the flux-system components
│
├── infrastructure/                             # Core cluster controllers & platform services
│   ├── base/                                   # Environment-agnostic platform components
│   │   └── gateway-api/
│   │       ├── gateway-api-v1.5.1.yaml         # Gateway API manifests
│   │       └── kustomization.yaml              # Kustomization for Gateway API base
│   ├── configs/                                # Environment-agnostic configurations & operators configs
│   │   ├── cert-manager/                       # Automated TLS certificate management configs
│   │   │   ├── cloudflare-api-token.yaml
│   │   │   ├── clusterissuer-production.yaml
│   │   │   ├── kustomization.yaml
│   │   │   ├── traefik-wildcard.yaml
│   │   │   └── wildcard-cert.yaml
│   │   ├── cnpg/                               # CloudNativePG configurations
│   │   │   └── kustomization.yaml
│   │   ├── local-path-provisioner/             # Local storage provisioner configs
│   │   │   └── kustomization.yaml
│   │   ├── traefik/                            # Edge ingress controller configurations
│   │   │   ├── dashboard-ingress.yaml
│   │   │   ├── kustomization.yaml
│   │   │   └── tlsStore.yaml
│   │   └── kustomization.yaml                  # Root kustomization for infrastructure configs
│   └── controllers/                            # Core infrastructure controller manifests & Helm releases
│       ├── cert-manager.yaml
│       ├── cloudnative-pg.yaml
│       ├── kustomization.yaml
│       ├── local-path-provisioner.yaml
│       ├── nfs-client.yaml
│       ├── traefik.yaml
│       └── traefik-values.yaml
│
├── apps/                                       # User-facing applications & stateful workloads
│   ├── base/                                   # Core application manifests (Deployments, Services, PVCs)
│   │   ├── cloudflare-tunnel/                  # Cloudflare tunnel base setup
│   │   │   ├── configmap.yaml
│   │   │   ├── deployment.yaml
│   │   │   ├── kustomization.yaml
│   │   │   └── namespace.yaml
│   │   ├── linkding/                           # Bookmark manager base setup
│   │   └── vaultwarden/                        # Password manager base setup
│   │       ├── deployment.yaml
│   │       ├── gateway.yaml
│   │       ├── httproute.yaml
│   │       ├── https-redirect.yaml
│   │       ├── kustomization.yaml
│   │       ├── namespace.yaml
│   │       ├── postgres-cluster.yaml
│   │       ├── pvc.yaml
│   │       ├── vaultwarden-db-secret.yaml
│   │       ├── vaultwarden-db-url.yaml
│   │       ├── vaultwarden-helmrelease.yaml
│   │       ├── vaultwarden-helmrepository.yaml
│   │       └── vw-secret.yaml
│   └── production/                             # Production overlays & environment-specific values
│       ├── cloudflare-tunnel/
│       │   ├── kustomization.yaml
│       │   └── tunnel-token.yaml
│       ├── vaultwarden/                        # Traefik IngressRoute & TLS cert binding overrides
│       │   ├── certificate.yaml
│       │   ├── ingressroute.yaml
│       │   └── kustomization.yaml
│       └── kustomization.yaml                  # Root kustomization for production apps
│
├── docs/                                       # Operational documentation & runbooks
│   ├── fix-cloudflare-traefik-gateway.md
│   ├── recent-cloudflare-traefik-vaultwarden-runbook.md
│   └── recent-cloudflare-traefik-vaultwarden-runbook.pdf
│
└── k8s-test/                                   # Temporary or ad-hoc cluster testing manifests
    ├── job.yaml
    ├── nfs-client-test.yaml
    └── network-test/
        ├── mtu-host-test.yaml
        ├── mtu-vlan75-p02.yaml
        ├── mtu-vlan75-p03.yaml
        ├── mtuNamespace.yaml
        └── test-bgp.yaml