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
- [1Password CLI](https://developer.1password.com/docs/cli/) installed and signed in (`eval $(op signin)`)

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
- Install common packages and apply OS-level config (cgroups, SSH hardening)
- Disable WiFi (all nodes are expected to be on Ethernet)
- Install k3s server on the control plane
- Join all workers to the cluster

WiFi is disabled by default via `dtoverlay=disable-wifi`. To keep WiFi enabled on a specific node, create a host_vars file:

```yaml
# inventory/host_vars/pi-w1.yml
disable_wifi: false
```

## Booting from NVMe

If a Pi has an NVMe drive attached, you can clone the SD card to it and boot from NVMe instead.

From the Pi:

```sh
# Clone SD card to NVMe
sudo dd if=/dev/mmcblk0 of=/dev/nvme0n1 bs=4M status=progress

# Expand partition to use full NVMe
sudo parted /dev/nvme0n1 resizepart 2 100%

# Check and resize filesystem
sudo e2fsck -f /dev/nvme0n1p2
sudo resize2fs /dev/nvme0n1p2

# Set boot order: NVMe first
sudo raspi-config nonint do_boot_order B2

# Reboot
sudo reboot
```

After reboot, verify root is on NVMe with `lsblk`. The SD card can then be removed.

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

## Deploying workloads

Kubernetes manifests live in `k8s/`, organised with Kustomize.

### Longhorn (distributed storage)

Download the Longhorn manifest (one-time):

```sh
curl -Lo k8s/longhorn/longhorn.yaml \
  https://raw.githubusercontent.com/longhorn/longhorn/v1.8.1/deploy/longhorn.yaml
```

Apply and set as default StorageClass:

```sh
kubectl apply -k k8s/
kubectl patch storageclass local-path \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

Wait for Longhorn to be ready (~2-5 min on Pi hardware):

```sh
kubectl -n longhorn-system get pods -w
```

### Forgejo + Postgres

Deployed automatically as part of `kubectl apply -k k8s/`. Postgres and Forgejo use Longhorn-backed PVCs for persistent storage.

Check pod status:

```sh
kubectl -n forgejo get pods -w
```

Forgejo is accessible via Traefik ingress at `http://git.pi.local/`. Add a DNS entry on your router or an `/etc/hosts` entry on your local machine pointing `git.pi.local` to any node IP.

Secrets in `k8s/postgres/secret.yaml` and `k8s/forgejo/secret.yaml` contain placeholder credentials — change these before deploying. SOPS encryption will be added later.

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
├── k8s/
│   ├── longhorn/            # Distributed storage (vendored manifest)
│   ├── postgres/            # Postgres StatefulSet for Forgejo
│   └── forgejo/             # Forgejo Git forge (ingress, config, storage)
└── site.yml                 # Master playbook
```

## Roadmap

- [ ] SOPS encryption for k8s secrets
- [ ] Flux for GitOps (sync manifests from Forgejo)
- [ ] Forgejo push mirror to GitHub
- [ ] Beowulf cluster setup (OpenMPI, NFS shared storage)
