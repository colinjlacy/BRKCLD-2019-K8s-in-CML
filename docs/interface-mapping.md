# Node-to-Interface and IP Mapping (Phase 1)

This document lists every node, interface, peer, IP address, subnet, and default gateway for the foundation lab. CML interface numbering is 0-based (slot 0 = first interface).

**Design:** Each site uses one **unmanaged L2 switch** between the site router and all three k8s nodes. Site A is a single **10.10.0.0/24** broadcast domain (gateway **10.10.0.1** on the router); Site B is **10.20.0.0/24** (gateway **10.20.0.1**).

---

## 1. wan-rtr (Cisco IOSv)

| Interface (CML slot) | Cisco name      | Connected to | IP address   | Subnet        |
|----------------------|-----------------|--------------|--------------|---------------|
| 0                     | GigabitEthernet0/0 | jump-1    | 10.30.0.1    | 10.30.0.0/24  |
| 1                     | GigabitEthernet0/1 | site-a-rtr | 10.255.0.1   | 10.255.0.0/30 |
| 2                     | GigabitEthernet0/2 | site-b-rtr | 10.255.0.5   | 10.255.0.4/30 |
| 3                     | GigabitEthernet0/3 | external   | 192.168.0.123 | 192.168.0.0/24 |

Static default route: 0.0.0.0/0 via **192.168.0.1** (home router on the same LAN). WAN is the single default-exit point; it originates 0.0.0.0/0 to site routers via BGP.

---

## 2. site-a-rtr (Cisco IOSv)

| Interface (CML slot) | Cisco name      | Connected to | IP address   | Subnet        |
|----------------------|-----------------|--------------|--------------|---------------|
| 0                     | GigabitEthernet0/0 | wan-rtr    | 10.255.0.2   | 10.255.0.0/30 |
| 1                     | GigabitEthernet0/1 | site-a-sw  | 10.10.0.1    | 10.10.0.0/24  |
| 2                     | GigabitEthernet0/2 | —          | —            | unused (shutdown) |
| 3                     | GigabitEthernet0/3 | —          | —            | unused (shutdown) |

Static default route: **0.0.0.0/0 via 10.255.0.1** (wan-rtr on the WAN /30). Traffic to the Internet and unknown prefixes uses wan-rtr → external_connector / home LAN.

---

## 3. site-b-rtr (Cisco IOSv)

| Interface (CML slot) | Cisco name      | Connected to | IP address   | Subnet        |
|----------------------|-----------------|--------------|--------------|---------------|
| 0                     | GigabitEthernet0/0 | wan-rtr    | 10.255.0.6   | 10.255.0.4/30 |
| 1                     | GigabitEthernet0/1 | site-b-sw  | 10.20.0.1    | 10.20.0.0/24  |
| 2                     | GigabitEthernet0/2 | —          | —            | unused (shutdown) |
| 3                     | GigabitEthernet0/3 | —          | —            | unused (shutdown) |

Static default route: **0.0.0.0/0 via 10.255.0.5** (wan-rtr on the WAN /30). Traffic to the Internet and unknown prefixes uses wan-rtr → external_connector / home LAN.

---

## 4. site-a-sw (CML unmanaged switch)

The CML **`unmanaged_switch`** definition provides **eight** ports (**port0**–**port7**); this lab uses **port0**–**port3** only.

| Port (slot) | Connected to   |
|---------------|----------------|
| 0             | site-a-rtr Gi0/1 |
| 1             | k8s-a-cp ens2 |
| 2             | k8s-a-w1 ens2 |
| 3             | k8s-a-w2 ens2 |

Layer 2 only; all used ports are on the same VLAN (flat segment **10.10.0.0/24**).

---

## 5. site-b-sw (CML unmanaged switch)

The CML **`unmanaged_switch`** definition provides **eight** ports (**port0**–**port7**); this lab uses **port0**–**port3** only.

| Port (slot) | Connected to   |
|---------------|----------------|
| 0             | site-b-rtr Gi0/1 |
| 1             | k8s-b-cp ens2 |
| 2             | k8s-b-w1 ens2 |
| 3             | k8s-b-w2 ens2 |

Layer 2 only; all used ports are on the same VLAN (flat segment **10.20.0.0/24**).

---

## 6. jump-1 (Linux)

| Interface (CML slot) | Linux name | Connected to | IP address | Subnet         | Default gateway |
|----------------------|------------|-------------|------------|----------------|-----------------|
| 0                     | eth0       | wan-rtr     | 10.30.0.10 | 10.30.0.0/24   | 10.30.0.1       |

---

## 7. k8s-a-cp (Linux)

| Interface (CML slot) | Linux name | Connected to  | IP address | Subnet         | Default gateway |
|----------------------|------------|--------------|------------|----------------|-----------------|
| 0                     | eth0       | site-a-sw    | 10.10.0.11 | 10.10.0.0/24   | 10.10.0.1       |

---

## 8. k8s-a-w1 (Linux)

| Interface (CML slot) | Linux name | Connected to  | IP address | Subnet         | Default gateway |
|----------------------|------------|--------------|------------|----------------|-----------------|
| 0                     | eth0       | site-a-sw    | 10.10.0.21 | 10.10.0.0/24   | 10.10.0.1       |

---

## 9. k8s-a-w2 (Linux)

| Interface (CML slot) | Linux name | Connected to  | IP address  | Subnet         | Default gateway |
|----------------------|------------|--------------|-------------|----------------|-----------------|
| 0                     | eth0       | site-a-sw    | 10.10.0.22 | 10.10.0.0/24   | 10.10.0.1       |

---

## 10. k8s-b-cp (Linux)

| Interface (CML slot) | Linux name | Connected to  | IP address | Subnet         | Default gateway |
|----------------------|------------|--------------|------------|----------------|-----------------|
| 0                     | eth0       | site-b-sw    | 10.20.0.11 | 10.20.0.0/24   | 10.20.0.1       |

---

## 11. k8s-b-w1 (Linux)

| Interface (CML slot) | Linux name | Connected to  | IP address | Subnet         | Default gateway |
|----------------------|------------|--------------|------------|----------------|-----------------|
| 0                     | eth0       | site-b-sw    | 10.20.0.21 | 10.20.0.0/24   | 10.20.0.1       |

---

## 12. k8s-b-w2 (Linux)

| Interface (CML slot) | Linux name | Connected to  | IP address  | Subnet         | Default gateway |
|----------------------|------------|--------------|-------------|----------------|-----------------|
| 0                     | eth0       | site-b-sw    | 10.20.0.22 | 10.20.0.0/24   | 10.20.0.1       |

---

## 13. external (CML External Connector – Internet/egress)

| Interface (CML slot) | Connected to | IP address | Notes |
|----------------------|--------------|------------|--------|
| 0                     | wan-rtr     | none       | CML External Connector node; bridges to NAT or System Bridge on the CML server. No VM, no IP on this node. |

The WAN router uses **192.168.0.123/24** on Gi0/3 (same subnet as the home router) and has a static default route to **192.168.0.1** (typical home-router address—adjust if yours differs). The External Connector bridges this interface to your LAN so return traffic can route back. All Linux nodes use their local router as default gateway; traffic to unknown destinations flows site-router → wan-rtr → home router.

---

## Summary by subnet

| Subnet          | Purpose              | Nodes / addresses |
|-----------------|----------------------|--------------------|
| 10.30.0.0/24    | Shared services      | wan-rtr .1, jump-1 .10 |
| 10.255.0.0/30   | WAN wan–site-a       | wan-rtr .1, site-a-rtr .2 |
| 10.255.0.4/30   | WAN wan–site-b       | wan-rtr .5, site-b-rtr .6 |
| 10.10.0.0/24    | Site A k8s LAN       | site-a-rtr .1 (GW); k8s-a-cp .11, k8s-a-w1 .21, k8s-a-w2 .22 |
| 10.20.0.0/24    | Site B k8s LAN       | site-b-rtr .1 (GW); k8s-b-cp .11, k8s-b-w1 .21, k8s-b-w2 .22 |
| 192.168.0.0/24  | External / home LAN       | wan-rtr Gi0/3 **.123**; default route to home router **.1** (CML External Connector bridges to your LAN) |

---

## Reserved for later phases (do not assign in Phase 1)

- **Pod CIDRs:** Cluster A 10.42.0.0/16, Cluster B 10.43.0.0/16  
- **Service CIDRs:** Cluster A 10.52.0.0/16, Cluster B 10.53.0.0/16  
- **LoadBalancer pools:** Cluster A 10.10.100.0/24, Cluster B 10.20.100.0/24  
