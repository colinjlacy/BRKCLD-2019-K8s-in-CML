# Deployment Notes – Foundation (Phase 1)

Ordered procedure to bring up the CML topology, assign images, apply router configs, set Linux IPs, and verify connectivity. No Kubernetes, K3s, Cilium, or application workloads in this phase.

---

## 1. Import or create the CML topology

- **If your CML supports YAML import:** Use **lab/topology.yaml**. Import via Lab Manager (Import Lab) and select the YAML file. Adjust node definitions if your server uses different IDs (e.g. `iosv-l2` instead of `iosv`, or a specific Ubuntu image ID).
- **If you build by hand:** Create a new lab and add **13** nodes with the names and roles below. Add links as in **docs/interface-mapping.md** and **lab/topology.yaml**. Topology: External → WAN Router → (Site A Router | Shared Svcs/jump-1 | Site B Router); each site router connects to an **unmanaged switch** that fans out to the three k8s nodes.

**Node list:**

| Node       | Type (CML nodedefinition) | RAM  | vCPU | Interfaces |
|------------|---------------------------|------|------|------------|
| wan-rtr    | iosv                      | 2048 | 1    | 4          |
| site-a-rtr | iosv                      | 2048 | 1    | 4          |
| site-b-rtr | iosv                      | 2048 | 1    | 4          |
| external   | external_connector        | —    | —    | 1          |
| jump-1     | ubuntu (or your Linux)    | 2048 | 1    | 1          |
| k8s-a-cp   | ubuntu                    | 4096 | 2    | 1          |
| k8s-a-w1   | ubuntu                    | 2048 | 2    | 1          |
| k8s-a-w2   | ubuntu                    | 2048 | 2    | 1          |
| k8s-b-cp   | ubuntu                    | 4096 | 2    | 1          |
| k8s-b-w1   | ubuntu                    | 2048 | 2    | 1          |
| k8s-b-w2   | ubuntu                    | 2048 | 2    | 1          |
| site-a-sw  | unmanaged_switch          | —    | —    | 8 (4 used) |
| site-b-sw  | unmanaged_switch          | —    | —    | 8 (4 used) |

**Links:**

- wan-rtr Gi0/3 ↔ external (External Connector; **192.168.0.123/24** on WAN, same subnet as home router; use **System Bridge** to your LAN or equivalent so return traffic can route via **192.168.0.1**)  
- wan-rtr Gi0/0 ↔ jump-1 eth0  
- wan-rtr Gi0/1 ↔ site-a-rtr Gi0/0  
- wan-rtr Gi0/2 ↔ site-b-rtr Gi0/0  
- site-a-rtr Gi0/1 ↔ site-a-sw port0; site-a-sw port1 ↔ k8s-a-cp; port2 ↔ k8s-a-w1; port3 ↔ k8s-a-w2  
- site-b-rtr Gi0/1 ↔ site-b-sw port0; site-b-sw port1 ↔ k8s-b-cp; port2 ↔ k8s-b-w1; port3 ↔ k8s-b-w2  

CML interface numbering is 0-based (first interface = slot 0). IOSv maps to GigabitEthernet0/0, 0/1, …

---

## 2. Assign images

- **Routers:** Attach your licensed **Cisco IOSv** image to wan-rtr, site-a-rtr, site-b-rtr (via node configuration → Image).
- **Linux nodes:** Attach your preferred **Ubuntu** (or other) server image to jump-1 and all six k8s-* nodes. Ensure the image has SSH and standard networking (e.g. cloud-init or manual config).
- **External Connector:** No image. Prefer **System Bridge** (or a bridge that places Gi0/3 on **192.168.0.0/24**) so **wan-rtr** at **192.168.0.123** can use **192.168.0.1** as default gateway. If your home router is not **192.168.0.1**, change `ip route 0.0.0.0 0.0.0.0` on **wan-rtr** to match. See [External Connectors](https://developer.cisco.com/docs/modeling-labs/2-6/cml-admin-tools-external-connectors/).

---

## 3. Boot order

1. Start **wan-rtr**, **site-a-rtr**, **site-b-rtr**. Wait until they finish booting (console or CML “booted” state).
2. Start **site-a-sw** and **site-b-sw** (or start them with the lab). Start the **external** External Connector (if it does not start with the link), then **jump-1**, **k8s-a-cp**, **k8s-a-w1**, **k8s-a-w2**, **k8s-b-cp**, **k8s-b-w1**, **k8s-b-w2**.
3. Wait until all nodes show “booted” or reachable.

---

## 4. Apply router configs

- Open console (or SSH after enabling SSH and credentials) for each router.
- Paste the contents of **configs/wan-rtr.config**, **configs/site-a-rtr.config**, **configs/site-b-rtr.config** into the respective router (enable/configure terminal, then paste; save with `write memory` or `copy run start`). Add a local user for SSH (e.g. `username admin privilege 15 secret <yourpassword>`) before the `login local` line, or remove `login local` if using passwordless console access.
- Or use CML configuration injection (if available) to push the same configs at boot.

**Verification:** On each router run `show ip interface brief` and `show ip bgp summary` after config apply. All used interfaces should be up; BGP neighbors should reach Established (may take a few seconds).

---

## 5. Configure Linux host IP addresses

If you use **`lab/topology.yaml`** user-data as-is, cloud-init already applies **static Netplan** on **`ens2`** (address, default gateway, DNS) for all Ubuntu nodes—skip this section unless you need to override.

Otherwise, each Linux node gets a single interface with the IP and default gateway from **docs/interface-mapping.md**. Use one of the methods below (adjust **`eth0`** → **`ens2`** if that matches your image).

**Option A – Netplan (Ubuntu 18.04+):** Create/edit `/etc/netplan/01-netcfg.yaml` (or similar) and apply with `sudo netplan apply`.

**jump-1:**
```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [10.30.0.10/24]
      routes:
        - to: default
          via: 10.30.0.1
```

**k8s-a-cp:**
```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [10.10.0.11/24]
      routes:
        - to: default
          via: 10.10.0.1
```

**k8s-a-w1:**
```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [10.10.0.21/24]
      routes:
        - to: default
          via: 10.10.0.1
```

**k8s-a-w2:**
```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [10.10.0.22/24]
      routes:
        - to: default
          via: 10.10.0.1
```

**k8s-b-cp:**
```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [10.20.0.11/24]
      routes:
        - to: default
          via: 10.20.0.1
```

**k8s-b-w1:**
```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [10.20.0.21/24]
      routes:
        - to: default
          via: 10.20.0.1
```

**k8s-b-w2:**
```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses: [10.20.0.22/24]
      routes:
        - to: default
          via: 10.20.0.1
```

**Option B – ip + route commands (temporary):**
```bash
# jump-1
sudo ip addr add 10.30.0.10/24 dev eth0
sudo ip link set eth0 up
sudo ip route add default via 10.30.0.1

# k8s-a-cp
sudo ip addr add 10.10.0.11/24 dev eth0
sudo ip link set eth0 up
sudo ip route add default via 10.10.0.1
# ... (repeat for each host using addresses from interface-mapping.md)
```

---

## 6. Basic connectivity testing

Follow **docs/validation-plan.md** in order:

1. Router `show ip interface brief` and direct pings.  
2. BGP `show ip bgp summary` and `show ip route` (WAN should have default via **192.168.0.1**; site routers should have 0.0.0.0/0 via BGP).  
3. From **jump-1**: ping all router and Linux node IPs, including **192.168.0.123** (wan-rtr outside) and **192.168.0.1** (home router / egress).  
4. From **k8s-a-cp** and **k8s-b-cp**: ping the other site’s CP, jump-1, and **192.168.0.1** to confirm default exit via WAN.  

When all steps pass, the foundation is ready for the next phase (Kubernetes/K3s, Cilium, etc.).

---

## Assumptions

- CML node definitions **iosv** and **ubuntu** exist (or you substitute your image IDs).  
- IOSv images are licensed and suitable for this topology.  
- Linux primary interface is **eth0**; if your image uses a different name (e.g. ens3), adjust the netplan/commands accordingly.  
- Site k8s LANs use **one L2 switch per site** and a single **/24** per site (**10.10.0.0/24**, **10.20.0.0/24**); see **docs/interface-mapping.md** for IPs and gateways.

---

## Notes for next phase

- **Kubernetes/K3s:** Install only on the six k8s-* nodes; use the same interface and default gateways as in this document. Preserve Pod CIDRs (10.42.0.0/16, 10.43.0.0/16) and Service CIDRs (10.52.0.0/16, 10.53.0.0/16) as planned.
- **Cilium BGP:** Later, add BGP peering between Cilium on cluster nodes and the site routers; advertise LoadBalancer VIPs from pools 10.10.100.0/24 (Site A) and 10.20.100.0/24 (Site B). Do not add BGP or these pools in Phase 1.
