# Talos Cluster MAC cluster(Stage) Rebuild Runbook

## Cluster Information

| Item               | Value                                                          |
| ------------------ | -------------------------------------------------------------- |
| **Cluster Name**   | `home-k8s-p01`                                                 |
| **Endpoint**       | `home-k8s-p01.tmatthews.casa`                                  |
| **Control Planes** | `IP address for cp02`, `IP address for cp03`                   |
| **Workers**        | None                                                           |
| **Talos Config**   | `/root/Labs/kubernetes/Talos/home-k8s-p01/secrets/talosconfig` |

---

## Phase 1 – Prepare the Nodes

### 1.1 Wipe Existing Installation

Clear the node hard drives using **GParted** and remove all existing partitions.

Then:

1. Reboot the node.
2. Boot from the **Talos installation USB**.
3. On Mac nodes, **hold `ALT` during boot** to select the USB device.

### 1.2 Validate Network Connectivity

Verify that the nodes are reachable before continuing:

```bash
ping <node-ip>
```

### 1.3 Load Environment Variables

```bash
source ~/Labs/kubernetes/Talos/config.sh
```

---

## Phase 2 – Verify Installation Target

### 2.1 Check Available Disks

Identify the correct disk before installing Talos:

```bash
talosctl get disks --insecure --nodes $CONTROL_PLANE_IP
```

Example disk:

```text
nvme0n1
500 GB
APPLE SSD AP0512M
```

> **⚠️ Warning:** Verify the disk carefully before proceeding. Installing Talos on the wrong disk can destroy existing data.

---

## Phase 3 – Generate Talos Configuration

### 3.1 Generate Secrets

```bash
talosctl gen secrets -o secrets.yaml
```

### 3.2 Generate Machine Configuration

```bash
talosctl gen config \
  --with-secrets /root/Labs/kubernetes/Talos/home-k8s-p01/secrets/secrets.yaml \
  --talos-version v1.12.9 \
  $CLUSTER_NAME \
  https://$YOUR_ENDPOINT:6443
```

### 3.3 Review Configuration

Review and update `configs.yaml` with all environment-specific values.

Verify items such as:

* Node IP addresses
* Hostnames
* Network configuration
* VLAN configuration
* DNS settings
* Kubernetes endpoint
* Talos version
* Cluster-specific settings

---

## Phase 4 – Deploy Talos

### 4.1 Apply Configuration

```bash
/root/Labs/kubernetes/Talos/talos.sh apply home-k8s-p01 --insecure
```

### 4.2 Bootstrap Cluster

```bash
/root/Labs/kubernetes/Talos/talos.sh bootstrap home-k8s-p01
```

### 4.3 Configure Local Access

```bash
/root/Labs/kubernetes/Talos/talos.sh setup-ctl home-k8s-p01
```

### 4.4 Reset Cluster — Optional

Use only when a cluster reset is required:

```bash
/root/Labs/kubernetes/Talos/talos.sh reset home-k8s-p01
```

---

## Phase 5 – Verify Cluster Health

Confirm that all control-plane nodes have successfully joined the cluster:

```bash
kubectl get nodes
```

### Expected Result

All nodes should report:

```text
STATUS = Ready
```

---

## Phase 6 – Install Cilium

### 6.1 Deploy Cilium

```bash
./cilium-setup.sh
```

### Expected Outcome

```text
Cilium setup complete!
```

The script installs/configures:

* **Cilium**
* **Hubble Relay**
* **Hubble UI**
* **BGP resources**
* **LoadBalancer resources**

---

## Phase 7 – Apply Cluster Fixes

### 7.1 CoreDNS Patch

Apply the CoreDNS patch:

```bash
kubectl apply -f coredns-patch.yaml
```

Verify CoreDNS is healthy:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

---

## Phase 8 – Install Flux

### 8.1 Bootstrap Flux

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=homelab-infra \
  --branch=main \
  --path=./clusters/home-k8s-p01 \
  --personal
```

### 8.2 Apply GitHub SOPS Secret

After Flux has been successfully installed, apply the **GitHub SOPS secret** required for encrypted secrets.

> **Important:** Ensure the SOPS age key is available to Flux before expecting encrypted resources to reconcile successfully.

---

## Phase 9 – Verify Flux Synchronization

### 9.1 Watch Kustomizations

```bash
watch flux get kustomizations -A
```

### Expected Status

All Flux Kustomizations should eventually report:

```text
READY = True
```

Allow a few reconciliation cycles for all infrastructure and applications to become healthy.

---

# Final Validation Checklist

* [ ] Nodes wiped
* [ ] Talos installed
* [ ] Network connectivity verified
* [ ] Installation disk verified
* [ ] Talos secrets generated
* [ ] Machine configurations generated
* [ ] Machine configurations reviewed
* [ ] Talos configuration applied
* [ ] Cluster bootstrapped
* [ ] Local `talosctl` access configured
* [ ] All nodes report `Ready`
* [ ] Cilium installed
* [ ] Hubble Relay installed
* [ ] Hubble UI installed
* [ ] BGP resources configured
* [ ] LoadBalancer resources configured
* [ ] CoreDNS patch applied
* [ ] Flux bootstrapped
* [ ] GitHub SOPS secret applied
* [ ] Flux Kustomizations report `Ready = True`

---

## Recovery Complete

The cluster rebuild is considered **complete** when:

1. All Talos nodes are healthy.
2. Kubernetes reports all nodes as `Ready`.
3. Cilium is operational.
4. CoreDNS is functioning.
5. Flux is successfully synchronized.
6. All Flux Kustomizations report `Ready = True`.
7. Applications have been successfully reconciled from Git.
