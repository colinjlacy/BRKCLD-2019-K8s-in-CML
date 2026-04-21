# Cisco Live Session Slides  
## Modeling Cloud-Native Networking with Kubernetes, Cilium, and Cisco Modeling Labs

---

# SECTION 1 — INTRODUCTION

---

# Slide 1 — Title Slide

## Modeling Cloud-Native Networking with Kubernetes, Cilium, and Cisco Modeling Labs

- Your Name
- Cisco Live Session ID
- Cisco Live Event Branding
- (Optional) Contact Info

Notes:
This is your anchor slide. Minimal text.

---

# Slide 2 — Why Kubernetes Networking Feels Different

## Why Does Kubernetes Networking Feel So Confusing?

- Pods don't have MAC addresses
- IP addresses appear and disappear
- No obvious VLANs
- Traffic paths are unclear
- Traditional tools don’t seem to apply

Visual:
Messy/confusing network path illustration.

Purpose:
Create emotional alignment with audience confusion.

---

# Slide 3 — The Core Claim

## Kubernetes Networking Is Linux Networking

Large centered statement:

**Kubernetes networking is Linux networking.**

Supporting bullets:

- Network namespaces
- Virtual interfaces
- Routing tables
- Policy enforcement

Purpose:
This becomes the central thesis of the session.

---

# Slide 4 — How We'll Learn This

## We'll Build Understanding Step-by-Step

1. Kubernetes networking fundamentals  
2. Service networking  
3. Observability  
4. Identity-based policy  
5. Routed integration

Visual:
Progressive staircase or layered model.

Purpose:
Explain the learning structure.

---

# Slide 5 — The Lab We'll Use

## The Lab Environment

Diagram:

```
Jump Host
     |
 WAN Router
   /     \
Site A   Site B
```

Clusters:

```
Cluster A
Cluster B
```

Purpose:
Introduce the master topology diagram.
This diagram will evolve throughout the talk.

---

# SECTION 2 — DEMO 1  
# Kubernetes Networking Fundamentals

---

# Slide 6 — Before Demo 1

## What Is a Pod?

A pod is:

- A Linux network namespace
- With virtual interfaces
- Connected to a node
- Assigned an IP address

Visual:
Namespace + veth diagram.

Purpose:
Set foundation before showing packet flow.

---

# Slide 7 — Before Demo 1

## Pod-to-Pod Packet Walk

Traffic path:

```
Pod A
 → veth
 → Node routing
 → Pod B
```

Visual:
Step-by-step packet path.

Purpose:
Prepare audience to observe traffic behavior.

---

# Slide 8 — Before Demo 1

## Pod CIDRs and Routing

Each node:

- Owns a Pod CIDR
- Routes to other Pod CIDRs
- Participates in cluster routing

Visual:
Multiple nodes with Pod CIDRs.

Purpose:
Introduce routing responsibility model.

---

# Slide 9 — After Demo 1

## Demo 1 Key Takeaway

**Kubernetes networking is automated Linux networking.**

Supporting bullets:

- No magic
- No overlays to memorize
- Just Linux primitives

Purpose:
Reinforce learning immediately after demo.

---

# SECTION 3 — DEMO 2  
# Service Networking

---

# Slide 10 — Before Demo 2

## What Is a Kubernetes Service?

A Service provides:

- A Virtual IP (VIP)
- Backend pods
- Traffic distribution

Visual:

```
Client → Service VIP → Pod
```

Purpose:
Introduce Service abstraction.

---

# Slide 11 — Before Demo 2

## How Service VIPs Work

Traffic flow:

```
Client
 → Service IP
 → Backend Pod
```

Visual:
Load balancing behavior.

Purpose:
Explain internal routing.

---

# Slide 12 — Before Demo 2

## Traditional Networking Analogy

| Kubernetes | Traditional Networking |
|-------------|------------------------|
| Service VIP | Load balancer VIP |
| Pod backend | Real server |
| Node datapath | Distributed load balancer |

Purpose:
Bridge mental models.

---

# Slide 13 — Before Demo 2

## Introducing eBPF

eBPF programs:

- Run inside the Linux kernel
- Select backend pods
- Route traffic efficiently

Visual:
Kernel-level decision point.

Purpose:
Introduce datapath behavior.

---

# Slide 14 — After Demo 2

## Demo 2 Key Takeaway

**Services behave like distributed load balancers.**

Supporting bullets:

- Every node participates
- No centralized device required
- Highly scalable model

Purpose:
Anchor the Service model.

---

# SECTION 4 — DEMO 3  
# Observability with Hubble

---

# Slide 15 — Before Demo 3

## Why Visibility Matters

Without visibility:

- Troubleshooting is guesswork
- Performance tuning is difficult
- Security enforcement is uncertain

Purpose:
Create operational urgency.

---

# Slide 16 — Before Demo 3

## Traditional vs Kubernetes Visibility

| Traditional Tool | Kubernetes Equivalent |
|------------------|----------------------|
| NetFlow | Hubble |
| SPAN | eBPF |
| Firewall logs | Policy verdict logs |

Purpose:
Connect familiar tools.

---

# Slide 17 — Before Demo 3

## What Hubble Shows

Traffic visibility includes:

```
Source → Destination → Verdict
```

Example:

```
frontend → backend → allowed
```

Purpose:
Set expectations for UI.

---

# Slide 18 — Before Demo 3

## Layer 7 Visibility

Hubble can observe:

```
HTTP GET /api
```

Purpose:
Show deep inspection capabilities.

---

# Slide 19 — After Demo 3

## Demo 3 Key Takeaway

**eBPF provides deep, real-time traffic visibility.**

Supporting bullets:

- No packet mirroring required
- Real-time insight
- Application-aware visibility

Purpose:
Solidify observability model.

---

# SECTION 5 — DEMO 4  
# Identity-Based Policy

---

# Slide 20 — Before Demo 4

## Default Kubernetes Behavior

By default:

```
Any pod → Any pod
```

Purpose:
Create surprise moment.

---

# Slide 21 — Before Demo 4

## Introducing Zero Trust

Zero Trust model:

- Deny by default
- Allow explicitly
- Verify continuously

Purpose:
Introduce security mindset.

---

# Slide 22 — Before Demo 4

## Identity vs IP-Based Policy

| IP-Based | Identity-Based |
|----------|----------------|
| Fragile | Portable |
| Static | Dynamic |
| Difficult to maintain | Easier to scale |

Purpose:
Highlight advantages.

---

# Slide 23 — Before Demo 4

## Policy Model

Traffic policy:

```
frontend → backend = allowed
backend → database = denied
```

Visual:
Three-tier diagram.

Purpose:
Prepare for live failure demo.

---

# Slide 24 — After Demo 4

## Demo 4 Key Takeaway

**Policies follow workloads — not IP addresses.**

Supporting bullets:

- Identity-driven security
- Portable policies
- Practical zero trust

Purpose:
Reinforce core security insight.

---

# SECTION 6 — DEMO 5  
# BGP + Routed Integration

---

# Slide 25 — Before Demo 5

## Kubernetes Meets Routing

Services can:

- Receive IP addresses
- Be advertised into the network
- Be reached externally

Purpose:
Bridge worlds.

---

# Slide 26 — Before Demo 5

## Service Advertisement Concept

Flow:

```
Service VIP → BGP → Router
```

Purpose:
Explain integration logic.

---

# Slide 27 — Before Demo 5

## The Routed Traffic Path

Traffic:

```
Client
 → WAN Router
 → Site Router
 → Service VIP
```

Visual:
End-to-end path.

Purpose:
Prepare mental model.

---

# Slide 28 — Before Demo 5

## Multi-Site Communication

Example:

```
Cluster B
 → WAN
 → Cluster A Service
```

Purpose:
Introduce cross-site behavior.

---

# Slide 29 — Before Demo 5

## Kubernetes as a Routing Domain

Key concept:

**Clusters behave like routed networks.**

Purpose:
Set big-picture insight.

---

# Slide 30 — After Demo 5

## Demo 5 Key Takeaway

**Kubernetes integrates directly with routed infrastructure.**

Supporting bullets:

- Service reachability
- BGP integration
- Multi-site routing

Purpose:
Anchor routing concept.

---

# SECTION 7 — WRAP-UP

---

# Slide 31 — What You Learned

## Core Lessons

1 — Kubernetes = Linux networking  
2 — Services = distributed load balancers  
3 — eBPF = visibility  
4 — Policy = zero trust  
5 — BGP = integration  

Purpose:
Summarize learning.

---

# Slide 32 — Why Cisco Modeling Labs Matters

## Build. Test. Break. Learn.

- Safe experimentation
- Reproducible labs
- Realistic environments

Purpose:
Promote lab-driven learning.

---

# Slide 33 — Key Message

## Cloud-Native Networking Builds on What You Already Know

Large centered statement.

Purpose:
Final memory anchor.

---

# Slide 34 — Resources & Next Steps

## Continue Learning

- GitHub repository
- Lab instructions
- Cilium documentation
- Kubernetes documentation

Purpose:
Extend learning.

---

# Slide 35 — Questions

## Questions?

Minimal content.

Purpose:
Close session.