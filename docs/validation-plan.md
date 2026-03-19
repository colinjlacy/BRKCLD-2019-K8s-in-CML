# Validation Plan (Phase 1 – Foundation)

Use this sequence to verify interfaces, addressing, BGP, and end-to-end reachability. Run from the routers (console or SSH) and from **jump-1**.

---

## 1. Interface status (routers)

On each router, confirm all used interfaces are up/up.

**wan-rtr:**
```
show ip interface brief
```
Expect: Gi0/0, Gi0/1, Gi0/2, Gi0/3 up, with correct IPs (10.30.0.1, 10.255.0.1, 10.255.0.5, 192.168.0.123).

**site-a-rtr:**
```
show ip interface brief
```
Expect: **Gi0/0, Gi0/1** up/up (**10.255.0.2**, **10.10.0.1**); Gi0/2–3 **administratively down** (unused).

**site-b-rtr:**
```
show ip interface brief
```
Expect: **Gi0/0, Gi0/1** up/up (**10.255.0.6**, **10.20.0.1**); Gi0/2–3 **administratively down** (unused).

---

## 2. IP reachability (router → directly connected)

From **wan-rtr**:
```
ping 10.30.0.10
ping 10.255.0.2
ping 10.255.0.6
ping 192.168.0.1
```

From **site-a-rtr**:
```
ping 10.255.0.1
ping 10.10.0.11
ping 10.10.0.21
ping 10.10.0.22
```

From **site-b-rtr**:
```
ping 10.255.0.5
ping 10.20.0.11
ping 10.20.0.21
ping 10.20.0.22
```

All should succeed.

---

## 3. BGP neighbor state

**wan-rtr:**
```
show ip bgp summary
```
Expect two neighbors (10.255.0.2 AS 64513, 10.255.0.6 AS 64514), state **Established**.

**site-a-rtr:**
```
show ip bgp summary
```
Expect one neighbor (10.255.0.1 AS 64512), state **Established**.

**site-b-rtr:**
```
show ip bgp summary
```
Expect one neighbor (10.255.0.5 AS 64512), state **Established**.

---

## 4. Route table verification

**wan-rtr:**
```
show ip route
```
Expect: **10.10.0.0/24 via 10.255.0.2** and **10.20.0.0/24 via 10.255.0.6** (static to site-a-rtr / site-b-rtr; may also appear via BGP); 10.30.0.0/24, 192.168.0.0/24, and WAN /30s connected; **0.0.0.0/0 via 192.168.0.1** (static).

**site-a-rtr:**
```
show ip route
```
Expect: 10.20.0.0/24 and 10.30.0.0/24 via BGP; **0.0.0.0/0 via 10.255.0.1** (static to wan-rtr; may also appear via BGP); **10.10.0.0/24** and **10.255.0.0/30** connected.

**site-b-rtr:**
```
show ip route
```
Expect: 10.10.0.0/24 and 10.30.0.0/24 via BGP; **0.0.0.0/0 via 10.255.0.5** (static to wan-rtr; may also appear via BGP); **10.20.0.0/24** and **10.255.0.4/30** connected.

---

## 5. End-to-end ping from jump-1

From **jump-1** (default gateway 10.30.0.1), run:

```
ping -c 2 10.30.0.1
ping -c 2 10.255.0.1
ping -c 2 10.255.0.2
ping -c 2 10.255.0.5
ping -c 2 10.255.0.6
ping -c 2 10.10.0.1
ping -c 2 10.10.0.2
ping -c 2 10.10.0.6
ping -c 2 10.10.0.10
ping -c 2 10.20.0.1
ping -c 2 10.20.0.2
ping -c 2 10.20.0.6
ping -c 2 10.20.0.10
ping -c 2 192.168.0.123
ping -c 2 192.168.0.1
```

All should succeed (foundation connectivity from shared services to both sites, all nodes, and external/egress segment).

---

## 6. Site A ↔ Site B infrastructure

From **jump-1**:
```
ping -c 2 10.10.0.2
ping -c 2 10.20.0.2
```

From **k8s-a-cp** (10.10.0.2):
```
ping -c 2 10.20.0.2
ping -c 2 10.30.0.10
```

From **k8s-b-cp** (10.20.0.2):
```
ping -c 2 10.10.0.2
ping -c 2 10.30.0.10
```

These confirm cross-site and shared-services reachability.

**Optional – default exit (egress):** From **wan-rtr**, `ping 192.168.0.1` tests reachability to the home router. From **jump-1** or any cluster node, `ping -c 2 192.168.0.1` tests full egress path (site router → wan-rtr → External Connector → home LAN). Add a return route on the home router to the lab prefixes (e.g. 10.0.0.0/8 or specific subnets) via **192.168.0.123** if you need traffic initiated from the LAN to reach the lab.

---

## BGP design reference (Phase 1)

| Router      | ASN   | Advertised prefix(es) | Peers                          |
|------------|-------|------------------------|--------------------------------|
| wan-rtr    | 64512 | 10.30.0.0/24, 0.0.0.0/0 (default-information originate) | site-a-rtr (64513), site-b-rtr (64514) |
| site-a-rtr | 64513 | 10.10.0.0/24           | wan-rtr (64512)                |
| site-b-rtr | 64514 | 10.20.0.0/24           | wan-rtr (64512)                |

- WAN has static default 0.0.0.0/0 via 192.168.0.1 and originates it to both site routers. **site-a-rtr** and **site-b-rtr** each have a **static default** to **wan-rtr** (10.255.0.1 and 10.255.0.5 respectively) so unknown-destination traffic always has a path to the WAN, then external_connector / home LAN.
- WAN learns 10.10.0.0/24 and 10.20.0.0/24 via BGP and re-advertises them to the other site.
- No BGP to Kubernetes nodes, no LoadBalancer VIPs, no pod routes in this phase.

**Useful BGP checks:**
- `show ip bgp` – full BGP table
- `show ip bgp neighbors 10.255.0.2` – detail for one neighbor (on wan-rtr)
