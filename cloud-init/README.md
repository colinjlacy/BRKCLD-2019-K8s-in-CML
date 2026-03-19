# Cloud-init (minimal)

Phase 2 **K3s installation is not done in cloud-init**. Use **Ansible** from **jump-1** (`ansible/site.yml`); see [`../ansible/README.md`](../ansible/README.md).

## What cloud-init does

- Sets **hostname** and appends the shared **`/etc/hosts`** block so all Linux nodes resolve each other.
- **jump-1:** installs **ansible**, **sshpass**, **python3**, **curl** so you can run the playbook over SSH with password auth.
- **Cluster nodes:** installs **python3** for Ansible modules.

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

Static IP addressing remains outside cloud-init (netplan / CML).
