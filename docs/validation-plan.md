# Validation Plan (Phase 1 – Foundation)

Use this sequence to verify interfaces, addressing, BGP, and end-to-end reachability. Run from the routers (console or SSH) and from **jump-1**.

---

## 1. Interface status (routers)

On each router, confirm all used interfaces are up/up.

**wan-rtr:**
```
show ip interface brief
```
Expect: Gi0/0, Gi0/1, Gi0/2, Gi0/3 up, with correct IPs (10.30.0.1, 10.255.0.1, 10.255.0.5, 192.0.2.1).

**site-a-rtr:**
```
show ip interface brief
```
Expect: Gi0/0–Gi0/3 up (10.255.0.2, 10.10.0.1, 10.10.0.5, 10.10.0.9).

**site-b-rtr:**
```
show ip interface brief
```
Expect: Gi0/0–Gi0/3 up (10.255.0.6, 10.20.0.1, 10.20.0.5, 10.20.0.9).

---

## 2. IP reachability (router → directly connected)

From **wan-rtr**:
```
ping 10.30.0.10
ping 10.255.0.2
ping 10.255.0.6
ping 192.0.2.2
```

From **site-a-rtr**:
```
ping 10.255.0.1
ping 10.10.0.2
ping 10.10.0.6
ping 10.10.0.10
```

From **site-b-rtr**:
```
ping 10.255.0.5
ping 10.20.0.2
ping 10.20.0.6
ping 10.20.0.10
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
Expect: 10.10.0.0/24 and 10.20.0.0/24 via BGP; 10.30.0.0/24, 192.0.2.0/24, and WAN /30s connected; **0.0.0.0/0 via 192.0.2.2** (static).

**site-a-rtr:**
```
show ip route
```
Expect: 10.20.0.0/24 and 10.30.0.0/24 via BGP; **0.0.0.0/0 via BGP** (from wan-rtr); 10.10.0.0/24 (or /30s) and 10.255.0.0/30 connected.

**site-b-rtr:**
```
show ip route
```
Expect: 10.10.0.0/24 and 10.30.0.0/24 via BGP; **0.0.0.0/0 via BGP** (from wan-rtr); 10.20.0.0/24 (or /30s) and 10.255.0.4/30 connected.

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
ping -c 2 192.0.2.1
ping -c 2 192.0.2.2
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

**Optional – default exit (egress):** From **wan-rtr**, `ping 192.0.2.2` tests the link to the External Connector’s bridge (succeeds if the CML server or upstream has a host at .2). From **jump-1** or any cluster node, `ping -c 2 192.0.2.2` tests full egress path (site router → wan-rtr → External Connector → bridge); requires 192.0.2.2 to be reachable on the external side (NAT or System Bridge config).

---

## BGP design reference (Phase 1)

| Router      | ASN   | Advertised prefix(es) | Peers                          |
|------------|-------|------------------------|--------------------------------|
| wan-rtr    | 64512 | 10.30.0.0/24, 0.0.0.0/0 (default-information originate) | site-a-rtr (64513), site-b-rtr (64514) |
| site-a-rtr | 64513 | 10.10.0.0/24           | wan-rtr (64512)                |
| site-b-rtr | 64514 | 10.20.0.0/24           | wan-rtr (64512)                |

- WAN has static default 0.0.0.0/0 via 192.0.2.2 and originates it to both site routers so all Linux nodes use WAN as single default exit.
- WAN learns 10.10.0.0/24 and 10.20.0.0/24 via BGP and re-advertises them to the other site.
- No BGP to Kubernetes nodes, no LoadBalancer VIPs, no pod routes in this phase.

**Useful BGP checks:**
- `show ip bgp` – full BGP table
- `show ip bgp neighbors 10.255.0.2` – detail for one neighbor (on wan-rtr)
