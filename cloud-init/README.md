# Phase 2: Cloud-Init Automation for K3s Bootstrap

This directory contains cloud-init **user-data** for all seven Linux nodes. When applied at first boot, they configure hostnames, `/etc/hosts`, packages, swap, sysctl, install K3s (control plane or agent), and on jump-1 fetch kubeconfig and run a basic health check—no manual steps.

## Addressing

| Node      | IP         | Gateway   | Role              |
|-----------|------------|-----------|-------------------|
| jump-1    | 10.30.0.10 | 10.30.0.1 | Admin; kubeconfig + validation |
| k8s-a-cp  | 10.10.0.11 | 10.10.0.1 | Cluster A control plane |
| k8s-a-w1  | 10.10.0.21 | 10.10.0.1 | Cluster A worker  |
| k8s-a-w2  | 10.10.0.22 | 10.10.0.1 | Cluster A worker  |
| k8s-b-cp  | 10.20.0.11 | 10.20.0.1 | Cluster B control plane |
| k8s-b-w1  | 10.20.0.21 | 10.20.0.1 | Cluster B worker  |
| k8s-b-w2  | 10.20.0.22 | 10.20.0.1 | Cluster B worker  |

Static IPs and default gateways must be configured **before** or **outside** cloud-init (e.g. netplan or CML interface config). These files do not set IPs.

## Files

| File | Node | Purpose |
|------|------|---------|
| `user-data-jump-1.yaml` | jump-1 | Host prep, curl + kubectl; wait for both APIs → fetch kubeconfig → `kubectl get nodes` for both clusters |
| `user-data-k8s-a-cp.yaml` | k8s-a-cp | Host prep, K3s server (CNI-less), serve token + k3s.yaml on port 8999 |
| `user-data-k8s-a-w1.yaml` | k8s-a-w1 | Host prep, wait for token from 10.10.0.11:8999, join as agent (node-ip 10.10.0.21) |
| `user-data-k8s-a-w2.yaml` | k8s-a-w2 | Host prep, wait for token, join as agent (node-ip 10.10.0.22) |
| `user-data-k8s-b-cp.yaml` | k8s-b-cp | Host prep, K3s server (CNI-less), serve token + k3s.yaml on port 8999 |
| `user-data-k8s-b-w1.yaml` | k8s-b-w1 | Host prep, wait for token from 10.20.0.11:8999, join as agent (node-ip 10.20.0.21) |
| `user-data-k8s-b-w2.yaml` | k8s-b-w2 | Host prep, wait for token, join as agent (node-ip 10.20.0.22) |

## How it works

1. **Control planes (k8s-a-cp, k8s-b-cp)**  
   Install K3s server with `--flannel-backend=none`, `--disable=traefik,servicelb`, `--disable-network-policy`, `--disable-kube-proxy`. After the server is up, a script copies the node token and k3s.yaml (with server URL fixed) into a directory and runs `python3 -m http.server 8999` in the background so workers and jump-1 can fetch them over HTTP.

2. **Workers**  
   A script (1) waits for the control plane to serve the token (`curl http://<cp-ip>:8999/node-token`) every 5 seconds for up to 5 minutes, then (2) waits for the API to respond (`curl -sk https://<cp>:6443/version`, up to 24×15s = 6 min) so the install runs only once the control plane is reachable. The K3s install script is downloaded and run once, not repeatedly. No manual token copy.

3. **jump-1**  
   Waits until both cluster APIs respond (`https://10.10.0.11:6443` and `https://10.20.0.11:6443`), then fetches k3s.yaml from each CP via port 8999, saves to `/root/.kube/k3s-a.yaml` and `k3s-b.yaml`, fixes server URLs, and runs `kubectl get nodes` for both clusters as a basic health check. Kubeconfig is ready for `export KUBECONFIG=/root/.kube/k3s-a.yaml` (or k3s-b.yaml).

## Applying in CML

1. **Per-node configuration**  
   In CML, open each Linux node’s configuration and set **cloud-init** / **user-data** to the contents of the matching file (e.g. for node **k8s-a-cp** use `user-data-k8s-a-cp.yaml`). CML often exposes this as a “Configuration” or “Cloud-Init” field where you paste the YAML.

2. **Embedded in topology (default)**  
   In `lab/topology.yaml`, each node’s `configuration` list can include an entry with `name: user-data` and `content: |` followed by the cloud-config YAML (indented). This cloud-init is already inlined in the topology; import it and start the nodes—no per-node paste needed.

3. **Boot order**  
   Start control plane nodes first, then workers, then jump-1 so that token and API are available when workers and jump-1 run their runcmd. If all nodes boot together, workers and jump-1 will retry until the control planes are ready (up to 5 minutes for token, ~3 minutes for APIs).

## Firewall / connectivity

- Control planes must allow **TCP 8999** from workers and jump-1 (and from the same subnet in your lab). Ubuntu 22.04 with default ufw is typically inactive; if you enable a firewall, open 8999.
- Workers and jump-1 must reach control plane IPs on **6443** (API) and **8999** (token/kubeconfig). Routing is assumed from Phase 1 (e.g. jump-1 → wan-rtr → site routers → cluster nodes).

## Validation after boot

- On **jump-1**:  
  `KUBECONFIG=/root/.kube/k3s-a.yaml kubectl get nodes`  
  `KUBECONFIG=/root/.kube/k3s-b.yaml kubectl get nodes`  
  Nodes will show **NotReady** until Cilium is installed in a later phase; that is expected.

## Security note

Port 8999 serves the K3s node token and kubeconfig with no authentication. Use only in an isolated lab; do not expose to untrusted networks.
