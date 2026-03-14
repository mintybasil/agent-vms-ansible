# AGENTS.md — AI Agent Guide for openclaw-ansible

This repo contains Ansible automation for deploying **hardened VMs running OpenClaw** on a Debian 13 host. It covers host setup, VM provisioning, firewall config, Caddy reverse proxy, Tailscale networking, and optional Promtail log shipping.

---

## Repo Structure

```
playbooks/          # Top-level playbooks (entry points)
roles/              # Reusable Ansible roles
inventory/          # Hosts and group_vars (hosts.yml is gitignored)
docs/               # Setup guides and reference configs
requirements.yml    # Ansible Galaxy collections
ansible.cfg         # Project-level Ansible config
```

### Playbooks

| Playbook | Target | Purpose |
|---|---|---|
| `host-setup.yml` | `openclaw_host` | Install Tailscale + Promtail on bare metal host |
| `host-deploy.yml` | `openclaw_host` | Provision KVM VM via cloud-init |
| `vm-setup.yml` | `openclaw_vms` | Configure Caddy inside the VM |
| `firewall-setup.yml` | `openclaw_host` | Apply nftables firewall rules to host |

### Roles

| Role | Purpose |
|---|---|
| `host_tailscale` | Install and auth Tailscale on host |
| `host_promtail` | Deploy Promtail as a systemd service |
| `host_nftables` | Configure nftables firewall on host |
| `host_vm_deploy` | Create one or more KVM VMs using libvirt + cloud-init |
| `vm_setup` | VM-level network/interface config |
| `caddy` | Install Caddy + generate Caddyfile from template |

The `host_vm_deploy` role loops over a `vms` list. One-time host resources (packages,
the libvirt `default` NAT network, the storage pool, the base Debian cloud image) are
set up once; per-VM resources (disk copy, cloud-init ISO, libvirt domain) are created
for each entry in the list.

---

## Inventory

- `inventory/hosts.yml` — **gitignored**, contains real IPs/hostnames. Copy from `hosts.example.yml`.
- `inventory/group_vars/all.example.yml` — copy to `all.yml`, fill in required vars.

Two host groups:
- `openclaw_host` — the bare metal Debian 13 machine running KVM
- `openclaw_vms` — the VM running OpenClaw

---

## Key Variables

### Host-level

| Variable | Where | Notes |
|---|---|---|
| `host_user` | group_vars/all.yml | SSH user on host |
| `host_ssh_port` | group_vars/all.yml | Default 22; change it |
| `promtail_enabled` | group_vars/all.yml | Set `true` to enable log shipping |
| `caddy_basicauth_user` | group_vars/all.yml | Basic auth username for Caddy |
| `caddy_basicauth_hash` | group_vars/all.yml | Bcrypt hash via `caddy hash-password` |

Caddy site domain and TLS cert/key paths are derived from `tailscale status --peers=false --json` (field `CertDomains`); no extra vars required.

### VM list (`vms`)

VMs are defined as a list in `group_vars/all.yml`. Each entry creates one libvirt domain.

| Field | Required | Default | Notes |
|---|---|---|---|
| `name` | ✅ | — | VM hostname + libvirt domain name; must be unique |
| `ip` | ✅ | — | Static IP on the libvirt NAT network (192.168.122.x) |
| `ssh_public_key` | ✅ | — | SSH public key for `vm_vm_user` |
| `disk_size` | | `20G` | Passed to `qemu-img resize` |
| `memory_mib` | | `4096` | RAM in MiB |
| `vcpus` | | `4` | vCPU count |
| `network_iface` | | `enp1s0` | Guest NIC name (consistent for virtio-net) |

The `vm_vm_user` global (default: `debian`) is the OS user provisioned inside every VM.

Secrets (`tailscale_auth_key`, passwords) are passed via `-e` at runtime — **never commit them**.

---

## Running Playbooks

```bash
# Install Galaxy collections first
ansible-galaxy collection install -r requirements.yml -p collections/

# Host setup (Tailscale + optional Promtail)
ansible-playbook playbooks/host-setup.yml -e tailscale_auth_key=<key>

# Deploy VM
ansible-playbook playbooks/vm-deploy.yml -e tailscale_auth_key=<key>

# VM Caddy setup
ansible-playbook playbooks/caddy.yml

# Firewall
ansible-playbook playbooks/firewall.yml
```

Use `-C` (check mode) and `-D` (diff) to preview changes before applying:

```bash
ansible-playbook playbooks/firewall.yml -C -D
```

---

## Conventions & Guidelines for AI Agents

### Making Changes

- **Roles are self-contained.** Each role owns its `defaults/`, `tasks/`, `handlers/`, and `templates/`. Don't reach across roles.
- **Defaults over hardcoding.** Variables belong in `roles/<role>/defaults/main.yml`, not hardcoded in tasks.
- **Templates use Jinja2.** Template files live in `templates/` with a `.j2` extension.
- **Handlers for service restarts.** Use `notify:` + handlers rather than restarting services directly in tasks.
- **No secrets in repo.** Vault-encrypt or pass via `-e` at runtime. Check `inventory/group_vars/all.example.yml` for which vars are sensitive.

### Adding a New Role

1. Create `roles/<role_name>/{tasks,defaults,handlers,templates}/`
2. Add a `tasks/main.yml` as the entry point
3. Add `defaults/main.yml` with sensible defaults and comments
4. Wire it into the appropriate playbook under `roles:`
5. Document new variables in `inventory/group_vars/all.example.yml`

### Linting

```bash
# Requires ansible-lint
ansible-lint
```

Fix any `yaml[truthy]`, `name[missing]`, or `no-changed-when` warnings before opening a PR.

### PR Checklist

- [ ] `all.example.yml` updated if new variables were added
- [ ] No secrets or real IPs committed
- [ ] Handlers used for service restarts
- [ ] New roles wired into a playbook
- [ ] `ansible-lint` passes (or violations documented)
- [ ] `README.md` updated if the usage flow changed

---

## Security Notes

- Avoid reusable Tailscale auth keys — a one-time ephemeral key prevents key extraction from a compromised VM.
- `host_ssh_port` should be changed from 22 in production.
- Caddy basic auth hash must be generated with `caddy hash-password`, never stored in plaintext.
- The nftables role (`host_nftables`) is the host firewall — review the template in `roles/host_nftables/templates/nftables.conf.j2` before applying to any host.

---

## Reference Docs

- `docs/hetzner/INSTALL-OS-HETZNER.md` — OS install guide for Hetzner bare metal
- `docs/kvm-libvrt-vm-guide.md` — KVM/libvirt setup reference
- `docs/vm-setup-guide.md` — VM configuration walkthrough
