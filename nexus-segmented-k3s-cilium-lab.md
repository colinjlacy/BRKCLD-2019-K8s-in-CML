# Kubernetes on CML: Nexus-Segmented k3s + Cilium Lab

This lab uses a single Cisco Nexus 9000v in Cisco Modeling Labs as the segmentation point for a k3s Kubernetes cluster. The intent is to show network engineers how familiar data center primitives, including VLANs, SVIs, routed segmentation, and port role ACLs, can protect Kubernetes infrastructure while still allowing Cilium to expose LoadBalancer services on an externally reachable network.

The lab keeps the worker nodes and Cilium LoadBalancer IPs on `198.18.1.0/24` so services can be reached from outside the lab through the CML External Connector. The control plane and database live on protected VLANs and are not directly reachable from the external connector.

This write-up is designed to fit the existing repository structure in `colinjlacy/BRKCLD-2019-K8s-in-CML`:

```text
BRKCLD-2019-K8s-in-CML/
  lab/                         # CML topology YAMLs
  cloud-init/                  # reusable Ubuntu user-data snippets
  configs/                     # network device startup configs
  kubernetes/                  # reusable Kubernetes/Cilium manifests
  ansible/                     # Ansible scenario folders and playbooks
```

The current examples use names such as `jump-1`, `k8s-cp`, `k8s-w1`, `k8s-w2`, the default CML Ubuntu user `cisco`, and scenario-specific Ansible folders such as `ansible/flat-k3s-cilium`. This lab follows that convention and should be added as a new scenario rather than replacing the existing flat or CSR multicluster labs.

## Target Design

```text
                         outside CML / host network
                                  |
                         External Connector
                                  |
                             Eth1/1
                           Nexus 9000v
             +--------------------+--------------------+
             |                    |                    |
        VLAN 198              VLAN 30              VLAN 40
   public/workload net    control-plane net       database net
     198.18.1.0/24        10.10.30.0/24        10.10.40.0/24
             |                    |                    |
     +-------+-------+       k8s-cp              postgres-1
     |               |
   k8s-w1          k8s-w2

             |
          VLAN 10
       management net
       10.10.10.0/24
             |
        jump-1
```

The management Ubuntu node is `jump-1`. It is the Ansible control host. It must be able to SSH to every Linux node and reach the Kubernetes API on the control-plane node. It is the only node that should have broad administrative access.

## Recommended CML Nodes

Use the following CML node types:

| Role | CML Node Type | Notes |
|---|---|---|
| Fabric segmentation | `NX-OS 9000` | Use Nexus 9000v/9300v. This lab uses SVIs, VLANs, and port ACLs. |
| Management / Ansible | `Ubuntu` | `jump-1`, the Ansible control host. Give it outbound access for downloads. |
| Kubernetes control plane | `Ubuntu` | Runs k3s server and Cilium agent. Protected VLAN. |
| Kubernetes workers | `Ubuntu` | Run k3s agents. Same VLAN as the external connector. |
| Database | `Ubuntu` | Runs PostgreSQL on a small data volume. Protected VLAN. |
| Outside access | `External Connector` | Use the connector that provides your existing `198.18.1.0/24` access network. |

Cisco's CML documentation notes that NX-OS 9000v uses a software data plane and does not emulate hardware ASIC behavior. Validate the specific ACL behavior in your selected CML/NX-OS image before using the lab on stage.

## Addressing Plan

| Segment | VLAN | Subnet | Gateway | Purpose |
|---|---:|---|---|---|
| Management | 10 | `10.10.10.0/24` | `10.10.10.1` | Ansible, SSH, kubectl |
| Control plane | 30 | `10.10.30.0/24` | `10.10.30.1` | Protected Kubernetes API/server |
| Database | 40 | `10.10.40.0/24` | `10.10.40.1` | Protected PostgreSQL node |
| Public/workload | 198 | `198.18.1.0/24` | `198.18.1.254` | Workers, Cilium LoadBalancer IPs, external connector |

Suggested node addresses:

| Node | Interface | Address | Default Gateway |
|---|---|---|---|
| `n9k-fabric` | `Vlan10` | `10.10.10.1/24` | N/A |
| `n9k-fabric` | `Vlan30` | `10.10.30.1/24` | N/A |
| `n9k-fabric` | `Vlan40` | `10.10.40.1/24` | N/A |
| `n9k-fabric` | `Vlan198` | `198.18.1.254/24` | Optional upstream route via your CML connector |
| `jump-1` | `ens2` | `10.10.10.10/24` | `10.10.10.1` |
| `k8s-cp` | `ens2` | `10.10.30.10/24` | `10.10.30.1` |
| `postgres-1` | `ens2` | `10.10.40.10/24` | `10.10.40.1` |
| `k8s-w1` | `ens2` | `198.18.1.21/24` | `198.18.1.254` |
| `k8s-w2` | `ens2` | `198.18.1.22/24` | `198.18.1.254` |

Reserve the following public/workload ranges:

| Range | Use |
|---|---|
| `198.18.1.1-198.18.1.20` | External network infrastructure, CML host bridge, test clients |
| `198.18.1.21-198.18.1.49` | Kubernetes worker node IPs |
| `198.18.1.50-198.18.1.99` | Future lab nodes |
| `198.18.1.100-198.18.1.149` | Cilium LoadBalancer IP pool |
| `198.18.1.254` | Nexus SVI gateway |

If your CML external network already uses `198.18.1.254`, move the Nexus SVI to another unused address and update node gateways accordingly.

## CML Link Map

Use deterministic interface assignments so the topology and configuration stay easy to explain.

| Nexus Interface | Connected Node | VLAN / Role |
|---|---|---|
| `Ethernet1/1` | External Connector | VLAN 198, external/public |
| `Ethernet1/2` | `jump-1` | VLAN 10, management |
| `Ethernet1/3` | `k8s-cp` | VLAN 30, control plane |
| `Ethernet1/4` | `k8s-w1` | VLAN 198, worker |
| `Ethernet1/5` | `k8s-w2` | VLAN 198, worker |
| `Ethernet1/6` | `postgres-1` | VLAN 40, database |
| `Ethernet1/7-1/10` | Future workers | VLAN 198, worker role |

Do not attach the control-plane or database nodes directly to the external connector.

## External Connector Choice

Use the CML External Connector mode that matches how you want to reach the LoadBalancer IPs:

| Connector Mode | Use When |
|---|---|
| System Bridge / custom L2 bridge | You want outside clients on the same L2 network to ARP for `198.18.1.x` LoadBalancer IPs. |
| NAT connector | You only need outbound Internet access from lab nodes. NAT mode is less suitable for direct L2 LoadBalancer demos. |

For this lab's LoadBalancer story, an L2 bridge or custom bridge that exposes `198.18.1.0/24` is the cleanest option.

## Nexus 9000v Base Configuration

Paste the following into the NX-OS node as day-0 config or apply it manually after boot. Adjust interface names if CML maps them differently.

```text
hostname n9k-fabric

feature interface-vlan
feature ssh

ip routing

vlan 10
  name MGMT
vlan 30
  name K8S_CONTROL_PLANE
vlan 40
  name DATABASE
vlan 198
  name PUBLIC_WORKLOAD

interface Vlan10
  description MGMT default gateway
  no shutdown
  ip address 10.10.10.1/24

interface Vlan30
  description K8S control-plane default gateway
  no shutdown
  ip address 10.10.30.1/24

interface Vlan40
  description Database default gateway
  no shutdown
  ip address 10.10.40.1/24

interface Vlan198
  description Public worker and LoadBalancer gateway
  no shutdown
  ip address 198.18.1.254/24

! If the external network has an upstream gateway, set it here.
! Remove or change this if 198.18.1.1 is not your upstream gateway.
ip route 0.0.0.0/0 198.18.1.1

ip access-list ACL-EXT-IN
  10 remark External connector must not reach protected segments
  15 remark Optional hardening: block direct SSH/API attempts from outside
  16 deny tcp any 198.18.1.0/24 eq 22
  17 deny tcp any 198.18.1.0/24 eq 6443
  20 deny ip any 10.10.10.0/24
  30 deny ip any 10.10.30.0/24
  40 deny ip any 10.10.40.0/24
  90 permit ip any any

ip access-list ACL-WORKER-IN
  10 remark Worker ports may reach the API, Cilium overlay, and database service
  15 remark Permit return traffic for management-initiated SSH and control-plane-initiated kubelet checks
  16 permit tcp any 10.10.10.0/24 established
  17 permit tcp any host 10.10.30.10 established
  20 permit tcp any host 10.10.30.10 eq 6443
  30 permit udp any host 10.10.30.10 eq 8472
  40 permit tcp any host 10.10.40.10 eq 5432
  50 permit icmp any 10.10.30.0/24
  60 permit icmp any 10.10.40.0/24
  70 deny ip any 10.10.10.0/24
  80 deny ip any 10.10.30.0/24
  90 deny ip any 10.10.40.0/24
  100 permit ip any any

ip access-list ACL-MGMT-IN
  10 remark Management node may administer all lab nodes
  20 permit tcp any 198.18.1.0/24 eq 22
  30 permit tcp any host 10.10.30.10 eq 22
  40 permit tcp any host 10.10.30.10 eq 6443
  50 permit tcp any host 10.10.40.10 eq 22
  60 permit tcp any host 10.10.40.10 eq 5432
  70 permit icmp any any
  90 permit ip any any

interface Ethernet1/1
  description external_connector
  switchport
  switchport mode access
  switchport access vlan 198
  spanning-tree port type edge
  ip port access-group ACL-EXT-IN in
  no shutdown

interface Ethernet1/2
  description jump-1
  switchport
  switchport mode access
  switchport access vlan 10
  spanning-tree port type edge
  ip port access-group ACL-MGMT-IN in
  no shutdown

interface Ethernet1/3
  description k8s-cp
  switchport
  switchport mode access
  switchport access vlan 30
  spanning-tree port type edge
  no shutdown

interface Ethernet1/4
  description k8s-w1
  switchport
  switchport mode access
  switchport access vlan 198
  spanning-tree port type edge
  ip port access-group ACL-WORKER-IN in
  no shutdown

interface Ethernet1/5
  description k8s-w2
  switchport
  switchport mode access
  switchport access vlan 198
  spanning-tree port type edge
  ip port access-group ACL-WORKER-IN in
  no shutdown

interface Ethernet1/6
  description postgres-1
  switchport
  switchport mode access
  switchport access vlan 40
  spanning-tree port type edge
  no shutdown

interface Ethernet1/7
  description future-k3s-worker
  switchport
  switchport mode access
  switchport access vlan 198
  spanning-tree port type edge
  ip port access-group ACL-WORKER-IN in
  shutdown

interface Ethernet1/8
  description future-k3s-worker
  switchport
  switchport mode access
  switchport access vlan 198
  spanning-tree port type edge
  ip port access-group ACL-WORKER-IN in
  shutdown
```

The worker ACL is intentionally applied to worker switchports rather than matching each worker source IP. To add another worker, connect it to a preconfigured worker port or apply the same access-port and PACL role to a new interface. You do not need to change address-specific policy unless the node needs a new protected service.

## Ubuntu Node Network Configuration

Use cloud-init, netplan, or your existing Ansible patterns to set static addressing. CML Ubuntu interface names commonly appear as `ens2`, but verify with `ip link`.

Example worker netplan:

```yaml
network:
  version: 2
  ethernets:
    ens2:
      addresses:
        - 198.18.1.21/24
      routes:
        - to: default
          via: 198.18.1.254
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

Example control-plane netplan:

```yaml
network:
  version: 2
  ethernets:
    ens2:
      addresses:
        - 10.10.30.10/24
      routes:
        - to: default
          via: 10.10.30.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

Example worker netplan for `k8s-w2` is identical except for the address:

```yaml
addresses:
  - 198.18.1.22/24
```

Example database netplan:

```yaml
network:
  version: 2
  ethernets:
    ens2:
      addresses:
        - 10.10.40.10/24
      routes:
        - to: default
          via: 10.10.40.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

Example management netplan:

```yaml
network:
  version: 2
  ethernets:
    ens2:
      addresses:
        - 10.10.10.10/24
      routes:
        - to: 10.10.30.0/24
          via: 10.10.10.1
        - to: 10.10.40.0/24
          via: 10.10.10.1
        - to: 198.18.1.0/24
          via: 10.10.10.1
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

If `jump-1` also needs Internet access for downloading Ansible collections, k3s artifacts, Helm charts, or Cilium manifests, give it either:

1. A second interface connected to a CML NAT External Connector, with default route through that NAT interface.
2. A default route through VLAN 10, if your upstream network has return routes or NAT for the protected lab subnets.

The first option is usually more predictable in CML. Keep Ansible traffic on the management VLAN and use the second interface only for outbound downloads.

## Management Node Requirements

`jump-1` is the automation control point. It must be able to reach:

| Destination | Required Access | Purpose |
|---|---|---|
| `k8s-cp` | TCP `22`, TCP `6443` | SSH provisioning and Kubernetes API access |
| `k8s-w1/w2` | TCP `22` | SSH provisioning |
| `postgres-1` | TCP `22`, optionally TCP `5432` | SSH provisioning and DB validation |
| `n9k-fabric` | SSH or console | Fabric configuration and validation |

The existing repository examples use the CML Ubuntu default `cisco` account. Keep that convention for this lab unless you are deliberately changing the base image:

```yaml
system_info:
  default_user:
    name: cisco
password: cisco
ssh_pwauth: true
chpasswd:
  expire: false
```

Install baseline tools on `jump-1`:

```bash
sudo apt-get update
sudo apt-get install -y ansible sshpass git curl jq openssh-client python3 python3-pip
```

Add this scenario to the repo with the same pattern used by the current `flat-k3s-cilium` and `csr-multicluster-k3s-cilium` examples:

```text
BRKCLD-2019-K8s-in-CML/
  lab/
    topology-nexus-segmented-k3s-cilium.yaml
  configs/
    n9k-fabric.config
  cloud-init/
    user-data-nexus-jump-1.yaml
    user-data-nexus-k8s-cp.yaml
    user-data-nexus-k8s-w1.yaml
    user-data-nexus-k8s-w2.yaml
    user-data-nexus-postgres-1.yaml
  kubernetes/
    nexus-lb-ipam.yaml
    nexus-l2-announcements.yaml
  ansible/
    nexus-segmented-k3s-cilium/
      ansible.cfg
      inventory.ini
      site.yml
```

Example Ansible inventory:

```ini
[mgmt_nodes]
jump-1 ansible_host=10.10.10.10 ansible_connection=local

[k8s_control]
k8s-cp ansible_host=10.10.30.10 k3s_cp_api_ip=10.10.30.10 cilium_cluster_name=nexus-segmented cilium_cluster_id=1

[k8s_workers]
k8s-w1 ansible_host=198.18.1.21 k3s_cp_api_ip=10.10.30.10
k8s-w2 ansible_host=198.18.1.22 k3s_cp_api_ip=10.10.30.10

[databases]
postgres-1 ansible_host=10.10.40.10 postgres_data_device=/dev/vdb postgres_data_mount=/var/lib/postgresql

[k8s_nodes:children]
k8s_control
k8s_workers

[ubuntu_lab:children]
mgmt_nodes
k8s_nodes
databases

[ubuntu_lab:vars]
ansible_user=cisco
ansible_ssh_pass=cisco

[all:vars]
cilium_helm_chart_version=1.19.3
cilium_cli_version=v0.19.2
hubble_cli_version=v1.19.3
k3s_cluster_cidr=10.200.0.0/16
k3s_service_cidr=10.201.0.0/16
cilium_cluster_pool_ipv4_cidr=10.200.0.0/16
cilium_cluster_pool_ipv4_mask_size=24
cilium_lb_pool_start=198.18.1.100
cilium_lb_pool_stop=198.18.1.149
```

Example `ansible/nexus-segmented-k3s-cilium/ansible.cfg`:

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
remote_user = cisco
timeout = 30
interpreter_python = auto_silent
remote_tmp = /tmp/.ansible-tmp
```

## Protected Node Software Installation

The cleanest security model is that protected nodes do not need direct inbound access from the external connector. They are configured by the management node over SSH.

For package and container image downloads, choose one of these provisioning models:

| Model | Description | Best Use |
|---|---|---|
| Direct outbound | Protected nodes use the Nexus as default gateway and your upstream CML/external network can return traffic to protected subnets. | Simple if your CML environment already routes or NATs these subnets. |
| Management staging | Management node downloads artifacts, then Ansible copies binaries, packages, Helm charts, and manifests to protected nodes. | Most faithful to the segmentation story. |
| Temporary provisioning access | Temporarily give protected nodes an install-only NAT path, then remove it before the demo. | Useful when building the lab repeatedly. |
| Prebuilt image | Build a custom Ubuntu image with common packages and container images preloaded. | Best for reliable conference delivery. |

Do not weaken the steady-state topology just to make package installation easier. If the control plane and database are meant to be protected, keep them off the external connector for the final demo state.

## PostgreSQL Database Node

The database node should run PostgreSQL and use a small dedicated data volume. A 512 MB volume is enough for this demo.

In CML, add a small additional disk to the `postgres-1` Ubuntu node if your node definition supports it. The examples below assume the disk appears as `/dev/vdb`.

Manual setup:

```bash
sudo parted --script /dev/vdb mklabel gpt
sudo parted --script /dev/vdb mkpart primary ext4 0% 100%
sudo mkfs.ext4 -F /dev/vdb1
sudo mkdir -p /var/lib/postgresql
sudo mount /dev/vdb1 /var/lib/postgresql
sudo blkid /dev/vdb1
```

Add the volume to `/etc/fstab` using the UUID from `blkid`:

```text
UUID=<uuid-from-blkid> /var/lib/postgresql ext4 defaults,nofail 0 2
```

Install PostgreSQL:

```bash
sudo apt-get update
sudo apt-get install -y postgresql postgresql-contrib
sudo systemctl enable --now postgresql
```

For the lab, listen only on the database VLAN address:

```bash
sudo sed -i "s/^#listen_addresses =.*/listen_addresses = '10.10.40.10'/" /etc/postgresql/*/main/postgresql.conf
```

Add a narrow `pg_hba.conf` rule for the worker subnet:

```text
host    all     all     198.18.1.0/24     scram-sha-256
```

Restart PostgreSQL:

```bash
sudo systemctl restart postgresql
```

The Nexus worker-port ACL only permits worker-role ports to initiate TCP `5432` to `10.10.40.10`. The external connector port is denied access to the database VLAN.

## k3s Installation

Install k3s with Flannel disabled so Cilium owns the CNI role.

On the control-plane node:

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="server \
    --flannel-backend=none \
    --disable-network-policy \
    --disable-kube-proxy \
    --disable=traefik,servicelb \
    --cluster-cidr=10.200.0.0/16 \
    --service-cidr=10.201.0.0/16 \
    --node-ip=10.10.30.10 \
    --advertise-address=10.10.30.10 \
    --tls-san=10.10.30.10 \
    --write-kubeconfig-mode=0644" \
  K3S_TOKEN="replace-with-lab-token" \
  sh -
```

Fetch the node token if you let k3s generate it:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

On each worker:

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL="https://10.10.30.10:6443" \
  K3S_TOKEN="replace-with-lab-token" \
  INSTALL_K3S_EXEC="agent --node-ip=<worker-node-ip>" \
  sh -
```

For example:

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL="https://10.10.30.10:6443" \
  K3S_TOKEN="replace-with-lab-token" \
  INSTALL_K3S_EXEC="agent --node-ip=198.18.1.21" \
  sh -
```

From the management node, copy the kubeconfig:

```bash
mkdir -p ~/.kube
scp cisco@10.10.30.10:/etc/rancher/k3s/k3s.yaml ~/.kube/cml-k3s.yaml
sed -i 's/127.0.0.1/10.10.30.10/g' ~/.kube/cml-k3s.yaml
export KUBECONFIG=~/.kube/cml-k3s.yaml
```

Label the worker nodes so Cilium L2 announcements only run on worker nodes:

```bash
kubectl label node k8s-w1 node-role.kubernetes.io/worker=true
kubectl label node k8s-w2 node-role.kubernetes.io/worker=true
```

## Cilium Mode Selection

Use Cilium tunnel mode with VXLAN for this lab.

That choice is deliberate:

1. The Kubernetes nodes span more than one VLAN.
2. You want the Nexus to demonstrate routed segmentation, not PodCIDR routing.
3. LoadBalancer exposure remains L2 on the public/workload VLAN.
4. No BGP is required in this version of the topology.

With VXLAN tunneling, the Nexus must permit node-to-node IP connectivity and UDP `8472` between Kubernetes nodes. Because worker and control-plane nodes are in different VLANs, the worker ingress PACL explicitly permits UDP `8472` from worker-role ports to the control-plane node.

Do not use Cilium native routing in this topology unless you also make the underlay aware of PodCIDRs through static routes or BGP. Native routing means the network must be able to route pod addresses. VXLAN tunnel mode means the underlay only needs to route node IPs and allow the tunnel protocol.

## Cilium Installation

Create the Cilium Helm values inside `ansible/nexus-segmented-k3s-cilium/site.yml` the same way the existing scenario playbooks generate temporary files on `jump-1`, or place the rendered file under `/tmp/cilium-values.yaml` during the play:

```yaml
kubeProxyReplacement: true
k8sServiceHost: 10.10.30.10
k8sServicePort: 6443

routingMode: tunnel
tunnelProtocol: vxlan

l2announcements:
  enabled: true

k8sClientRateLimit:
  qps: 20
  burst: 40

hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true

operator:
  replicas: 1
```

Install Cilium from the management node:

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
helm upgrade --install cilium cilium/cilium \
  --namespace kube-system \
  --create-namespace \
  --version 1.19.4 \
  --values /tmp/cilium-values.yaml
```

If you prefer the Cilium CLI, keep the same values in an Ansible template and pass them through Helm. The important settings are:

```text
kubeProxyReplacement=true
k8sServiceHost=10.10.30.10
k8sServicePort=6443
routingMode=tunnel
tunnelProtocol=vxlan
l2announcements.enabled=true
```

## Cilium LoadBalancer IPAM

Create `kubernetes/nexus-lb-ipam.yaml` with a LoadBalancer pool inside the public/workload VLAN:

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumLoadBalancerIPPool
metadata:
  name: nexus-public-workload-pool
spec:
  blocks:
    - start: 198.18.1.100
      stop: 198.18.1.149
```

Create `kubernetes/nexus-l2-announcements.yaml` with an L2 announcement policy that only selects worker nodes:

```yaml
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: nexus-public-worker-l2
spec:
  nodeSelector:
    matchExpressions:
      - key: node-role.kubernetes.io/control-plane
        operator: DoesNotExist
  interfaces:
    - ^eth[0-9]+
    - ^ens[0-9]+
  loadBalancerIPs: true
```

The policy prevents the protected control-plane node from announcing LoadBalancer IPs on the control-plane VLAN. The LoadBalancer VIP should live only on the worker-facing/public segment.

The repo already has a simple `kubernetes/l2-announcements.yaml` example that announces broadly. For this Nexus-segmented lab, use the worker-restricted policy above so the protected control-plane node does not become the elected L2 announcer for a public VIP.

Apply the resources:

```bash
kubectl apply -f kubernetes/nexus-lb-ipam.yaml
kubectl apply -f kubernetes/nexus-l2-announcements.yaml
```

Test with a simple workload:

```bash
kubectl create deployment web --image=nginx --replicas=2
kubectl expose deployment web --type=LoadBalancer --port=80 --target-port=80
kubectl get svc web
```

From an outside host on `198.18.1.0/24`, curl the assigned LoadBalancer IP:

```bash
curl http://198.18.1.100/
```

## Routing Mode and Encapsulation Boundary Warning

The advice you were given is the right mental model:

> Make sure your routing modes match your encapsulation boundaries, or your virtual BGP peerings will collapse long before you ever take the stage at Cisco Live.

For this lab, the matching is:

| Layer | Choice | Reason |
|---|---|---|
| Underlay | Nexus SVIs route between VLANs | Familiar data center segmentation model |
| Cilium pod networking | VXLAN tunnel mode | Underlay routes node IPs, not PodCIDRs |
| LoadBalancer exposure | Cilium LB IPAM plus L2 announcements | VIPs are on the same VLAN as workers and external connector |
| BGP | Not used in this topology | Avoid mixing native PodCIDR routing with tunneled pod networking |

If you later convert this into a BGP lab, change the design intentionally:

1. Use Cilium native routing, not VXLAN tunnel mode.
2. Advertise PodCIDRs or LoadBalancer VIPs to the Nexus with Cilium BGP Control Plane.
3. Make sure the Nexus has return routes for every advertised prefix.
4. Do not announce a LoadBalancer pool via L2 and BGP at the same time unless you are deliberately teaching the failure mode.
5. Keep route ownership clear: L2 announcements for same-subnet VIPs, BGP for routed VIPs and PodCIDRs.

## Required Traffic Matrix

Use this matrix to verify the Nexus ACL intent.

| Source | Destination | Protocol | Action | Reason |
|---|---|---|---|---|
| External connector | LoadBalancer IPs `198.18.1.100-149` | TCP app ports | Permit | External app access |
| External connector | `10.10.30.10` | Any | Deny | Protect Kubernetes API/control plane |
| External connector | `10.10.40.10` | Any | Deny | Protect database |
| Management | All Ubuntu nodes | TCP `22` | Permit | Ansible SSH |
| Management | `10.10.30.10` | TCP `6443` | Permit | kubectl / Kubernetes API |
| Management | `10.10.40.10` | TCP `5432` | Optional permit | DB validation |
| Workers | `10.10.30.10` | TCP `6443` | Permit | k3s agents to Kubernetes API |
| Workers | `10.10.30.10` | UDP `8472` | Permit | Cilium VXLAN to control-plane node |
| Workers | `10.10.40.10` | TCP `5432` | Permit | App-to-database traffic |
| Workers | Management VLAN | Any | Deny | Keep workers out of management |
| Workers | Database VLAN except TCP `5432` | Any | Deny | Restrict backend access |

Kubernetes also commonly uses TCP `10250` for kubelet API access. In k3s, the apiserver often uses the k3s agent tunnel for some node access patterns. If you disable or change that behavior, explicitly validate whether the control-plane node needs direct TCP `10250` access to workers and add the matching policy.

## Validation Checklist

Run these checks before the demo.

On the Nexus:

```text
show vlan brief
show ip interface brief
show ip route
show ip access-lists ACL-EXT-IN
show ip access-lists ACL-WORKER-IN
show ip access-lists ACL-MGMT-IN
show interface status
```

From the management node:

```bash
ansible all -i ansible/nexus-segmented-k3s-cilium/inventory.ini -m ping
ssh cisco@10.10.30.10 hostname
ssh cisco@198.18.1.21 hostname
ssh cisco@198.18.1.22 hostname
ssh cisco@10.10.40.10 hostname
kubectl --kubeconfig ~/.kube/cml-k3s.yaml get nodes -o wide
```

From worker nodes:

```bash
curl -k https://10.10.30.10:6443/readyz
nc -vz 10.10.40.10 5432
```

From an outside host on `198.18.1.0/24`:

```bash
curl http://198.18.1.100/
nc -vz 10.10.30.10 6443
nc -vz 10.10.40.10 5432
```

Expected result:

1. LoadBalancer IP access succeeds.
2. Direct external access to the control plane fails.
3. Direct external access to PostgreSQL fails.
4. Management SSH and Kubernetes API access succeeds.
5. Worker-to-control-plane and worker-to-database access succeeds only on allowed ports.

From Kubernetes:

```bash
kubectl get nodes -o wide
kubectl -n kube-system get pods -o wide
kubectl -n kube-system exec ds/cilium -- cilium-dbg status
kubectl get CiliumLoadBalancerIPPool
kubectl get CiliumL2AnnouncementPolicy
kubectl -n kube-system get lease | grep cilium-l2announce
```

If Hubble is enabled:

```bash
cilium hubble enable
cilium hubble port-forward &
hubble observe --from-pod default/<pod-name>
```

Use Hubble to show the workload flow to PostgreSQL, then use the Nexus ACL counters to show the underlay policy boundary. That gives the audience both cloud-native and traditional network visibility.

## Suggested Ansible Playbook Responsibilities

Keep the playbooks narrow and demo-friendly:

| Playbook | Responsibility |
|---|---|
| `00-preflight.yml` | Verify SSH, hostname resolution, Python, and routes from management to all nodes. |
| `10-linux-base.yml` | Configure hostnames, `/etc/hosts`, kernel modules, sysctls, and common packages. |
| `20-postgres.yml` | Format/mount the 512 MB data volume and install/configure PostgreSQL. |
| `30-k3s-server.yml` | Install k3s server with Flannel, network policy, kube-proxy, and Traefik disabled. |
| `31-k3s-agents.yml` | Join worker nodes to the k3s server. |
| `40-cilium.yml` | Install Cilium, LB IPAM, L2 announcement policy, and Hubble. |
| `50-validate.yml` | Run connectivity and Kubernetes validation checks. |

Make the Ansible control path obvious in the talk: the management node can reach every node for provisioning, but outside clients can only reach published LoadBalancer services.

## Demo Storyline

1. Show the flat worker/LB segment on `198.18.1.0/24`.
2. Show the control plane and database isolated on separate VLANs.
3. Show the Nexus config: VLANs, SVIs, and port role ACLs.
4. From outside the lab, prove that LoadBalancer IPs work.
5. From outside the lab, prove that the control plane and database are blocked.
6. From the management node, prove Ansible and kubectl access works.
7. From a workload, connect to PostgreSQL.
8. Use Hubble to show pod-to-database traffic.
9. Use Nexus ACL counters to show the same flow at the underlay boundary.

## References

- Cisco CML VM images: https://developer.cisco.com/docs/modeling-labs/vm-images-for-cml-labs/
- Cisco CML NX-OS 9000 reference platform: https://developer.cisco.com/docs/modeling-labs/nx-os-9000/
- Cisco CML External Connectors: https://developer.cisco.com/docs/modeling-labs/networking-external-connectors/
- Kubernetes ports and protocols: https://kubernetes.io/docs/reference/networking/ports-and-protocols/
- K3s networking options: https://docs.k3s.io/networking/basic-network-options
- Cilium on k3s: https://docs.cilium.io/en/stable/installation/k3s/
- Cilium routing modes: https://docs.cilium.io/en/stable/network/concepts/routing/
- Cilium L2 announcements: https://docs.cilium.io/en/stable/network/l2-announcements/
