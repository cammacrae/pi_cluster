# Pi Cluster

Ansible-managed Raspberry Pi 5 cluster running k3s (lightweight Kubernetes).

## Hardware

| Node | Role | RAM |
|------|------|-----|
| pi-cp1 | k3s server (control plane) | 16 GB |
| pi-w1 | k3s agent (worker) | 8 GB |
| pi-w2 | k3s agent (worker) | 8 GB |
| pi-w3 | k3s agent (worker) | 8 GB |

All nodes run Raspberry Pi OS Lite (64-bit).

## Prerequisites

- Ansible installed on your local machine
- SSH key access to all Pis (`ssh-copy-id pi@<ip>`)
- DHCP reservations configured on your router for each Pi

Install required Ansible collections:

```sh
ansible-galaxy collection install -r requirements.yml
```

## Setup

1. Flash Pi OS Lite (64-bit) to each Pi and enable SSH.

2. Update the placeholder IPs in `inventory/hosts.yml` and `inventory/group_vars/all.yml` to match your DHCP reservations.

3. Run the playbook:

```sh
ansible-playbook site.yml
```

This will:

- Set hostnames and `/etc/hosts` on all nodes
- Install common packages and apply OS-level config (cgroups, iptables legacy mode, SSH hardening)
- Install k3s server on the control plane
- Join all workers to the cluster

## Running against a single node

You can prep individual Pis before the full cluster is available. Use `--limit` to target a specific node:

```sh
ansible-playbook site.yml --limit pi-w1
```

This applies the common role (hostname, packages, cgroups, swap, networking, SSH hardening) and safely skips the k3s plays — the agent role will skip if the control plane hasn't been provisioned yet.

## Resolving hostnames

Nodes advertise themselves via mDNS (avahi). To resolve a node by name:

```sh
avahi-resolve -n pi-w1.local
```

You may need `avahi-utils` on your local machine (`sudo apt install avahi-utils`).

## Verifying the cluster

```sh
ssh pi-cp1 'sudo k3s kubectl get nodes'
```

All 4 nodes should show `Ready`.

## Using kubectl locally

The playbook fetches the kubeconfig to `kubeconfig.yml`. To use it:

```sh
# Replace 127.0.0.1 with your control plane IP
sed -i 's/127.0.0.1/<pi-cp1-ip>/' kubeconfig.yml
export KUBECONFIG=$(pwd)/kubeconfig.yml
kubectl get nodes
```

## Project structure

```
├── ansible.cfg              # Ansible config (inventory path, SSH user, become)
├── inventory/
│   ├── hosts.yml            # Node inventory (IPs and groups)
│   └── group_vars/          # Variables by group
├── roles/
│   ├── common/              # Base OS setup for all nodes
│   ├── k3s_server/          # k3s server install (control plane)
│   └── k3s_agent/           # k3s agent install (workers)
└── site.yml                 # Master playbook
```

## Roadmap

- [ ] Beowulf cluster setup (OpenMPI, NFS shared storage)
