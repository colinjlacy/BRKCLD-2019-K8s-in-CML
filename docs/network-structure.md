# Kubernetes Networking Lab Design Decisions (Phase 1)

This document summarizes the recommended design choices for the Cisco Modeling Labs Kubernetes networking lab used in the session demos.

---

# 1. External Service Exposure Model

Use **Cilium BGP Control Plane with Kubernetes `LoadBalancer` services**.

### How it works

1. A Kubernetes service is created with type `LoadBalancer`.
2. Cilium allocates a Service VIP from a predefined pool.
3. Cilium advertises that VIP via **BGP** to the local **site router**.
4. The site router propagates reachability across the WAN.
5. External clients or remote clusters can reach the service through normal routing.

### Why this model

- Aligns with traditional **network engineering concepts**
- Demonstrates **Kubernetes integrating with routed networks**
- Avoids introducing additional load-balancer platforms
- Keeps the demo focused on **Cilium networking**

Ingress/Gateway API is **not included in Phase 1** to avoid unnecessary complexity.

---

# 2. Routing Protocol and Peering Design

Use **BGP throughout the topology**.

### Infrastructure BGP

```
Site A Router  <----BGP---->  WAN Router  <----BGP---->  Site B Router
```

### Cilium Service Advertisement

```
Cluster A Nodes  <----BGP---->  Site A Router
Cluster B Nodes  <----BGP---->  Site B Router
```

### Route Advertisements

Route advertisements include:

- **LoadBalancer service VIPs only**

Pod CIDR advertisement is intentionally avoided to keep the routing design simple and focused on service exposure.

---

# 3. IP Addressing Plan

A **simple IPv4-only addressing model** is used for clarity and reproducibility.

## WAN Transit Networks

```
10.255.0.0/30
10.255.0.4/30
```

Used for point-to-point links between WAN and site routers.

---

## Site Infrastructure Networks

| Site | Subnet |
|-----|------|
| Site A | `10.10.0.0/24` |
| Site B | `10.20.0.0/24` |
| Shared Services / Jump Host | `10.30.0.0/24` |

Example node addresses:

```
Site A Router      10.10.0.1
Cluster A CP       10.10.0.11
Cluster A Worker1  10.10.0.21
Cluster A Worker2  10.10.0.22

Site B Router      10.20.0.1
Cluster B CP       10.20.0.11
Cluster B Worker1  10.20.0.21
Cluster B Worker2  10.20.0.22
```

---

## Pod CIDR Allocation

Each cluster receives a **non-overlapping pod network**.

```
Cluster A Pod CIDR   10.42.0.0/16
Cluster B Pod CIDR   10.43.0.0/16
```

---

## Service CIDR Allocation

Each cluster receives its own service network.

```
Cluster A Service CIDR   10.52.0.0/16
Cluster B Service CIDR   10.53.0.0/16
```

---

## LoadBalancer Service Pools

Each site receives a dedicated address pool.

```
Cluster A LoadBalancer Pool   10.10.100.0/24
Cluster B LoadBalancer Pool   10.20.100.0/24
```

Example advertised route:

```
10.10.100.10/32 → Cluster A Service
```

This makes service ownership easy to identify by site.

---

# 4. Dual-Stack Support

Dual-stack (IPv4 + IPv6) will be implemented in **Phase 2**, not Phase 1.

### Reasons

- Reduces complexity for the first lab version
- Simplifies debugging and deployment
- Keeps focus on Kubernetes networking concepts

IPv6 can later be introduced as an extension of the topology.

---

# 5. Failure Demo Scenario

The primary failure demonstration will be **identity-based network policy enforcement**.

### Scenario

```
frontend → backend     (allowed)
backend  → database    (denied)
```

### Demonstration

Traffic is blocked by policy and observed with Hubble.

Example:

```
hubble observe --verdict DROPPED
```

Output example:

```
backend → database
VERDICT: DROPPED
```

### Why this demo

- Clearly demonstrates **zero-trust networking**
- Shows **identity-based policy**
- Provides strong **Hubble observability output**
- Easy for the audience to understand

---

# Optional Secondary Failure Demo

An optional infrastructure failure demo may include:

**Withdrawing a LoadBalancer service route**

Example flow:

```
Service VIP withdrawn from BGP
→ Site router loses route
→ External client loses connectivity
```

This demonstrates how Kubernetes services participate in routed networks.

---

# Lab Summary

| Category | Decision |
|--------|--------|
| Kubernetes distribution | **K3s** |
| CNI | **Cilium** |
| Observability | **Hubble** |
| Service exposure | **Cilium BGP + LoadBalancer services** |
| Routing protocol | **BGP** |
| Addressing | **IPv4 only** |
| Pod CIDRs | `10.42.0.0/16`, `10.43.0.0/16` |
| Service CIDRs | `10.52.0.0/16`, `10.53.0.0/16` |
| LoadBalancer pools | `10.10.100.0/24`, `10.20.100.0/24` |
| Primary failure demo | **Policy drop + Hubble visibility** |
| Dual-stack | **Phase 2 feature** |

---

# Design Philosophy

Lab prioritizes:

- **Clarity over completeness**
- **Networking relevance**
- **Repeatability in Cisco Modeling Labs**
- **Minimal infrastructure complexity**

The goal is to allow network engineers to **observe, understand, and reproduce Kubernetes networking behavior using familiar routing concepts**.