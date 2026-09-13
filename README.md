# homelab-infra

# 🏡 Home Infrastructure & GitOps Cluster

An automated, declarative home lab environment powered by Kubernetes, Talos Linux, Cilium, and GitOps.

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.35.4-326CE5?logo=kubernetes\&logoColor=white)](https://kubernetes.io/)
[![Talos Linux](https://img.shields.io/badge/Talos%20Linux-v1.12.9-FF7300?logo=linux\&logoColor=white)](https://www.talos.dev/)
[![Cilium](https://img.shields.io/badge/Cilium-v1.18.12-F8C517?logo=cilium\&logoColor=black)](https://cilium.io/)
[![FluxCD](https://img.shields.io/badge/FluxCD-GitOps-5468FF?logo=flux\&logoColor=white)](https://fluxcd.io/)
[![Gateway API](https://img.shields.io/badge/Gateway%20API-v1.5.1-326CE5?logo=kubernetes\&logoColor=white)](https://gateway-api.sigs.k8s.io/)

---

## 💻 Hardware & Network

### Physical Nodes

| Hostname  | Role                   | Specs               | Network     | Notes     |
| :-------- | :--------------------- | :------------------ | :---------- | :-------- |
| `node-01` | Control Plane / Worker | Mac Mini (10GbE)    | `10.0.30.x` |   Died    |
| `node-02` | Control Plane / Worker | Mac Mini (10GbE)    | `10.0.30.x` |           |
| `node-03` | Control Plane / Worker | Mac Mini (10GbE)    | `10.0.30.x` |           |
| `ugreen`  | NAS / Storage          | DXP4800 Pro (10GbE) | `10.0.90.x` | NFS share |

### Network Segmentation

| VLAN ID | Subnet / CIDR   | Purpose             | Routing Type     | Notes                                                           |
| :------ | :-------------- | :------------------ | :--------------- | :-------------------------------------------------------------- |
| `30`    | `10.0.30.0/24`  | K8s Hosts / Nodes   | Routed           | Physical bare-metal OS IPs of the mini PCs.                     |
| `35`    | `N/A (L2 Only)` | Pod Network         | Non-Routed       | L2 VLAN for pod traffic to separate it from node traffic.       |
| `38`    | `10.0.38.0/24`  | BGP Peering         | Routed (Transit) | BGP peering transit subnet connecting nodes to router.          |
| `40`    | `10.0.40.0/24`  | Service IPs via BGP | BGP Advertised   | Data Plane: Ingress and External Service VIPs.                  |
| `75`    | `10.0.75.0/24`  | L2 Storage Data     | Non-Routed       | High-speed NFS/iSCSI storage traffic (Jumbo Frames / MTU 9000). |

---

## 🛠️ Tech Stack

### 🖥️ Kubernetes Platform

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.35.4-326CE5?logo=kubernetes\&logoColor=white)](https://kubernetes.io/)
[![Talos](https://img.shields.io/badge/Talos%20Linux-v1.12.9-FF7300?logo=linux\&logoColor=white)](https://www.talos.dev/)
[![Flux](https://img.shields.io/badge/FluxCD-GitOps-5468FF?logo=flux\&logoColor=white)](https://fluxcd.io/)
[![Kustomize](https://img.shields.io/badge/Kustomize-Kubernetes-326CE5?logo=kubernetes\&logoColor=white)](https://kustomize.io/)

| Technology      | Role                                              |
| :-------------- | :------------------------------------------------ |
| **Talos Linux** | Immutable, API-driven Kubernetes operating system |
| **Kubernetes**  | Container orchestration platform                  |
| **FluxCD**      | GitOps continuous delivery and reconciliation     |
| **Kustomize**   | Declarative Kubernetes configuration and overlays |

---

### 🌐 Networking & Ingress

[![Cilium](https://img.shields.io/badge/Cilium-v1.18.12-F8C517?logo=cilium\&logoColor=black)](https://cilium.io/)
[![Gateway API](https://img.shields.io/badge/Gateway%20API-v1.5.1-326CE5?logo=kubernetes\&logoColor=white)](https://gateway-api.sigs.k8s.io/)
[![Traefik](https://img.shields.io/badge/Traefik-Gateway%20%26%20Ingress-24A1C1?logo=traefikproxy\&logoColor=white)](https://traefik.io/)
[![Cloudflare](https://img.shields.io/badge/Cloudflare-Tunnel-F38020?logo=cloudflare\&logoColor=white)](https://www.cloudflare.com/)

| Technology             | Role                                                      |
| :--------------------- | :-------------------------------------------------------- |
| **Cilium**             | Kubernetes CNI, eBPF networking, network policy, and BGP  |
| **Cilium Gateway API** | Internal application gateway                              |
| **Traefik**            | External application gateway / ingress                    |
| **Gateway API**        | Kubernetes-native HTTP routing                            |
| **BGP**                | Advertises Kubernetes service IPs to the physical network |
| **Cloudflare Tunnel**  | Secure external access without direct inbound exposure    |

---

### 🔐 TLS & DNS

[![cert-manager](https://img.shields.io/badge/cert--manager-TLS-1E88E5?logo=kubernetes\&logoColor=white)](https://cert-manager.io/)
[![Let's Encrypt](https://img.shields.io/badge/Let's%20Encrypt-Certificates-003A70?logo=letsencrypt\&logoColor=white)](https://letsencrypt.org/)
[![Cloudflare](https://img.shields.io/badge/Cloudflare-DNS-F38020?logo=cloudflare\&logoColor=white)](https://www.cloudflare.com/)
[![Technitium](https://img.shields.io/badge/Technitium-DNS-333333?logo=dns\&logoColor=white)](https://technitium.com/dns/)
[![ExternalDNS](https://img.shields.io/badge/ExternalDNS-Kubernetes-326CE5?logo=kubernetes\&logoColor=white)](https://kubernetes-sigs.github.io/external-dns/)

| Technology               | Role                                       |
| :----------------------- | :----------------------------------------- |
| **cert-manager**         | Automated certificate issuance and renewal |
| **Let's Encrypt**        | Public certificate authority               |
| **Cloudflare DNS**       | DNS-01 challenge provider                  |
| **ExternalDNS**          | Automated DNS record management            |
| **Technitium DNS**       | Internal DNS / split-horizon DNS           |
| **Wildcard Certificate** | `*.tmatthews.casa`                         |

---

### 💾 Storage

[![NFS](https://img.shields.io/badge/Storage-NFS-555555?logo=linux\&logoColor=white)](https://en.wikipedia.org/wiki/Network_File_System)
[![Backblaze](https://img.shields.io/badge/Backblaze-B2-E21E26?logo=backblaze\&logoColor=white)](https://www.backblaze.com/cloud-storage)

| Technology                 | Role                                |
| :------------------------- | :---------------------------------- |
| **UGREEN NAS**             | Primary network storage             |
| **NFS**                    | Shared persistent storage           |
| **NFS Client Provisioner** | Dynamic Kubernetes PV provisioning  |
| **Local Path Provisioner** | Node-local persistent storage       |
| **Backblaze B2**           | Off-site object storage for backups |

---

### 🗄️ Databases & Disaster Recovery

[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-18-4169E1?logo=postgresql\&logoColor=white)](https://www.postgresql.org/)
[![CloudNativePG](https://img.shields.io/badge/CloudNativePG-PostgreSQL-336791?logo=postgresql\&logoColor=white)](https://cloudnative-pg.io/)

| Technology              | Role                                |
| :---------------------- | :---------------------------------- |
| **PostgreSQL**          | Application database                |
| **CloudNativePG**       | Kubernetes PostgreSQL operator      |
| **Barman Cloud Plugin** | PostgreSQL backup to object storage |
| **Backblaze B2**        | Off-site PostgreSQL backup storage  |

Critical PostgreSQL workloads are backed up to Backblaze B2 and restore procedures are tested using manifests under `k8s-test/`.

---

### 📦 Applications

[![Vaultwarden](https://img.shields.io/badge/Vaultwarden-Password%20Manager-175DDC?logo=bitwarden\&logoColor=white)](https://github.com/dani-garcia/vaultwarden)
[![Firefly III](https://img.shields.io/badge/Firefly%20III-Personal%20Finance-8B5CF6?logo=firefly\&logoColor=white)](https://www.firefly-iii.org/)
[![Linkding](https://img.shields.io/badge/Linkding-Bookmarks-333333?logo=bookmark\&logoColor=white)](https://github.com/sissbruecker/linkding)

| Application           | Purpose                                  |
| :-------------------- | :--------------------------------------- |
| **Vaultwarden**       | Self-hosted password manager             |
| **Firefly III**       | Personal finance management              |
| **Linkding**          | Self-hosted bookmark manager             |
| **Cloudflare Tunnel** | External access to selected applications |

---

## 🔄 GitOps Architecture

The repository follows a declarative GitOps workflow:

```text
                    ┌──────────────────────┐
                    │      Git Repository  │
                    │     homelab-infra    │
                    └──────────┬───────────┘
                               │
                               │ Git
                               ▼
                    ┌──────────────────────┐
                    │       FluxCD         │
                    │                      │
                    │  Source Controller   │
                    │  Kustomize Controller│
                    │  Helm Controller     │
                    └──────────┬───────────┘
                               │
                               │ Reconcile
                               ▼
                    ┌──────────────────────┐
                    │   Talos Kubernetes   │
                    │                      │
                    │ Infrastructure       │
                    │        ↓             │
                    │ Applications         │
                    └──────────────────────┘
```

Git is the source of truth for the desired state of the cluster.

---

## 🌐 Traffic Architecture

External traffic enters through Cloudflare Tunnel and is routed through Traefik:

```text
Internet
   │
   ▼
Cloudflare
   │
   ▼
Cloudflare Tunnel
   │
   ▼
Traefik
External Gateway
   │
   ▼
Kubernetes Services
   │
   ▼
Applications
```

Internal applications can use the Cilium Gateway:

```text
Internal Network
       │
       ▼
Cilium Gateway
192.168.40.1
       │
       ▼
Kubernetes Services
       │
       ▼
Applications
```

Cilium advertises service IPs using BGP:

```text
┌───────────────────┐
│ Kubernetes Nodes  │
│                   │
│ Cilium BGP        │
│ ASN 65251         │
└─────────┬─────────┘
          │
          │ BGP
          │
          ▼
┌───────────────────┐
│ MikroTik RB5009   │
│                   │
│ ASN 65250         │
└───────────────────┘
```

---

## 🔐 Secret Management

Secrets are encrypted before being committed to Git using SOPS and age.

```text
Secret
  │
  ▼
SOPS
  │
  ▼
age encryption
  │
  ▼
Encrypted YAML
  │
  ▼
Git
  │
  ▼
FluxCD
  │
  ▼
SOPS decryption
  │
  ▼
Kubernetes Secret
```

Plain-text secrets should never be committed to the repository.

---

## 📂 Repository Structure

The repository structure follows the FluxCD recommended separation between cluster configuration, infrastructure, and applications.

```text
.
├── # Talos Cluster Rebuild Runbook.md
├── README.md
├── apps
│   ├── base
│   │   ├── cloudflare-tunnel
│   │   ├── firefly
│   │   ├── linkding
│   │   └── vaultwarden
│   └── production
│       ├── cloudflare-tunnel
│       ├── firefly
│       └── vaultwarden
│
├── clusters
│   └── production
│       ├── apps.yaml
│       ├── gateway-api.yaml
│       ├── infrastructure.yaml
│       ├── kustomization.yaml
│       └── flux-system
│
├── infrastructure
│   ├── base
│   │   ├── cilium-gateway
│   │   └── gateway-api
│   ├── configs
│   │   ├── cert-manager
│   │   ├── cnpg
│   │   ├── external-dns
│   │   ├── local-path-provisioner
│   │   └── traefik
│   ├── controllers
│   │   ├── cert-manager.yaml
│   │   ├── cloudnative-pg.yaml
│   │   ├── external-dns.yaml
│   │   ├── local-path-provisioner.yaml
│   │   ├── nfs-client.yaml
│   │   ├── traefik.yaml
│   │   └── traefik-values.yaml
│   └── secrets
│
├── docs
│   ├── firefly-cilium-gateway-runbook.md
│   ├── fix-cloudflare-traefik-gateway.md
│   └── recent-cloudflare-traefik-vaultwarden-runbook.md
│
└── k8s-test
    ├── cnpg-restore.yaml
    ├── dns-test.yaml
    ├── nfs-client-test.yaml
    ├── vaultwarden-restore.yaml
    └── network-test
```

---

## 📚 Documentation

Operational procedures and troubleshooting documentation are maintained under [`docs/`](docs/).

Current documentation includes:

* Talos cluster rebuild procedures
* Cilium Gateway configuration
* Cloudflare Tunnel troubleshooting
* Traefik Gateway troubleshooting
* Vaultwarden ingress and TLS
* Firefly III networking
* PostgreSQL restore testing
* NFS and storage testing
* BGP and network testing

---

## 🎯 Learning Goals

This homelab is primarily a learning environment focused on:

* Kubernetes administration
* Talos Linux
* GitOps and FluxCD
* Kubernetes networking
* Cilium and eBPF
* Gateway API
* BGP
* TLS automation
* DNS automation
* PostgreSQL administration
* Kubernetes storage
* Database backup and disaster recovery
* Infrastructure as Code
* Secure secret management

The goal is not simply to run applications, but to build infrastructure that is **declarative, reproducible, secure, observable, and recoverable**.
