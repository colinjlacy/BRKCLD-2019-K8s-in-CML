# DNS cache on jump-1 — what we tried (lab topology)

This note summarizes work on the **foundation lab** (`lab/topology.yaml`) to use **jump-1** (`10.30.0.10`) as a **central DNS cache** for Ubuntu / Kubernetes nodes, so image pulls and package installs do not all hammer upstream resolvers directly.

---

## Goal

- Run **`dnsmasq`** on **jump-1**, listening on the shared-services interface (**`ens2`**, **`10.30.0.10`**).
- Point **jump-1** and **all six cluster nodes** at that address **first**, with a **fallback** resolver for bootstrap windows when the cache is not up yet.

---

## What we changed in the topology

### Jump-1 (`10.30.0.10`)

- **`write_files`**: `/etc/dnsmasq.d/lab-dns-cache.conf`
  - **`no-resolv`** and explicit **`server=`** lines toward **198.18.133.1** (CML) and public resolvers (e.g. **1.1.1.1**, **9.9.9.9**, **8.8.8.8**).
  - **`cache-size`** and **`dns-forward-max`** raised to handle bursts (many parallel image pulls).
- **`runcmd`**: `netplan apply` → install **`dnsmasq`** (with Ansible deps) → **`systemctl enable` / `restart dnsmasq`** → Ansible bootstrap script.
- **Netplan** on jump: **`nameservers.addresses`**: **`10.30.0.10`**, then **`198.18.133.1`** (same pattern as workers so jump can use the cache when it is healthy).

### Six Kubernetes nodes (site A / site B)

- **Netplan**: **`10.30.0.10`** first, **`198.18.133.1`** second (so **`cloud-init`** / **`apt`** still resolve if jump is slow to start).

### Documentation

- **`docs/interface-mapping.md`**: short note that Linux nodes prefer **jump’s** cache, then CML’s resolver.

---

## Iterations / lessons (why nodes “fell back” to 198.18.133.1)

1. **`systemd-resolved`** on Ubuntu tracks per-link DNS quality. If **`10.30.0.10`** **timeouts** or misbehaves, **resolved** marks it bad and **prefers the next** server (**198.18.133.1**). That looks like a “revert” in **`resolvectl status`**, but it is **fallback behavior**, not netplan deleting **10.30.0.10**.

2. **First dnsmasq tuning** used a **large EDNS UDP ceiling** (**`edns-packet-max=4096`**). On some **NAT / overlay paths**, oversized UDP DNS answers can **fragment or black-hole**, which again makes **resolved** abandon **10.30.0.10**.

3. **`bind-dynamic`** together with **`listen-address=10.30.0.10`** was **risky** for actually binding where workers send traffic (**`ens2`**). We moved to **`interface=ens2`** + **`bind-interfaces`** so the cache listens on the correct L3 interface.

4. We **dropped** options that were not needed for a simple forwarder (**`strict-order`**, **`domain-needed`**, **`bogus-priv`**) to reduce odd failure modes.

5. We added **`systemctl try-restart systemd-resolved || true`** after **`netplan apply`** on **all Ubuntu nodes**, and **again on jump after `dnsmasq` restarts**, so the **stub resolver** reloads and **re-tests** **10.30.0.10** once the cache is listening.

6. **`edns-packet-max`** was set to a **conservative 1232** bytes to align with common **“DNS and UDP MTU”** guidance on tunneled paths.

---

## Routing reminder (unchanged lab design)

Workers reach **10.30.0.10** over **site router → WAN router → 10.30.0.0/24**; no extra DNS-specific routes were added beyond what the lab already used for general connectivity.

---

## Outcome (still unresolved for Kubernetes image pulls)

The configuration improved **direct** DNS checks (e.g. **`dig @10.30.0.10`** to registry-related names). **Container runtime / cluster** image pulls can still fail with **`lookup …: Try again`**, which points to **different code paths** (e.g. **containerd** resolver behavior, **ndots**, **concurrent** lookups, **IPv6/AAAA** timing, or **MTU/TCP** to **HTTPS** registries—not only “UDP to **10.30.0.10:53**”).

Further debugging would likely separate:

- **Host DNS** (**resolved** + **`dig`**) vs.
- **kubelet/containerd** pull path (**`/etc/resolv.conf`** inside pull context, **cgroup DNS**, **systemd-resolved** stub vs **127.0.0.53**, **CoreDNS** if in-cluster, etc.).

---

## Your conclusion

I tried to pull images using the local jump-1 as a DNS cache. I even confirmed that the jump-1 IP address, 10.30.0.10, was the only DNS server available using `resolvectl status`, e.g.:

```
cisco@k8s-a-w2:~$ resolvectl status
Global
Protocols: -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
resolv.conf mode: stub

Link 2 (ens2)
Current Scopes: DNS
Protocols: +DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
Current DNS Server: 10.30.0.10
DNS Servers: 10.30.0.10
cisco@k8s-a-w2:~$ dig @10.30.0.10 cdn01.quay.io

; <<>> DiG 9.18.39-0ubuntu0.24.04.2-Ubuntu <<>> @10.30.0.10 cdn01.quay.io
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 31027
;; flags: qr rd ra; QUERY: 1, ANSWER: 5, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;cdn01.quay.io. IN A

;; ANSWER SECTION:
cdn01.quay.io. 0 IN CNAME cdn01.quay.io.edgesuite.net.
cdn01.quay.io.edgesuite.net. 0 IN CNAME a1663.dscd.akamai.net.
a1663.dscd.akamai.net. 20 IN A 23.67.33.137
a1663.dscd.akamai.net. 20 IN A 23.67.33.133
a1663.dscd.akamai.net. 20 IN A 23.67.33.151

;; Query time: 26 msec
;; SERVER: 10.30.0.10#53(10.30.0.10) (UDP)
;; WHEN: Wed May 13 19:44:41 UTC 2026
;; MSG SIZE rcvd: 163
```

However images are still stuck in ImagePullBackoff, with DNS issues showing as the core culprit:

```
Failed to pull image "quay.io/cilium/cilium:v1.19.3@sha256:2e61680593cddca8b6c055f6d4c849d87a26a1c91c7e3b8b56c7fb76ab7b7b10": failed to pull and unpack image "quay.io/cilium/cilium@sha256:2e61680593cddca8b6c055f6d4c849d87a26a1c91c7e3b8b56c7fb76ab7b7b10": failed to copy: httpReadSeeker: failed open: failed to do request: Get "https://quay.io/v2/cilium/cilium/manifests/sha256:2fc150a148f9aa6685cb122d2bc6c6e503bce981135a9963fa2e0971f6bc4161": dial tcp: lookup quay.io: Try again
```
