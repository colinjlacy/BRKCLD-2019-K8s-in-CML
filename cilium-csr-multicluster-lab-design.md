# Multi-Cluster Kubernetes + Cilium + Cisco CSR1000v Lab Design

## Overview

This lab demonstrates how traditional networking and cloud-native networking can work together using:

- Cisco CSR1000v virtual routers
- Kubernetes clusters running on Ubuntu
- Cilium overlay networking
- Cilium BGP Control Plane
- Cilium ClusterMesh
- eBGP between Cisco edge routers

The goal is to create two independent Kubernetes clusters connected through Cisco routers, with BGP distributing pod and service reachability information between the environments.

---

# High-Level Goals

The lab will demonstrate:

1. Cisco CSR routers peering with each other using BGP
2. Cilium advertising Kubernetes PodCIDRs and Service IPs via BGP
3. CSR routers learning Kubernetes routes dynamically
4. Inter-cluster pod-to-pod connectivity
5. Inter-cluster service discovery using Cilium ClusterMesh
6. Integration between traditional network infrastructure and cloud-native networking

---

# Proposed Topology

```text
                 Internet / external_connector
                              |
                      Shared L2 Segment
                          19.18.1.0/24
                --------------------------------
                |                              |
           CSR-A                          CSR-B
        AS 65001                       AS 65002
                |                              |
        10.10.0.0/24                    10.20.0.0/24
         Cluster A                       Cluster B
      Ubuntu + Kubernetes             Ubuntu + Kubernetes
         Cilium Overlay                 Cilium Overlay
```

---

# Addressing Plan

## Shared Router Segment

| Device | Interface | IP |
|---|---|---|
| CSR-A | outside | 19.18.1.2/24 |
| CSR-B | outside | 19.18.1.3/24 |
| external_connector/gateway | shared switch | 19.18.1.1/24 |

---

## Cluster A

| Component | CIDR |
|---|---|
| Node VLAN | 10.10.0.0/24 |
| Router Gateway | 10.10.0.1 |
| Kubernetes Nodes | 10.10.0.11-20 |
| PodCIDR | 10.244.0.0/16 |
| Service CIDR | 10.96.0.0/16 |

---

## Cluster B

| Component | CIDR |
|---|---|
| Node VLAN | 10.20.0.0/24 |
| Router Gateway | 10.20.0.1 |
| Kubernetes Nodes | 10.20.0.11-20 |
| PodCIDR | 10.245.0.0/16 |
| Service CIDR | 10.97.0.0/16 |

---

# Autonomous System Design

| Component | ASN |
|---|---|
| CSR-A | 65001 |
| CSR-B | 65002 |
| Cilium Cluster A | 65101 |
| Cilium Cluster B | 65102 |

---

# BGP Architecture

## Layer 1 — Router-to-Router eBGP

CSR-A and CSR-B peer across the shared 19.18.1.0/24 segment.

Purpose:
- Simulate inter-site WAN routing
- Exchange summarized cluster reachability
- Demonstrate traditional Cisco routing infrastructure

Example:

```text
CSR-A (65001) <------ eBGP ------> CSR-B (65002)
```

---

## Layer 2 — Cilium-to-CSR BGP

Cilium nodes peer with their local CSR router.

Purpose:
- Dynamically advertise PodCIDRs
- Dynamically advertise Service IPs
- Demonstrate Kubernetes-aware routing
- Show cloud-native BGP integration

Example:

```text
Cluster A Nodes (65101)
        |
        |
      CSR-A (65001)

Cluster B Nodes (65102)
        |
        |
      CSR-B (65002)
```

---

# Why Use Both Cisco BGP and Cilium BGP?

This architecture demonstrates clear separation of responsibilities.

## Cisco CSR Routers

The CSR routers provide:

- Routed edge connectivity
- Inter-site routing
- WAN simulation
- Route summarization
- Route policy control
- Traditional infrastructure networking

---

## Cilium BGP Control Plane

Cilium provides:

- Dynamic Kubernetes route advertisement
- PodCIDR advertisement
- Service IP advertisement
- Cloud-native routing awareness
- Kubernetes-native route management

---

# Recommended Deployment Order

## Phase 1 — Base Connectivity

Validate:

- CSR-A ↔ CSR-B connectivity
- Internet connectivity via external_connector
- Node VLAN reachability
- Kubernetes node-to-node communication

Tests:
- Ping router interfaces
- Ping remote node interfaces
- Validate default gateways

---

## Phase 2 — CSR eBGP

Configure:

- eBGP between CSR-A and CSR-B
- Route advertisement for node VLANs

Validate:

```text
CSR-A learns:
10.20.0.0/24

CSR-B learns:
10.10.0.0/24
```

---

## Phase 3 — Kubernetes + Cilium

Deploy:

- Kubernetes clusters
- Cilium in overlay/tunnel mode

Validate:

- Pod networking
- Local service communication
- Cilium health checks

---

# Phase 4 — Cilium BGP Control Plane

Enable:

- Cilium BGP Control Plane
- BGP peering to local CSR router

Advertise:

## Cluster A

```text
10.244.0.0/16
10.96.0.0/16
```

## Cluster B

```text
10.245.0.0/16
10.97.0.0/16
```

Validate:

- CSR routing tables
- BGP learned routes
- Route propagation across both routers

---

# Phase 5 — ClusterMesh

Enable Cilium ClusterMesh between the clusters.

Purpose:
- Cross-cluster pod communication
- Cross-cluster service discovery
- Shared Kubernetes identity model
- Multi-cluster networking

Validate:

- Cross-cluster pod ping
- Cross-cluster DNS resolution
- Global service discovery

---

# Important Overlay Networking Notes

The clusters will use Cilium in overlay/tunnel mode.

This means:

- Pod traffic is encapsulated between nodes
- Node-to-node reachability is critical
- Router reachability must support node InternalIPs
- PodCIDRs still need to be routable between environments

The underlay network must successfully route traffic between cluster node interfaces.

---

# Example CSR Configuration

## CSR-A

```ios
router bgp 65001
 bgp log-neighbor-changes

 neighbor 19.18.1.3 remote-as 65002

 neighbor 10.10.0.11 remote-as 65101
 neighbor 10.10.0.12 remote-as 65101

 address-family ipv4
  neighbor 19.18.1.3 activate
  neighbor 10.10.0.11 activate
  neighbor 10.10.0.12 activate

  network 10.10.0.0 mask 255.255.255.0
 exit-address-family
```

---

## CSR-B

```ios
router bgp 65002
 bgp log-neighbor-changes

 neighbor 19.18.1.2 remote-as 65001

 neighbor 10.20.0.11 remote-as 65102
 neighbor 10.20.0.12 remote-as 65102

 address-family ipv4
  neighbor 19.18.1.2 activate
  neighbor 10.20.0.11 activate
  neighbor 10.20.0.12 activate

  network 10.20.0.0 mask 255.255.255.0
 exit-address-family
```

---

# Example Cilium BGP Enablement

```bash
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set bgpControlPlane.enabled=true
```

---

# Key Demo Narrative

This lab demonstrates:

> Cisco routers provide the routed edge and inter-site BGP fabric, while Cilium participates as a cloud-native BGP speaker to advertise Kubernetes pod and service reachability. ClusterMesh then adds multi-cluster Kubernetes identity and service discovery.

This creates a strong story around:

- Cloud-native networking
- Traditional routing integration
- Kubernetes-aware infrastructure
- BGP-based service exposure
- Multi-cluster networking architecture

---

# Potential Future Enhancements

Possible future expansions:

- Native-routing mode instead of overlay mode
- EVPN/VXLAN integration
- BGP route reflectors
- Dual-stack IPv4/IPv6
- Multi-site failover
- Anycast service advertisement
- eBGP multihop
- Route policies and filtering
- External LoadBalancer IP advertisement
- Observability with Hubble

---

# Suggested Validation Tests

## Routing

- `show ip bgp summary`
- `show ip route bgp`

## Kubernetes

- Cross-cluster pod ping
- Cross-cluster service access
- Global service discovery

## Cilium

- `cilium status`
- `cilium bgp peers`
- `cilium clustermesh status`

---

# Final Recommendation

The recommended architecture is:

- CSR-to-CSR eBGP for site-to-site routing
- Cilium BGP for Kubernetes route advertisement
- ClusterMesh for multi-cluster identity and service discovery

This provides the cleanest demonstration of how Cisco infrastructure and cloud-native networking can operate together in a modern multi-cluster environment.
