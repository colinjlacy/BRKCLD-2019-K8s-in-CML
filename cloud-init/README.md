# Cloud-init (minimal)

Phase 2 **K3s installation** is done by **Ansible**; **jump-1** cloud-init clones the repo (branch **`ansible`**) and runs the playbook automatically. See [`../ansible/README.md`](../ansible/README.md).

## What cloud-init does

- **`system_info.default_user.name: cisco`**, **`password: cisco`**, **`ssh_pwauth: true`** so the CML default account and SSH password login work as expected.
- **Netplan:** **`write_files`** → `/etc/netplan/99-*.yaml`, then **`runcmd`:** **`netplan apply`** first (CML often ignores cloud-config **`network:`**). Static **`ens2`**, default via **10.30.0.1** (jump) or **10.10.0.1** / **10.20.0.1** (cluster); no custom **`nameservers`** (OS / **systemd-resolved** defaults).
- **`apt`:** **`mirrors.kernel.org/ubuntu`** for primary and security (replaces default archive/security hosts). **`apt-get update` / `install`** run in **`runcmd` after `netplan apply`** so DNS works (cloud-init’s package module runs before **`runcmd`**).
- Sets **hostname** and appends the shared **`/etc/hosts`** block.
- **jump-1:** **`apt-get install`** **ansible**, **sshpass**, **python3**, **curl**, **git**; then **`/usr/local/bin/jump-bootstrap-k3s.sh`** (wait for K8s nodes on TCP 22, clone repo branch **`ansible`**, **`ansible-playbook site.yml`**).
- **Cluster nodes:** **`apt-get install`** **python3** for Ansible.

The same content is embedded under `configuration` → `user-data` in [`lab/topology.yaml`](../lab/topology.yaml).

## Files

| File | Node |
|------|------|
| `user-data-jump-1.yaml` | jump-1 |
| `user-data-k8s-a-cp.yaml` | k8s-a-cp |
| `user-data-k8s-a-w1.yaml` | k8s-a-w1 |
| `user-data-k8s-a-w2.yaml` | k8s-a-w2 |
| `user-data-k8s-b-cp.yaml` | k8s-b-cp |
| `user-data-k8s-b-w1.yaml` | k8s-b-w1 |
| `user-data-k8s-b-w2.yaml` | k8s-b-w2 |

Static IPs are defined in the Netplan snippets in each **`user-data`** file.
