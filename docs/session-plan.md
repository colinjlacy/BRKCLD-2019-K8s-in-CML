# Cisco Live Session Plan  
**Title:** Modeling Cloud-Native Networking with Kubernetes, Cilium, and Cisco Modeling Labs

## Session Goal

Help traditional network engineers understand that:

> Kubernetes networking is fundamentally Linux networking, enhanced with programmable datapaths and identity-based policy — and it can be explored safely using Cisco Modeling Labs.

The session walks through a **progressive series of demos**, each introducing a new capability while reinforcing familiar networking concepts.

---

# Session Structure (60 Minutes)

| Section | Topic | Time |
|---|---|---|
| Introduction | Why Kubernetes networking confuses network engineers | 5 min |
| Demo 1 | Kubernetes networking fundamentals + packet walk | 10 min |
| Demo 2 | Service networking and Cilium eBPF load balancing | 10 min |
| Demo 3 | Observability with Hubble | 10 min |
| Demo 4 | Identity-based zero-trust networking | 10 min |
| Demo 5 | Multi-cluster networking and BGP integration | 12 min |
| Wrap-up | Key lessons + CML lab reproducibility | 3 min |

---

# Core Learning Progression

Each demo introduces **one new idea**, building a mental model gradually.

1. **Kubernetes networking is Linux networking**
2. **Services behave like distributed load balancers**
3. **eBPF gives deep visibility into microservice traffic**
4. **Network policies create identity-based segmentation**
5. **Clusters can integrate directly with routed networks**

Each section should create a **small "oh wow" moment** rather than one large reveal.

---

# Demo 1 – Kubernetes Networking Fundamentals

## Objective

Demystify Kubernetes networking by showing that it is built from familiar Linux primitives.

## Concepts Covered

- Pod network namespaces
- veth pairs
- Node routing tables
- Pod CIDR allocation
- Basic east-west pod connectivity

## Packet Walk

Example flow:

```
Pod A → veth → Node routing → Pod B
```

### Confirm with Hubble

Example:

```
hubble observe --from-pod frontend
```

Output example:

```
frontend → backend
TCP SYN
VERDICT: FORWARDED
```

## Key Takeaway

> Kubernetes networking is not magic — it is automated Linux networking.

---

# Demo 2 – Service Networking and eBPF Load Balancing

## Objective

Explain how Kubernetes services provide load balancing and how Cilium handles this efficiently.

## Concepts Covered

- Kubernetes Service abstraction
- ClusterIP
- Service VIP behavior
- Backend pod selection
- eBPF datapath

## Traffic Flow

```
Client → Service IP → eBPF program → Backend pod
```

## Discussion Points

- No need for centralized load balancer
- Each node participates in service routing
- Distributed data plane

## Networking Analogy

| Kubernetes | Traditional Networking |
|---|---|
| Service VIP | Load balancer virtual IP |
| Pod backend | Real server |
| Node datapath | Distributed load balancer |

---

# Demo 3 – Observability with Hubble

## Objective

Show how eBPF enables deep visibility into application traffic.

## Concepts Covered

- Flow monitoring
- Service identity
- Layer 7 visibility
- Policy verdict tracking

## Example Command

```
hubble observe
```

Example output:

```
world → frontend
frontend → backend
backend → database
```

### Layer 7 example

```
HTTP GET /api
```

## Networking Comparison

| Traditional Tool | Kubernetes Equivalent |
|---|---|
| NetFlow | Hubble flow logs |
| SPAN | eBPF kernel visibility |
| Firewall logs | Policy verdict telemetry |

## Key Insight

> eBPF allows deep visibility into application traffic without packet mirroring.

---

# Demo 4 – Identity-Based Network Policy

## Objective

Demonstrate zero-trust networking using Kubernetes workload identity.

## Default Kubernetes Behavior

```
Any pod → Any pod
```

Which surprises many network engineers.

## Policy Example

Allow only frontend to talk to backend.

Example concept:

```
frontend → backend = allowed
backend → database = denied
```

## Example Policy

```
allow:
  from: frontend
  to: backend
```

## Demonstration

Trigger blocked traffic and observe with Hubble:

```
hubble observe --verdict DROPPED
```

Example output:

```
backend → database
VERDICT: DROPPED
```

## Key Insight

> Policies follow workloads, not IP addresses.

---

# Demo 5 – Multi-Cluster Networking and BGP Integration

## Objective

Show how Kubernetes clusters can participate in routed networks.

## Concepts Covered

- Cilium BGP control plane
- Service advertisement into the network
- Multi-cluster connectivity
- Cluster Mesh

## Example BGP Route

```
10.100.10.20/32 via Kubernetes node
```

Service VIP becomes reachable from outside the cluster.

## Traffic Flow

Example cross-site path:

```
Cluster B Pod
→ Site Router
→ WAN Router
→ Site Router
→ Cluster A Service
```

## Networking Analogy

| Kubernetes Feature | Networking Concept |
|---|---|
| Cluster Mesh | Multi-site routing |
| Service export | Route advertisement |
| Cilium BGP | Control plane integration |

## Key Insight

> Kubernetes clusters can behave like autonomous routing domains.

---

# Final Takeaways

By the end of the session, attendees should understand:

1. Kubernetes networking is built on familiar Linux networking primitives.
2. eBPF enables powerful observability and efficient packet processing.
3. Identity-based policy enables practical zero-trust networking.
4. Kubernetes clusters can integrate with traditional routing using BGP.
5. Cisco Modeling Labs can be used to safely experiment with these architectures.

---

# Key Message to the Audience

> Cloud-native networking does not replace traditional networking knowledge — it builds on it.

Using Cisco Modeling Labs, network engineers can experiment with Kubernetes networking the same way they have historically explored routing and switching technologies.
