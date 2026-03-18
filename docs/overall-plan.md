# Kubernetes Networking Lab Environment (Session Scope)

This document summarizes the **lab environment and architecture** used for the Cisco Live session demonstrations. The lab supports a series of progressive demos designed to explain Kubernetes networking concepts to traditional network engineers.

---

# Session Demo Progression

The session is structured around **five demos**, each introducing one new concept while building on the previous ones.

```
Demo 1  Kubernetes networking fundamentals + packet walk
Demo 2  Service networking and Cilium load balancing
Demo 3  Observability with Hubble
Demo 4  Identity-based network policy (zero trust)
Demo 5  Service exposure using BGP + multi-cluster reachability
```

The demos collectively demonstrate how Kubernetes networking integrates with traditional routing concepts.

---

# Lab Environment Overview

The lab consists of:

- Two Kubernetes clusters
- A routed WAN connecting two sites
- Cisco routers representing enterprise network infrastructure
- A jump host acting as both operator workstation and external client

This environment supports demonstrations of:

- **East-west pod traffic**
- **North-south service access**
- **Cross-cluster communication**
- **Identity-based network policies**
- **BGP-based service advertisement**

---

# Node Inventory

Infrastructure components:

```
1 WAN Router (Cisco IOSv)
1 Site A Router (Cisco IOSv)
1 Site B Router (Cisco IOSv)
1 Jump Host / External Client
```

Kubernetes clusters:

```
Cluster A
  1 Control Plane
  2 Workers

Cluster B
  1 Control Plane
  2 Workers
```

Total nodes:

```
10
```

---

# Resource Allocation

| Node Type | Count | RAM | vCPU |
|---|---|---|---|
| Control Plane | 2 | 4 GB | 2 |
| Worker | 4 | 2 GB | 2 |
| Routers | 3 | 2–3 GB | 1 |
| Jump Host | 1 | 2 GB | 1 |

Estimated totals:

```
RAM   ≈ 24–27 GB
CPU   ≈ 16 vCPU
```

This fits within the available:

```
30 GB RAM
20 vCPU
```

---

# Kubernetes Platform

Kubernetes distribution:

```
K3s
```

Networking stack:

```
Cilium (CNI)
Hubble (Observability)
```

K3s is used because it is lightweight, quick to deploy, and easy for attendees to reproduce in their own environments.

---

# Service Exposure Model

External service exposure uses:

```
Kubernetes LoadBalancer services
+ Cilium BGP Control Plane
```

Workflow:

```
Service created
      ↓
Cilium allocates Service VIP
      ↓
VIP advertised via BGP to site router
      ↓
Route propagated across WAN
      ↓
External client accesses service
```

This approach demonstrates how Kubernetes services can integrate with traditional routed networks.

---

# Routing Model

Routing protocol:

```
BGP
```

Infrastructure peering:

```
Site A Router  <->  WAN Router  <->  Site B Router
```

Service advertisement:

```
Cluster Nodes (Cilium BGP)  <->  Site Router
```

Only **LoadBalancer service VIPs** are advertised in BGP to keep routing simple and focused on service reachability.

---

# IP Addressing Plan

Infrastructure networks:

```
Site A network        10.10.0.0/24
Site B network        10.20.0.0/24
Shared services       10.30.0.0/24
WAN transit           10.255.0.0/30
```

Pod networks:

```
Cluster A pods        10.42.0.0/16
Cluster B pods        10.43.0.0/16
```

Service networks:

```
Cluster A services    10.52.0.0/16
Cluster B services    10.53.0.0/16
```

LoadBalancer pools:

```
Cluster A LB pool     10.10.100.0/24
Cluster B LB pool     10.20.100.0/24
```

---

# Failure Demonstration

The session includes a failure scenario demonstrating **identity-based policy enforcement**.

Scenario:

```
frontend → backend      allowed
backend  → database     denied
```

Traffic is blocked and observed using Hubble.

Example command:

```
hubble observe --verdict DROPPED
```

Example output:

```
backend → database
VERDICT: DROPPED
```

This demonstrates:

- identity-based policy
- zero-trust networking
- observable dataplane enforcement

---

# Core Session Message

The lab reinforces the central message of the session:

> Kubernetes networking is built on familiar networking principles, and Cisco Modeling Labs allows network engineers to explore cloud-native networking using the same lab-driven approach traditionally used for routing and switching technologies.