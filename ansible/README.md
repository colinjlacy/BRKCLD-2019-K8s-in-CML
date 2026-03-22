# Ansible: K3s bootstrap from jump-1

Run this after Phase 1 routing works and all Linux nodes have finished cloud-init (hostname, `/etc/hosts`, packages).

## Automated run (default)

**jump-1** cloud-init waits until **all six** K8s nodes accept **TCP 22** (up to **20 minutes**). If that wait times out, the bootstrap script **exits with failure** and does **not** run Ansible (avoids SSH UNREACHABLE on slow nodes). After the wait succeeds, it clones [BRKCLD-2019-K8s-in-CML](https://github.com/colinjlacy/BRKCLD-2019-K8s-in-CML.git) **branch `ansible`** into `/home/cisco/BRKCLD-2019-K8s-in-CML`, then runs `ansible-playbook site.yml` with a **2-hour** wall-clock limit. The playbook’s first play waits again (**1200s** per host, **`any_errors_fatal`**) before any real SSH tasks. Logs: `/var/log/jump-bootstrap.log`.

## Manual run (optional)

- Log in to **jump-1** as `cisco`.
- Either use the cloned repo at `~/BRKCLD-2019-K8s-in-CML/ansible` or copy only this directory, then:

```bash
cd ~/BRKCLD-2019-K8s-in-CML/ansible
ansible-playbook site.yml
```

`ansible.cfg` and `inventory.ini` are read from the current directory. SSH uses `group_vars/all.yml` (lab password for user `cisco`).

## What it does

1. Waits for SSH on all K8s nodes.
2. Host prep (modules, sysctl, swap, curl) on every cluster node.
3. Installs K3s server on **k8s-a-cp** and **k8s-b-cp** in **parallel** (same play, `hosts: k8s_control`).
4. Waits for TCP **6443** on both control planes in **parallel** (async `wait_for`).
5. Reads join tokens from both control planes in **parallel**; installs **k3s-agent** on **all workers** in **parallel** (one play, `hosts: k8s_workers`).
6. On jump-1: installs **`kubectl`**, pulls kubeconfig into `~/.kube/k3s-a.yaml` and `k3s-b.yaml`, runs `kubectl get nodes` for each.
7. Installs **Helm**, **Cilium** (kube-proxy replacement, tunnel mode, Hubble + relay + UI) on **both** clusters, **cilium** and **hubble** CLIs on jump.
8. **`cilium status --wait`**, asserts nodes **Ready**, prints **CEP**, runs **Service + DNS** checks (nginx + curl + nslookup), then deletes transient workloads.

**K3s CIDRs** (on each control-plane host in `inventory.ini`): Cluster A pods `10.42.0.0/16`, services `10.52.0.0/16`; Cluster B pods `10.43.0.0/16`, services `10.53.0.0/16`. If K3s was already installed without these flags, remove `/etc/systemd/system/k3s.service` on that control plane and re-run (or rebuild the node).

**Hubble UI from jump (manual, after playbook):**

```bash
KUBECONFIG=~/.kube/k3s-a.yaml kubectl -n kube-system port-forward svc/hubble-ui 12001:80
# browse http://127.0.0.1:12001
KUBECONFIG=~/.kube/k3s-b.yaml kubectl -n kube-system port-forward svc/hubble-ui 12002:80
```

## Credentials

`group_vars/all.yml` sets `ansible_ssh_pass` for the CML image. Replace with Vault or SSH keys for anything outside an isolated lab.

## Troubleshooting

- **`kubectl` download fails:** jump-1 needs HTTPS to **dl.k8s.io**; check proxy/DNS. The playbook uses the stable version from **https://dl.k8s.io/release/stable.txt**.
- **Permission denied (sudo):** CML `cisco` user usually has passwordless sudo; if not, set `ansible_become_pass` in `group_vars/all.yml`.
