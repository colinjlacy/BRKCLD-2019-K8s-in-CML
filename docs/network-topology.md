# Kubernetes Networking Lab Topology (Cisco Modeling Labs)

This lab environment models two enterprise sites connected over a routed WAN, with each site hosting a Kubernetes cluster. The topology is designed to demonstrate cloud-native networking concepts such as eBPF-powered datapaths, service routing, identity-based policy enforcement, observability with Hubble, and multi-cluster communication using Cilium.

The topology intentionally mirrors how Kubernetes platforms integrate into traditional enterprise networks: clusters sit behind site routers, which are connected through a routed WAN. A shared jump host represents the rest of the enterprise network and is used for administration and traffic generation.

---

# High-Level Topology

```
                     Jump Host / Client
                            |
                     Shared Services Network
                            |
                        WAN Router
                         /      \
                        /        \
                Site A Router   Site B Router
                     |               |
                Cluster A        Cluster B
            (1 CP + 2 Workers) (1 CP + 2 Workers)
```

---

# Node Inventory

The lab contains **10 total nodes**.

## Infrastructure Nodes

| Node | Role |
|-----|-----|
| WAN Router | Inter-site routing between Site A and Site B |
| Site A Router | Edge router for Site A Kubernetes cluster |
| Site B Router | Edge router for Site B Kubernetes cluster |
| Jump Host | Management host, traffic generator, and observability workstation |

Routers are implemented using **Cisco IOSv** to maintain Cisco routing realism.

---

## Kubernetes Nodes

Each site runs an independent Kubernetes cluster.

### Cluster A

| Node | Role |
|-----|-----|
| k8s-a-cp | Kubernetes control plane |
| k8s-a-w1 | Worker node |
| k8s-a-w2 | Worker node |

### Cluster B

| Node | Role |
|-----|-----|
| k8s-b-cp | Kubernetes control plane |
| k8s-b-w1 | Worker node |
| k8s-b-w2 | Worker node |

Each cluster runs:

- Kubernetes
- Cilium CNI
- Hubble observability
- Envoy for service inspection

---

# Network Domains

The topology is divided into four logical networks.

## 1. Shared Services Network

This network connects the jump host to the WAN router.

Purpose:

- Administrative access
- Traffic generation for demos
- Kubernetes management
- Running tools such as `kubectl`, `cilium`, and `hubble`

Example traffic flow:

```
Jump Host → WAN Router → Site Router → Kubernetes Service
```

---

## 2. WAN Transit Network

The WAN router connects Site A and Site B.

Purpose:

- Inter-site routing
- BGP route propagation
- Multi-cluster communication

Example traffic flow:

```
Cluster A Pod → Site A Router → WAN Router → Site B Router → Cluster B Pod
```

---

## 3. Site A Infrastructure Network

This network connects the Site A router to the Kubernetes nodes in Cluster A.

Purpose:

- Node connectivity
- BGP peering with Cilium
- North-south service routing

Example traffic flow:

```
External Client → Site A Router → Worker Node → Pod
```

---

## 4. Site B Infrastructure Network

This network mirrors Site A but hosts Cluster B.

Purpose:

- Cluster B node connectivity
- Cross-site application communication
- Multi-cluster service access

Example traffic flow:

```
Cluster B Pod → Site B Router → WAN Router → Site A Service
```

---

# Kubernetes Networking Architecture

Each cluster uses **Cilium** as the CNI.

Key components include:

- **eBPF datapath** for service load balancing
- **Hubble** for flow observability
- **Cilium Network Policies** for identity-based security
- **Cilium BGP Control Plane** for service route advertisement

Services may be advertised into the network using BGP, allowing external clients to reach Kubernetes workloads through standard routing mechanisms.

Example flow:

```
Client
  ↓
WAN Router
  ↓
Site Router
  ↓
Kubernetes Node
  ↓
eBPF Service Load Balancer
  ↓
Backend Pod
```

---

# Resource Allocation

The topology is designed to operate within a constrained environment.

| Node Type | Count | RAM per Node | vCPU per Node |
|-----------|------|--------------|---------------|
| Control Plane | 2 | 4 GB | 2 |
| Worker | 4 | 2 GB | 2 |
| Routers (IOSv) | 3 | 2 GB | 1 |
| Jump Host | 1 | 2 GB | 1 |

Total resource usage:

- **24 GB RAM**
- **16 vCPU**

This fits within a **30 GB RAM / 20 vCPU** lab environment while leaving headroom for CML overhead.

---

# Demo Capabilities Enabled by This Topology

This topology supports several demonstrations of cloud-native networking concepts.

### Packet Walk Demonstration

```
Jump Host
 → WAN Router
 → Site Router
 → Kubernetes Node
 → Pod
```

### Service Load Balancing

Demonstrates Cilium's eBPF-based service routing.

```
Client → Service VIP → Backend Pods
```

### Observability with Hubble

Real-time visibility into:

- pod-to-pod traffic
- service flows
- policy decisions
- L7 HTTP requests

### Network Policy Enforcement

Identity-based policy enforcement between microservices.

```
frontend → backend (allowed)
backend → database (denied)
```

### Multi-Cluster Communication

Inter-site service access between clusters.

```
Cluster B Pod
 → WAN
 → Cluster A Service
```

---

# Summary

This lab environment provides a compact but realistic model of how Kubernetes networking integrates with traditional enterprise infrastructure.

Key characteristics include:

- Two routed enterprise sites
- Cisco-based routing infrastructure
- Two Kubernetes clusters
- eBPF-powered networking with Cilium
- Flow observability using Hubble
- Multi-cluster service communication

The topology is intentionally designed to be **small enough for personal CML labs while still reflecting real-world enterprise networking architecture**.