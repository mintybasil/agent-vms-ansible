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
| `vm-setup.yml` | `openclaw_vm` | Configure Caddy inside the VM |
| `firewall-setup.yml` | `openclaw_host` | Apply nftables firewall rules to host |

### Roles

| Role | Purpose |
|---|---|
| `host_tailscale` | Install and auth Tailscale on host |
| `host_promtail` | Deploy Promtail as a systemd service |
| `host_nftables` | Configure nftables firewall on host |
| `host_vm_deploy` | Create KVM VM using libvirt + cloud-init |
| `vm_setup` | VM-level network/interface config |
| `caddy` | Install Caddy + generate Caddyfile from template |

---

## Inventory

- `inventory/hosts.yml` — **gitignored**, contains real IPs/hostnames. Copy from `hosts.example.yml`.
- `inventory/group_vars/all.example.yml` — copy to `all.yml`, fill in required vars.

Two host groups:
- `openclaw_host` — the bare metal Debian 13 machine running KVM
- `openclaw_vm` — the VM running OpenClaw

---

## Key Variables

| Variable | Where | Notes |
|---|---|---|
| `host_user` | group_vars/all.yml | SSH user on host |
| `host_ssh_port` | group_vars/all.yml | Default 22; change it |
| `vm_vm_user` | group_vars/all.yml | Default `debian` |
| `vm_ssh_public_key` | group_vars/all.yml | Your SSH public key for VM |
| `vm_vm_name` | group_vars/all.yml | VM name (used in libvirt) |
| `vm_ip` | group_vars/all.yml | Local IP of VM on host |
| `caddy_basicauth_user` | group_vars/all.yml | Basic auth username for Caddy |
| `caddy_basicauth_hash` | group_vars/all.yml | Bcrypt hash via `caddy hash-password` |
| `caddy_tailnet_domain` | group_vars/all.yml | Tailscale MagicDNS domain (`*.ts.net`) |
| `promtail_enabled` | group_vars/all.yml | Set `true` to enable log shipping |

Secrets (auth keys, passwords) are passed via `-e` at runtime — **never commit them**.

---

## Running Playbooks

```bash
# Install Galaxy collections first
ansible-galaxy collection install -r requirements.yml -p collections/

# Host setup (Tailscale + optional Promtail)
ansible-playbook playbooks/host-setup.yml -e tailscale_auth_key=<key>

# Deploy VM
ansible-playbook playbooks/host-deploy.yml -e tailscale_auth_key=<key>

# VM app setup (Caddy)
ansible-playbook playbooks/vm-setup.yml

# Firewall
ansible-playbook playbooks/firewall-setup.yml
```

Use `-C` (check mode) and `-D` (diff) to preview changes before applying:

```bash
ansible-playbook playbooks/firewall-setup.yml -C -D
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
