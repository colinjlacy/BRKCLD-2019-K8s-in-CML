# Phase 2: Linux Host Preparation and K3s Cluster Bootstrap

This phase brings up two K3s clusters (Cluster A and Cluster B) on the existing topology with correct addressing and no default CNI (Cilium will be installed in a later phase). No Cilium, Hubble, BGP on Kubernetes, or application workloads in this phase.

**Assumptions:** Ubuntu 22.04 LTS on all Linux nodes; static IPs already assigned per topology. Phase 2 addressing: Cluster A nodes 10.10.0.11/24, .21/24, .22/24 (gateway **10.10.0.1** on site-a-rtr, shared L2 via **site-a-sw**); Cluster B 10.20.0.11/24, .21/24, .22/24 (gateway **10.20.0.1** on site-b-rtr, shared L2 via **site-b-sw**).

---

## Automated setup (recommended): cloud-init on jump-1 + Ansible

K3s install, worker join, and basic validation are driven by **Ansible** on **jump-1** (user `cisco`). **jump-1** cloud-init applies **Netplan** then **`apt-get`** installs dependencies (after DNS works), **waits up to 20 minutes** until **all six** K8s nodes accept **TCP 22** (if not, bootstrap **fails** and Ansible is not run), **clones** [github.com/colinjlacy/BRKCLD-2019-K8s-in-CML](https://github.com/colinjlacy/BRKCLD-2019-K8s-in-CML.git) **branch `ansible`** to `/home/cisco/BRKCLD-2019-K8s-in-CML`, and runs **`ansible/site.yml`** with a **2-hour** timeout. The playbook’s first play waits again (**1200s** per host, **`any_errors_fatal`**) before SSH to nodes. Logs: `/var/log/jump-bootstrap.log`.

Other Ubuntu nodes use the same **Netplan + `mirrors.kernel.org` apt** pattern and install **python3** after **`netplan apply`**. See [`lab/topology.yaml`](../lab/topology.yaml) and [`cloud-init/`](../cloud-init/).

- **Playbook:** [`ansible/site.yml`](../ansible/site.yml) with [`ansible/inventory.ini`](../ansible/inventory.ini) and [`ansible/group_vars/all.yml`](../ansible/group_vars/all.yml).
- **Manual re-run:** `cd ~/BRKCLD-2019-K8s-in-CML/ansible && ansible-playbook site.yml`. See [`ansible/README.md`](../ansible/README.md).

After a successful playbook run, on jump-1:

```bash
KUBECONFIG=~/.kube/k3s-a.yaml kubectl get nodes
KUBECONFIG=~/.kube/k3s-b.yaml kubectl get nodes
```

Nodes will show **NotReady** until Cilium is installed; that is expected.

**Cloud-init only:** [`cloud-init/`](../cloud-init/) mirrors the topology; see [`cloud-init/README.md`](../cloud-init/README.md).

---

## Manual steps (fallback reference)

The following sections document the same host prep, K3s install, and validation for use without Ansible or for troubleshooting.

**Interface name:** CML Ubuntu node definition uses `ens2`. If your node uses `eth0`, substitute it in the commands below.

---

## 1. Linux host preparation

Run these steps on **each** Linux node (jump-1, k8s-a-cp, k8s-a-w1, k8s-a-w2, k8s-b-cp, k8s-b-w1, k8s-b-w2). Use the hostname and IP for that node from the tables.

### 1.1 Set hostname

Ensures the node has a stable hostname for K3s and /etc/hosts.

**jump-1:**
```bash
sudo hostnamectl set-hostname jump-1
```

**k8s-a-cp:**
```bash
sudo hostnamectl set-hostname k8s-a-cp
```

**k8s-a-w1:**
```bash
sudo hostnamectl set-hostname k8s-a-w1
```

**k8s-a-w2:**
```bash
sudo hostnamectl set-hostname k8s-a-w2
```

**k8s-b-cp:**
```bash
sudo hostnamectl set-hostname k8s-b-cp
```

**k8s-b-w1:**
```bash
sudo hostnamectl set-hostname k8s-b-w1
```

**k8s-b-w2:**
```bash
sudo hostnamectl set-hostname k8s-b-w2
```

### 1.2 /etc/hosts entries

Add the same block to **every** Linux node so all hosts resolve by name. Edit once and copy to each node, or run the same append on each.

```bash
sudo tee -a /etc/hosts << 'EOF'

# K8s lab - all nodes
10.30.0.10   jump-1
10.10.0.11   k8s-a-cp
10.10.0.21   k8s-a-w1
10.10.0.22   k8s-a-w2
10.20.0.11   k8s-b-cp
10.20.0.21   k8s-b-w1
10.20.0.22   k8s-b-w2
EOF
```

(If you already have duplicate lines, remove them first or use a single shared file and deploy it to all nodes.)

### 1.3 Required packages

Needed for K3s install script and general use (curl, optional apt utils).

```bash
sudo apt-get update
sudo apt-get install -y curl
```

### 1.4 Disable swap

Kubernetes expects swap off. Disable it now and persist across reboots.

```bash
sudo swapoff -a
sudo sed -i '/\bswap\b/d' /etc/fstab
```

### 1.5 Sysctl settings for Kubernetes

Load overlay and br_netfilter, enable bridge-nf-call-iptables and bridge-nf-call-ip6tables so that traffic between pods and nodes is correctly handled once a CNI is in place.

```bash
sudo tee /etc/modules-load.d/k8s.conf << 'EOF'
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/99-kubernetes.conf << 'EOF'
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl -p /etc/sysctl.d/99-kubernetes.conf
```

---

## 2. Network validation

Run on **each** node (or from jump-1 to each) to confirm connectivity before installing K3s.

### 2.1 Default gateway

```bash
ip route show default
```

- **jump-1:** default via 10.30.0.1  
- **Cluster A (k8s-a-cp, k8s-a-w1, k8s-a-w2):** default via 10.10.0.1  
- **Cluster B (k8s-b-cp, k8s-b-w1, k8s-b-w2):** default via 10.20.0.1  

### 2.2 DNS resolution

```bash
ping -c 1 k8s-a-cp
ping -c 1 k8s-b-cp
ping -c 1 jump-1
```

(Assumes /etc/hosts is populated as above.)

### 2.3 Outbound Internet access

From each node:

```bash
curl -s -o /dev/null -w "%{http_code}" https://get.k3s.io
```

Expect `200`. If it fails, check default route and that the WAN router has a default to the external segment (Phase 1).

---

## 3. K3s control plane installation

Install the **server** (control plane) on **k8s-a-cp** and **k8s-b-cp** only. Flags disable default networking and optional components so Cilium can be the sole CNI later; nodes will stay NotReady until Cilium is installed.

### 3.1 Cluster A – control plane (k8s-a-cp, 10.10.0.11)

Run on **k8s-a-cp**:

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --node-ip=10.10.0.11 \
  --flannel-backend=none \
  --disable=traefik,servicelb \
  --disable-network-policy \
  --disable-kube-proxy
```

- `--node-ip=10.10.0.11`: Use this node’s IP so the API and certificates match.
- `--flannel-backend=none`: No built-in CNI; Cilium will be installed later.
- `--disable=traefik,servicelb`: No default ingress or LoadBalancer controller.
- `--disable-network-policy`: No built-in network policy; Cilium will provide it.
- `--disable-kube-proxy`: So Cilium can replace kube-proxy in a later phase.

Wait until the server is up (e.g. `sudo systemctl status k3s`).

### 3.2 Cluster B – control plane (k8s-b-cp, 10.20.0.11)

Run on **k8s-b-cp**:

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --node-ip=10.20.0.11 \
  --flannel-backend=none \
  --disable=traefik,servicelb \
  --disable-network-policy \
  --disable-kube-proxy
```

Again wait until the server is running.

---

## 4. Worker node join procedure

### 4.1 Get the node token from the control plane

On **k8s-a-cp** (Cluster A token):

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Copy the token (single line). Use it as `NODE_TOKEN_A` below.

On **k8s-b-cp** (Cluster B token):

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Copy the token. Use it as `NODE_TOKEN_B` below.

### 4.2 Cluster A – join workers

Run on **k8s-a-w1** (replace `NODE_TOKEN_A` with the token from k8s-a-cp):

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://10.10.0.11:6443 K3S_TOKEN=NODE_TOKEN_A sh -s - agent \
  --node-ip=10.10.0.21
```

Run on **k8s-a-w2**:

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://10.10.0.11:6443 K3S_TOKEN=NODE_TOKEN_A sh -s - agent \
  --node-ip=10.10.0.22
```

`K3S_URL` must be the control plane IP (10.10.0.11). `--node-ip` sets the worker’s IP so the control plane registers it correctly.

### 4.3 Cluster B – join workers

Run on **k8s-b-w1** (replace `NODE_TOKEN_B` with the token from k8s-b-cp):

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://10.20.0.11:6443 K3S_TOKEN=NODE_TOKEN_B sh -s - agent \
  --node-ip=10.20.0.21
```

Run on **k8s-b-w2**:

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://10.20.0.11:6443 K3S_TOKEN=NODE_TOKEN_B sh -s - agent \
  --node-ip=10.20.0.22
```

---

## 5. Kubeconfig handling

### 5.1 Copy kubeconfig from control planes to jump host

On **jump-1**, create a directory and fetch both kubeconfigs (from jump-1, assuming SSH or console access to the CP nodes):

```bash
mkdir -p ~/.kube
# From jump-1, if you can SSH to the control planes:
scp k8s-a-cp:/etc/rancher/k3s/k3s.yaml ~/.kube/k3s-a.yaml
scp k8s-b-cp:/etc/rancher/k3s/k3s.yaml ~/.kube/k3s-b.yaml
```

If you don’t have SSH from jump-1 to the CPs, copy the files manually (e.g. from CML console or copy/paste):

- From **k8s-a-cp:** `sudo cat /etc/rancher/k3s/k3s.yaml` → save on jump-1 as `~/.kube/k3s-a.yaml`
- From **k8s-b-cp:** `sudo cat /etc/rancher/k3s/k3s.yaml` → save on jump-1 as `~/.kube/k3s-b.yaml`

### 5.2 Fix server URL in kubeconfigs

On **jump-1**, point each kubeconfig at the control plane IP (not 127.0.0.1):

```bash
sed -i 's/127.0.0.1/10.10.0.11/' ~/.kube/k3s-a.yaml
sed -i 's/127.0.0.1/10.20.0.11/' ~/.kube/k3s-b.yaml
```

### 5.3 Use Cluster A or Cluster B

**Cluster A:**
```bash
export KUBECONFIG=~/.kube/k3s-a.yaml
kubectl get nodes
```

**Cluster B:**
```bash
export KUBECONFIG=~/.kube/k3s-b.yaml
kubectl get nodes
```

### 5.4 Use both clusters (optional)

```bash
export KUBECONFIG=~/.kube/k3s-a.yaml:~/.kube/k3s-b.yaml
kubectl config get-contexts
kubectl config use-context default
# Or switch by context name shown in get-contexts
```

---

## 6. Validation steps

From **jump-1** with the correct `KUBECONFIG` set.

### 6.1 Cluster A

```bash
export KUBECONFIG=~/.kube/k3s-a.yaml
kubectl get nodes
```

You should see k8s-a-cp, k8s-a-w1, k8s-a-w2. Until Cilium is installed, nodes will show **NotReady** (and possibly a “network plugin not ready” or “cni config uninitialized” message). That is expected.

```bash
kubectl get pods -A
```

System pods (e.g. coredns if not disabled) may be Pending or not fully running without a CNI; control plane components should be running on the CP node.

### 6.2 Cluster B

```bash
export KUBECONFIG=~/.kube/k3s-b.yaml
kubectl get nodes
kubectl get pods -A
```

Same expectations: three nodes (k8s-b-cp, k8s-b-w1, k8s-b-w2), nodes NotReady until Cilium is in place.

### 6.3 Control plane stability

On each control plane node:

```bash
sudo systemctl status k3s
```

Should be `active (running)`. Check logs if needed:

```bash
sudo journalctl -u k3s -f --no-pager
```

---

## 7. Troubleshooting

### 7.1 Worker not joining

- **Check token:** Token must be from the correct cluster and unchanged (no extra newlines when pasting).
- **Check connectivity:** From worker, `curl -k https://10.10.0.11:6443/version` (Cluster A) or `https://10.20.0.11:6443/version` (Cluster B). Must get JSON.
- **Check node-ip:** Worker’s `--node-ip` must match the IP of the interface the control plane can reach (e.g. 10.10.0.21 for k8s-a-w1).
- **Check K3S_URL:** Must match control plane IP and port 6443.

### 7.2 API server unreachable

- **From jump-1:** `curl -k https://10.10.0.11:6443/version` and same for 10.20.0.11. If this fails, check routing (default gateway on jump-1 and site routers) and that k3s server is running on the CP.
- **Firewall:** Ubuntu 22.04 ufw may be inactive by default; if you enabled it, allow 6443/tcp to the control plane IPs.

### 7.3 Nodes NotReady / “network plugin not ready” / “cni config uninitialized”

Expected in Phase 2. We installed K3s with `--flannel-backend=none` and no CNI. Nodes will become Ready after Cilium is installed in a later phase. No action needed unless you are already installing Cilium.

### 7.4 Kubeconfig “connection refused” or “no route to host”

- Ensure `KUBECONFIG` points at the correct file and that the file’s `server:` URL is the control plane IP (10.10.0.11 or 10.20.0.11), not 127.0.0.1, when using kubectl from jump-1.
- Re-run the `sed` commands in section 5.2 if you copied kubeconfig before changing the URLs.

### 7.5 Token or join command wrong cluster

- Cluster A: URL `https://10.10.0.11:6443`, token from k8s-a-cp.
- Cluster B: URL `https://10.20.0.11:6443`, token from k8s-b-cp.
- Do not mix tokens or URLs between clusters.

---

## Quick reference: node and IP mapping

| Node      | IP         | Gateway   | Role              |
|-----------|------------|-----------|-------------------|
| jump-1    | 10.30.0.10 | 10.30.0.1 | Client / admin    |
| k8s-a-cp  | 10.10.0.11 | 10.10.0.1 | Cluster A control plane |
| k8s-a-w1  | 10.10.0.21 | 10.10.0.1 | Cluster A worker  |
| k8s-a-w2  | 10.10.0.22 | 10.10.0.1 | Cluster A worker  |
| k8s-b-cp  | 10.20.0.11 | 10.20.0.1 | Cluster B control plane |
| k8s-b-w1  | 10.20.0.21 | 10.20.0.1 | Cluster B worker  |
| k8s-b-w2  | 10.20.0.22 | 10.20.0.1 | Cluster B worker  |

---

## Phase 2 complete

When the steps above are done you have:

- Hostnames and /etc/hosts set on all Linux nodes.
- Swap off and Kubernetes-related sysctls applied.
- Two K3s clusters (A and B) with one control plane and two workers each, CNI-less and with kube-proxy disabled, ready for Cilium in the next phase.
- Kubeconfig on jump-1 for both clusters and validation steps to confirm nodes and control plane stability.

Do **not** install Cilium, Hubble, or BGP on the nodes, or deploy applications, until the next phase.

Cilium, Hubble, and networking checks are automated in [`ansible/site.yml`](../ansible/site.yml) after K3s bootstrap.
