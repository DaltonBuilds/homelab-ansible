# homelab-ansible

Ansible playbooks for OS-level configuration of all nodes in my homelab — bare-metal k3s nodes, Proxmox VMs, and LXC containers.

This is one of three repos that make up my homelab infrastructure. Ansible runs after [homelab-terraform](https://github.com/DaltonBuilds/homelab-terraform) provisions the VMs and before [homelab-gitops](https://github.com/DaltonBuilds/homelab-gitops) brings up the Kubernetes workloads.

## Inventory

| Group | Hosts |
|---|---|
| baremetal | gandalf (CP), aragorn, legolas (workers) |
| proxmox_vms | nfs-server, mgmt-plane, gimli |
| proxmox_lxc | garage |

All bare-metal and VM hosts use `ansible_user: dalton`. LXC containers use `ansible_user: root` — Proxmox LXC doesn't support cloud-init user creation, so Terraform injects SSH keys for root only.

## Playbooks

| Playbook | Target | What it does |
|---|---|---|
| `common.yaml` | All hosts | Base packages, user/SSH hardening, node_exporter, timezone |
| `k3s-server.yaml` | Control plane | k3s with Cilium flags, Gateway API CRDs |
| `k3s-agent.yaml` | Workers | k3s agent join |
| `nfs-server.yaml` | nfs-server | ZFS pool + NFS export (`/tank/k8s`) |
| `garage.yaml` | garage | Garage S3 binary, systemd unit, bucket setup |
| `mgmt-plane.yaml` | mgmt-plane | Single-node k3s for the observability cluster |
| `tailscale.yaml` | All hosts | Tailscale installation and auth |

`site.yaml` runs all playbooks in order for a full environment rebuild.

## Notable Decisions

**No UFW or iptables on k3s nodes** — Cilium's eBPF dataplane manages all network policy. Host-level firewalling conflicts with Cilium and is intentionally omitted.

**k3s installed with Cilium flags** — `--flannel-backend=none --disable-network-policy --disable-kube-proxy --disable traefik --disable servicelb`. Cilium replaces kube-proxy via eBPF, and L2 load balancing replaces MetalLB.

**Secrets via Ansible Vault** — `group_vars/all/secrets.yaml` is encrypted. Run playbooks with `--ask-vault-pass`.

## Usage

```bash
# Run base configuration on all hosts
ansible-playbook -i inventory.yaml playbooks/common.yaml --ask-vault-pass

# Full rebuild
ansible-playbook -i inventory.yaml site.yaml --ask-vault-pass
```
