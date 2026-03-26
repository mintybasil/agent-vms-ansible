# AGENTS.md — AI Agent Guide for openclaw-ansible

This repo contains Ansible automation for deploying **hardened VMs running OpenClaw** on a Debian 13 host. It covers host setup, VM provisioning, firewall config, Caddy reverse proxy, Tailscale networking, and optional Promtail log shipping.

---

## Repo Structure

```
playbooks/          # Top-level playbooks (entry points)
roles/              # Reusable Ansible roles
inventory/          # Inventory files (hosts.yml is gitignored)
docs/               # Setup guides and reference configs
requirements.yml    # Ansible Galaxy collections
ansible.cfg         # Project-level Ansible config
```

### Playbooks

| Playbook | Target | Purpose |
|---|---|---|
| `host-setup.yml` | `openclaw_host` | User, SSH hardening, libvirt, nftables, Tailscale, Promtail (use `--tags` to run specific parts) |
| `vm-deploy.yml` | `openclaw_host` | Provision KVM VM via cloud-init (requires libvirt from host-setup) |
| `vm-setup.yml` | `openclaw_vms` | Configure Caddy (and other VM-level config) inside the VM; use `--tags caddy` for Caddy only |
### Roles

| Role | Purpose |
|---|---|
| `host_setup` | Single role for host config: user, SSH hardening, libvirt (pool/network), nftables, Tailscale, Promtail. Each area is a task file invoked by tag: `user`, `ssh_hardening`, `libvirt`, `nftables`, `tailscale`, `promtail`. |
| `host_vm_deploy` | Create one or more KVM VMs using libvirt + cloud-init; assumes libvirt pool and default NAT network from `host_setup` (tag `libvirt`) |
| `vm_setup` | VM-level network/interface config (e.g. static interfaces); not currently wired into a playbook |
| `caddy` | Install Caddy + generate Caddyfile from template |

One-time host resources (packages, libvirt `default` NAT network, storage pool, base Debian cloud image) are set up by `host_setup` (tag `libvirt`) and `host_vm_deploy` respectively. The `host_vm_deploy` role loops over a `vms` list; per-VM resources (disk copy, cloud-init ISO, libvirt domain) are created for each entry.

---

## Inventory

- `inventory/hosts.yml` — **gitignored**, contains host and VM variables plus real IPs/hostnames. Copy from `hosts.example.yml`.

Two host groups:
- `openclaw_host` — the bare metal Debian 13 machine running KVM
- `openclaw_vms` — the VM running OpenClaw

---

## Key Variables

### Host-level

| Variable | Where | Notes |
|---|---|---|
| `host_user` | inventory/hosts.yml | SSH user on host |
| `host_ssh_port` | inventory/hosts.yml | Default 22; change it |
| `promtail_enabled` | inventory/hosts.yml | Set `true` to enable log shipping |
| `caddy_basicauth_user` | inventory/hosts.yml | Basic auth username for Caddy |
| `caddy_basicauth_hash` | inventory/hosts.yml | Bcrypt hash via `caddy hash-password` |

Caddy site domain and TLS cert/key paths are derived from `tailscale status --peers=false --json` (field `CertDomains`); no extra vars required.

### VM list (`vms`)

VMs are defined as a list in `inventory/hosts.yml`. Each entry creates one libvirt domain.

| Field | Required | Default | Notes                                                |
|---|---|---|------------------------------------------------------|
| `name` | ✅ | — | VM hostname + libvirt domain name; must be unique    |
| `ip` | ✅ | — | Static IP on the libvirt NAT network (192.168.122.x) |
| `ssh_public_key` | ✅ | — | SSH public key for `vm.user`                         |
| `disk_size` | | `20G` | Passed to `qemu-img resize`                          |
| `memory_mib` | | `4096` | RAM in MiB                                           |
| `vcpus` | | `4` | vCPU count                                           |
| `network_iface` | | `enp1s0` | Guest NIC name (consistent for virtio-net)           |
| `user` | | `debian` | System user                                          |

Secrets (`tailscale_auth_key`, passwords) are passed via `-e` at runtime — **never commit them**.

---

## Running Playbooks

```bash
# Install Galaxy collections first
ansible-galaxy collection install -r requirements.yml -p collections/

# Host setup (use --tags for user, ssh_hardening, libvirt, nftables, tailscale, promtail)
ansible-playbook playbooks/host-setup.yml -e tailscale_auth_key=<key>

# Deploy all VMs (run host-setup with --tags libvirt first)
ansible-playbook playbooks/vm-deploy.yml -e tailscale_auth_key=<key>

# Deploy one VM from `vms` by name
ansible-playbook playbooks/vm-deploy.yml -e tailscale_auth_key=<key> -e vm_name=<name>

# Destroy one VM (domain + per-VM artifacts)
ansible-playbook playbooks/vm-deploy.yml -e vm_state=absent -e vm_name=<name>

# Configure Caddy (and other VM config) on deployed VMs
ansible-playbook playbooks/vm-setup.yml --tags caddy

# Firewall only (alternative to host-setup --tags nftables)
ansible-playbook playbooks/firewall.yml --tags nftables
```

Use `-C` (check mode) and `-D` (diff) to preview changes before applying:

```bash
ansible-playbook playbooks/host-setup.yml -C -D
```

---

## Conventions & Guidelines for AI Agents

### Making Changes

- **Roles are self-contained.** Each role owns its `defaults/`, `tasks/`, `handlers/`, and `templates/`. Don't reach across roles.
- **Defaults over hardcoding.** Variables belong in `roles/<role>/defaults/main.yml`, not hardcoded in tasks.
- **Templates use Jinja2.** Template files live in `templates/` with a `.j2` extension.
- **Handlers for service restarts.** Use `notify:` + handlers rather than restarting services directly in tasks.
- **No secrets in repo.** Vault-encrypt or pass via `-e` at runtime.

### Adding a New Role

1. Create `roles/<role_name>/{tasks,defaults,handlers,templates}/`
2. Add a `tasks/main.yml` as the entry point
3. Add `defaults/main.yml` with sensible defaults and comments
4. Wire it into the appropriate playbook under `roles:`
5. Document new variables in `inventory/hosts.example.yml`

### Linting

```bash
# Install Galaxy collections first (ansible.posix required for host_setup nftables)
ansible-galaxy collection install -r requirements.yml -p collections/
# Requires ansible-lint
ansible-lint
```

Fix any `yaml[truthy]`, `name[missing]`, or `no-changed-when` warnings before opening a PR.

### PR Checklist

- [ ] `hosts.example.yml` updated if new variables were added
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
- The host firewall is in `host_setup` (tag `nftables`) — review `roles/host_setup/templates/nftables.conf.j2` before applying to any host.

---

## Reference Docs

- `docs/hetzner/INSTALL-OS-HETZNER.md` — OS install guide for Hetzner bare metal
- `docs/kvm-libvrt-vm-guide.md` — KVM/libvirt setup reference
- `docs/vm-setup-guide.md` — VM configuration walkthrough
