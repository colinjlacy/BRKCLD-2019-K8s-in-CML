# Mermaid Diagram Definitions  
## Modeling Cloud-Native Networking with Kubernetes, Cilium, and Cisco Modeling Labs

These Mermaid definitions are intended as first-pass diagram sources for programmatic slide generation.  
They follow the actual lab topology from `topology.yaml` and the demo progression from the session plan. :contentReference[oaicite:0]{index=0} :contentReference[oaicite:1]{index=1}

---

# Diagram 1 — Full Lab Topology

```mermaid
flowchart TB
    ext["External / Internet<br/>192.168.0.0/24"]
    jump["Jump Host<br/>jump-1<br/>10.30.0.10"]
    wan["WAN Router<br/>wan-rtr<br/>AS 64512"]
    sra["Site A Router<br/>site-a-rtr<br/>AS 64513"]
    srb["Site B Router<br/>site-b-rtr<br/>AS 64514"]
    swa["Site A Switch"]
    swb["Site B Switch"]

    subgraph clusterA["Cluster A / Site A LAN<br/>10.10.0.0/24"]
        acp["k8s-a-cp<br/>10.10.0.11"]
        aw1["k8s-a-w1<br/>10.10.0.21"]
        aw2["k8s-a-w2<br/>10.10.0.22"]
    end

    subgraph clusterB["Cluster B / Site B LAN<br/>10.20.0.0/24"]
        bcp["k8s-b-cp<br/>10.20.0.11"]
        bw1["k8s-b-w1<br/>10.20.0.21"]
        bw2["k8s-b-w2<br/>10.20.0.22"]
    end

    ext ---|"outside / default route"| wan
    jump ---|"10.30.0.0/24"| wan
    wan ---|"10.255.0.0/30"| sra
    wan ---|"10.255.0.4/30"| srb

    sra --- swa
    swa --- acp
    swa --- aw1
    swa --- aw2

    srb --- swb
    swb --- bcp
    swb --- bw1
    swb --- bw2
```

---

# Diagram 2 — Pod Networking Fundamentals

```mermaid
flowchart LR
    subgraph nodeA["Node A"]
        subgraph podA["Pod A network namespace"]
            pa["Pod A IP"]
        end
        va["veth"]
        rta["Linux routing"]
    end

    fabric["Cluster routed connectivity"]

    subgraph nodeB["Node B"]
        rtb["Linux routing"]
        vb["veth"]
        subgraph podB["Pod B network namespace"]
            pb["Pod B IP"]
        end
    end

    pa --> va
    va --> rta
    rta --> fabric
    fabric --> rtb
    rtb --> vb
    vb --> pb
```

---

# Diagram 3 — Service VIP and eBPF Load Balancing

```mermaid
flowchart LR
    client["Client pod or external requester"]
    vip["Kubernetes Service VIP"]
    ebpf["eBPF datapath on node"]
    b1["Backend Pod 1"]
    b2["Backend Pod 2"]

    client --> vip
    vip --> ebpf
    ebpf -->|"selected backend"| b1
    ebpf -.->|"alternate backend"| b2
```

---

# Diagram 4A — Observability View with Hubble

```mermaid
flowchart LR
    fe["frontend"]
    be["backend"]
    db["database"]
    hub["Hubble<br/>flow visibility"]

    fe -->|"HTTP request"| be
    be -->|"HTTP request"| db

    fe -. observes .-> hub
    be -. observes .-> hub
    db -. observes .-> hub
```

---

# Diagram 4B — Identity-Based Policy Enforcement

```mermaid
flowchart LR
    fe["frontend"]
    be["backend"]
    db["database"]
    pol["Cilium Policy"]
    hub["Hubble<br/>VERDICT: DROPPED"]

    fe -->|"allowed"| be
    be -->|"evaluated by policy"| pol
    pol -->|"deny"| hub
    be -.-x|"blocked"| db
    fe -.-x|"blocked"| db
```

---

# Diagram 5A — BGP Service Advertisement

```mermaid
flowchart TB
    svc["Cluster A Service VIP<br/>LoadBalancer Service"]
    cilium["Cilium BGP Control Plane"]
    sra["Site A Router"]
    wan["WAN Router"]

    svc --> cilium
    cilium -->|"BGP advertises VIP"| sra
    sra -->|"route propagated"| wan
```

---

# Diagram 5B — Cross-Site Routed Service Reachability

```mermaid
flowchart LR
    src["Cluster B Pod / Client"]
    srb["Site B Router"]
    wan["WAN Router"]
    sra["Site A Router"]
    vip["Cluster A Service VIP"]
    be["Backend Pod"]

    src --> srb
    srb --> wan
    wan --> sra
    sra --> vip
    vip --> be
```

---

# Diagram 5C — Demo 5 Full Context

```mermaid
flowchart LR
    subgraph siteB["Site B"]
        src["Cluster B Pod"]
        srb["Site B Router"]
    end

    subgraph core["WAN"]
        wan["WAN Router"]
    end

    subgraph siteA["Site A"]
        sra["Site A Router"]
        cilium["Cilium BGP Control Plane"]
        vip["Service VIP"]
        be["Backend Pod"]
    end

    src --> srb
    srb --> wan
    wan --> sra
    cilium -->|"advertises VIP"| sra
    sra --> vip
    vip --> be
```

---

# Optional Styling Starter

If your Mermaid renderer supports class definitions, you can extend diagrams like this:

```mermaid
classDef infra fill:#e8f1fb,stroke:#2b5c88,stroke-width:1px;
classDef workload fill:#eef8ea,stroke:#4d7c0f,stroke-width:1px;
classDef service fill:#fff4d6,stroke:#b7791f,stroke-width:1px;
classDef observe fill:#f3e8ff,stroke:#7c3aed,stroke-width:1px;
classDef deny fill:#fde8e8,stroke:#b91c1c,stroke-width:1px;
```

Suggested mapping:
- routers, switches, jump host, external = `infra`
- pods and application tiers = `workload`
- service VIP = `service`
- Hubble and observability elements = `observe`
- drop / deny elements = `deny`

