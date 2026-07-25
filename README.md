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
| `node-01` | Control Plane / Worker | Mac Mini (10GbE) | 192.168.30.11 | died/needs to replaced |
| `node-02` | Control Plane / Worker | Mac Mini (10GbE) | 192.168.30.12 | |
| `node-03` | Control Plane / Worker | Mac Mini (10GbE) | 192.168.30.13 | |
| `ugreen` | NAS / Storage | DXP4800 Pro (10GbE) | 192.168.90.250 | NFS share |

### Network Segmentation

| VLAN ID | Subnet / CIDR | Purpose | Routing Type | Notes |
| :--- | :--- | :--- | :--- | :--- |
| `10` | `192.168.10.0/24` | Main / Trusted | Routed | Laptops, desktops, trusted mobile devices. |
| `20` | `192.168.20.0/24` | Media | Routed | Streaming sticks, TVs, game consoles. |
| `30` | `192.168.30.0/24` | K8s Hosts / Nodes | Routed | Physical bare-metal OS IPs of the mini PCs. |
| `35` | `N/A (L2 Only)` | Pod Network | Non-Routed | L2 VLAN for pod traffic to separate it from node traffic. |
| `38` | `192.168.38.0/24` | BGP Peering | Routed (Transit) | BGP peering transit subnet connecting nodes to router. |
| `40` | `192.168.40.0/24` | Service IPs via BGP | BGP Advertised | Data Plane: Ingress and External Service VIPs. |
| `50` | `192.168.50.0/23` | IoT / Smart Home | Routed | Spans 192.168.50.1 to 192.168.51.254. |
| `60` | `192.168.60.0/24` | Guest Wi-Fi | Routed | Isolated internet-only access. |
| `75` | `192.168.75.0/24` | L2 Storage Data | Non-Routed | High-speed NFS/iSCSI storage traffic (Jumbo Frames / MTU 9000). |
| `80` | `192.168.80.0/24` | DMZ / Ingress | Routed | Public-facing reverse proxies and edge tunnels. |
| `90` | `192.168.90.0/24` | Infra Management | Routed | Switches, RouterOS management, and NAS Web UI. |
| `91` | `192.168.91.0/24` | Infra Services | Routed | Technitium DNS, Omada Controller, and core services. |

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
├── clusters/
│   └── production/             # Flux cluster entry point & root manifests
│       ├── flux-system/        # Flux controller components & Git repository sync config
│       ├── infrastructure.yaml # Kustomization pointing to core infra controllers
│       └── apps.yaml           # Kustomization pointing to application workloads
│
├── infrastructure/             # Core cluster controllers & platform services
│   ├── base/                   # Environment-agnostic Helm releases & repositories
│   │   ├── cert-manager/       # Automated TLS certificate management
│   │   └── traefik/            # Edge ingress controller
│   ├── production/             # Environment-specific overrides & custom resources
│   │   ├── cert-manager/       # Let's Encrypt ClusterIssuers & wildcard cert definitions
│   │   └── traefik/            # Traefik Helm values, TLS stores & dashboard routes
│   └── staging/                # Staging overlays (prepared for future expansion)
│
└── apps/                       # User-facing applications & stateful workloads
    ├── base/                   # Core application manifests (Deployments, Services, PVCs)
    │   ├── linkding/           # Bookmark manager base setup
    │   └── vaultwarden/        # Password manager base setup
    ├── production/             # Production overlays (IngressRoutes, Certs, custom routes)
    │   └── vaultwarden/        # Traefik IngressRoute & TLS cert binding
    └── staging/                # Staging application overlays
