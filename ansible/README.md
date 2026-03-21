# Ansible: K3s bootstrap from jump-1

Run this after Phase 1 routing works and all Linux nodes have finished cloud-init (hostname, `/etc/hosts`, packages).

## Automated run (default)

**jump-1** cloud-init waits for all six K8s nodes to accept **TCP 22** (up to **20 minutes**), clones [BRKCLD-2019-K8s-in-CML](https://github.com/colinjlacy/BRKCLD-2019-K8s-in-CML.git) **branch `ansible`** into `/home/cisco/BRKCLD-2019-K8s-in-CML`, then runs `ansible-playbook site.yml` with a **2-hour** wall-clock limit. Logs: `/var/log/jump-bootstrap.log`.

If the wait times out, the script still continues; the playbook’s first play also waits for SSH.

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
6. On jump-1: installs **`kubectl`** from **dl.k8s.io** (stable release, **amd64** or **arm64**) to **`/usr/local/bin/kubectl`**, pulls kubeconfig into `~/.kube/k3s-a.yaml` and `k3s-b.yaml`, runs `kubectl get nodes` for each.

Nodes stay **NotReady** until Cilium (later phase).

## Credentials

`group_vars/all.yml` sets `ansible_ssh_pass` for the CML image. Replace with Vault or SSH keys for anything outside an isolated lab.

## Troubleshooting

- **`kubectl` download fails:** jump-1 needs HTTPS to **dl.k8s.io**; check proxy/DNS. The playbook uses the stable version from **https://dl.k8s.io/release/stable.txt**.
- **Permission denied (sudo):** CML `cisco` user usually has passwordless sudo; if not, set `ansible_become_pass` in `group_vars/all.yml`.
