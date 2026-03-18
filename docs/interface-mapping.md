# Node-to-Interface and IP Mapping (Phase 1)

This document lists every node, interface, peer, IP address, subnet, and default gateway for the foundation lab. CML interface numbering is 0-based (slot 0 = first interface).

**Assumption:** Site A and Site B use point-to-point links between the site router and each cluster node (no L2 switch). Each link uses a /30 from the site’s /24. If you later add an L2 switch per site, you can use a single /24 segment and the preferred addresses (e.g. k8s-a-cp 10.10.0.11/24, gateway 10.10.0.1).

---

## 1. wan-rtr (Cisco IOSv)

| Interface (CML slot) | Cisco name      | Connected to | IP address   | Subnet        |
|----------------------|-----------------|--------------|--------------|---------------|
| 0                     | GigabitEthernet0/0 | jump-1    | 10.30.0.1    | 10.30.0.0/24  |
| 1                     | GigabitEthernet0/1 | site-a-rtr | 10.255.0.1   | 10.255.0.0/30 |
| 2                     | GigabitEthernet0/2 | site-b-rtr | 10.255.0.5   | 10.255.0.4/30 |
| 3                     | GigabitEthernet0/3 | external   | 192.0.2.1    | 192.0.2.0/24  |

Static default route: 0.0.0.0/0 via 192.0.2.2 (external gateway). WAN is the single default-exit point; it originates 0.0.0.0/0 to site routers via BGP.

---

## 2. site-a-rtr (Cisco IOSv)

| Interface (CML slot) | Cisco name      | Connected to | IP address   | Subnet        |
|----------------------|-----------------|--------------|--------------|---------------|
| 0                     | GigabitEthernet0/0 | wan-rtr    | 10.255.0.2   | 10.255.0.0/30 |
| 1                     | GigabitEthernet0/1 | k8s-a-cp   | 10.10.0.1    | 10.10.0.0/30  |
| 2                     | GigabitEthernet0/2 | k8s-a-w1   | 10.10.0.5    | 10.10.0.4/30  |
| 3                     | GigabitEthernet0/3 | k8s-a-w2   | 10.10.0.9    | 10.10.0.8/30  |

No default gateway (router).

---

## 3. site-b-rtr (Cisco IOSv)

| Interface (CML slot) | Cisco name      | Connected to | IP address   | Subnet        |
|----------------------|-----------------|--------------|--------------|---------------|
| 0                     | GigabitEthernet0/0 | wan-rtr    | 10.255.0.6   | 10.255.0.4/30 |
| 1                     | GigabitEthernet0/1 | k8s-b-cp   | 10.20.0.1    | 10.20.0.0/30  |
| 2                     | GigabitEthernet0/2 | k8s-b-w1   | 10.20.0.5    | 10.20.0.4/30  |
| 3                     | GigabitEthernet0/3 | k8s-b-w2   | 10.20.0.9    | 10.20.0.8/30  |

No default gateway (router).

---

## 4. jump-1 (Linux)

| Interface (CML slot) | Linux name | Connected to | IP address | Subnet         | Default gateway |
|----------------------|------------|-------------|------------|----------------|-----------------|
| 0                     | eth0       | wan-rtr     | 10.30.0.10 | 10.30.0.0/24   | 10.30.0.1       |

---

## 5. k8s-a-cp (Linux)

| Interface (CML slot) | Linux name | Connected to  | IP address | Subnet         | Default gateway |
|----------------------|------------|--------------|------------|----------------|-----------------|
| 0                     | eth0       | site-a-rtr   | 10.10.0.2  | 10.10.0.0/30   | 10.10.0.1       |

---

## 6. k8s-a-w1 (Linux)

| Interface (CML slot) | Linux name | Connected to  | IP address | Subnet         | Default gateway |
|----------------------|------------|--------------|------------|----------------|-----------------|
| 0                     | eth0       | site-a-rtr   | 10.10.0.6  | 10.10.0.4/30   | 10.10.0.5       |

---

## 7. k8s-a-w2 (Linux)

| Interface (CML slot) | Linux name | Connected to  | IP address  | Subnet         | Default gateway |
|----------------------|------------|--------------|-------------|----------------|-----------------|
| 0                     | eth0       | site-a-rtr   | 10.10.0.10 | 10.10.0.8/30   | 10.10.0.9       |

---

## 8. k8s-b-cp (Linux)

| Interface (CML slot) | Linux name | Connected to  | IP address | Subnet         | Default gateway |
|----------------------|------------|--------------|------------|----------------|-----------------|
| 0                     | eth0       | site-b-rtr   | 10.20.0.2  | 10.20.0.0/30   | 10.20.0.1       |

---

## 9. k8s-b-w1 (Linux)

| Interface (CML slot) | Linux name | Connected to  | IP address | Subnet         | Default gateway |
|----------------------|------------|--------------|------------|----------------|-----------------|
| 0                     | eth0       | site-b-rtr   | 10.20.0.6  | 10.20.0.4/30   | 10.20.0.5       |

---

## 10. k8s-b-w2 (Linux)

| Interface (CML slot) | Linux name | Connected to  | IP address  | Subnet         | Default gateway |
|----------------------|------------|--------------|-------------|----------------|-----------------|
| 0                     | eth0       | site-b-rtr   | 10.20.0.10 | 10.20.0.8/30   | 10.20.0.9       |

---

## 11. external (CML External Connector – Internet/egress)

| Interface (CML slot) | Connected to | IP address | Notes |
|----------------------|--------------|------------|--------|
| 0                     | wan-rtr     | none       | CML External Connector node; bridges to NAT or System Bridge on the CML server. No VM, no IP on this node. |

The WAN router uses 192.0.2.1/24 on Gi0/3 and has a static default route to **192.0.2.2** (external gateway). The address 192.0.2.2 is provided by the CML server’s external bridge (e.g. NAT gateway or System Bridge) or by an upstream router on that segment. All Linux nodes use their local router as default gateway; traffic to unknown destinations flows site-router → wan-rtr → 192.0.2.2 (external side of the bridge).

---

## Summary by subnet

| Subnet          | Purpose              | Nodes / addresses |
|-----------------|----------------------|--------------------|
| 10.30.0.0/24    | Shared services      | wan-rtr .1, jump-1 .10 |
| 10.255.0.0/30   | WAN wan–site-a       | wan-rtr .1, site-a-rtr .2 |
| 10.255.0.4/30   | WAN wan–site-b       | wan-rtr .5, site-b-rtr .6 |
| 10.10.0.0/30    | Site A rtr–k8s-a-cp   | site-a-rtr .1, k8s-a-cp .2 |
| 10.10.0.4/30    | Site A rtr–k8s-a-w1   | site-a-rtr .5, k8s-a-w1 .6 |
| 10.10.0.8/30    | Site A rtr–k8s-a-w2   | site-a-rtr .9, k8s-a-w2 .10 |
| 10.20.0.0/30    | Site B rtr–k8s-b-cp   | site-b-rtr .1, k8s-b-cp .2 |
| 10.20.0.4/30    | Site B rtr–k8s-b-w1   | site-b-rtr .5, k8s-b-w1 .6 |
| 10.20.0.8/30    | Site B rtr–k8s-b-w2   | site-b-rtr .9, k8s-b-w2 .10 |
| 192.0.2.0/24    | External/Internet transit | wan-rtr .1; gateway .2 on external side (CML bridge/upstream) |

---

## Reserved for later phases (do not assign in Phase 1)

- **Pod CIDRs:** Cluster A 10.42.0.0/16, Cluster B 10.43.0.0/16  
- **Service CIDRs:** Cluster A 10.52.0.0/16, Cluster B 10.53.0.0/16  
- **LoadBalancer pools:** Cluster A 10.10.100.0/24, Cluster B 10.20.100.0/24  
