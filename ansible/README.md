# Ansible: K3s bootstrap from jump-1

Run this after Phase 1 routing works and all Linux nodes have finished cloud-init (hostname, `/etc/hosts`, packages).

## Prerequisites

- Log in to **jump-1** as user `cisco` (CML default).
- Copy this entire `ansible/` directory onto jump-1, e.g.  
  `scp -r ansible cisco@10.30.0.10:~/k8s-lab-ansible`
- On jump-1, cloud-init should have installed **ansible**, **sshpass**, **python3**, and **curl** (see `lab/topology.yaml` for **jump-1**).

## Run

```bash
cd ~/k8s-lab-ansible
ansible-playbook site.yml
```

`ansible.cfg` and `inventory.ini` are read from the current directory. SSH uses `group_vars/all.yml` (lab password for user `cisco`).

## What it does

1. Waits for SSH on all K8s nodes.
2. Host prep (modules, sysctl, swap, curl) on every cluster node.
3. Installs K3s server on **k8s-a-cp** and **k8s-b-cp** (CNI-less, same flags as before).
4. Waits for TCP **6443** on each control plane.
5. Reads join tokens from each control plane; installs agents on workers with correct `--node-ip`.
6. On jump-1: installs **kubectl** (if needed), pulls kubeconfig from both clusters into `~/.kube/k3s-a.yaml` and `k3s-b.yaml`, runs `kubectl get nodes` for each.

Nodes stay **NotReady** until Cilium (later phase).

## Credentials

`group_vars/all.yml` sets `ansible_ssh_pass` for the CML image. Replace with Vault or SSH keys for anything outside an isolated lab.

## Troubleshooting

- **`kubectl` package not found:** enable Ubuntu **universe** or install kubectl from Kubernetes docs; adjust the final play in `site.yml` if needed.
- **Permission denied (sudo):** CML `cisco` user usually has passwordless sudo; if not, set `ansible_become_pass` in `group_vars/all.yml`.
