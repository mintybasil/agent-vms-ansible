# OpenClaw VM Deployments

Hardened VM deployments for OpenClaw.

# Features
- Tailscale setup on host and in VMs
- Firewall configuration on host and VMs
- Caddy proxy for authenticated access to APIs in VM, via HTTPs
- Minimal OpenClaw config for quick start
- Promtail setup for log collection

# Usage

## Requirements
- Host with fresh Debian 13 install
- Tailscale account (+ auth key) with HTTPs enabled
- `inventory/hosts.yml` is set with the IP/hostname of your host machine

## Host Setup

### 1) Setup host user
> If you already have a hardened non-root user, you can skip this step.

1. Set `host_user_manage: true` in `inventory/hosts.yml`
2. Set `host_user` to desired username
3. Run playbook, temporarily overriding user as `root`:
   ```shell
   ansible-playbook playbooks/host-setup.yml --tags user -e ansible_user=root
   ```
4. SSH as `root` and set a strong password (`passwd <user>`)
5. SSH as the new user to confirm setup and verify `sudo ls` works

### 2) Setup Tailscale on host
> Visit [tailscale.com/admin/settings/keys](https://tailscale.com/admin/settings/keys) to create an auth key.

1. Install and configure Tailscale:
   ```shell
   ansible-playbook playbooks/host-setup.yml --tags tailscale -e tailscale_auth_key=<key> --ask-become-pass
   ```
2. Update `inventory/hosts.yml` (or SSH config) to use the host's Tailscale IP/hostname

### 3) SSH hardening (optional)
> Run after the non-root user is working. Ensure the firewall allows `host_ssh_port` before changing the port.

1. Set `host_ssh_port` (and optionally `ssh_permit_root_login`, `ssh_password_authentication`) in `inventory/hosts.yml`
2. If you have an external firewall, allow the new port
3. Run:
   ```shell
   ansible-playbook playbooks/host-setup.yml --tags ssh_hardening --ask-become-pass
   ```

### 4) Setup libvirt
1. Install and configure libvirt:
   ```shell
   ansible-playbook playbooks/host-setup.yml --tags libvirt --ask-become-pass
   ```

### 5) Deploy firewall
1. Review [default allowed ports](roles/host_setup/defaults/main.yml) (`vm_allowed_outbound_ports`)
2. Install and configure nftables:
   ```shell
   ansible-playbook playbooks/host-setup.yml --tags nftables --ask-become-pass
   ```
3. (Recommended) Review external firewall so only required ports are open

### Optional: Promtail
1. Set `promtail_enabled` (true/false)
2. Run promtail setup:
   ```shell
   ansible-playbook playbooks/host-setup.yml --tags promtail
   ```

## Deploying/Updating VMs

1. Define VM entries under `vms` in `inventory/hosts.yml`
2. Acquire a Tailscale auth key (**warning:** key is stored in VM; prefer one-time/ephemeral keys)
3. Deploy or update all VMs in `vms`:
   ```shell
   ansible-playbook playbooks/vm-deploy.yml -e tailscale_auth_key=<key> --ask-become-pass
   ```
   OR deploy or update one VM by name:
   ```shell
   ansible-playbook playbooks/vm-deploy.yml -e tailscale_auth_key=<key> -e vm_name=<vm-name> --ask-become-pass
   ```
5. Verify VM(s) are reachable via Tailscale SSH after boot
6. Ensure `inventory/hosts.yml` includes VM host entries you want to configure with VM-level playbooks

## Configuring VM (Caddy + OpenClaw)

### Caddy
1. Set `caddy_basicauth_user` and `caddy_basicauth_hash`
2. Deploy Caddy with Tailscale HTTPS:
   ```shell
   ansible-playbook playbooks/vm-setup.yml --tags caddy
   # or for one VM host
   ansible-playbook playbooks/vm-setup.yml --tags caddy --limit <host>
   ```
3. Confirm authentication works on `https://<vm_tailscale_hostname>`

### OpenClaw
> Installation is not automated yet because interactive setup is useful.

1. SSH into the VM and run the install script: [https://docs.openclaw.ai/install](https://docs.openclaw.ai/install)

## Destroying VMs

1. Destroy a VM (domain + disk + seed ISO + cloud-init files):
   ```shell
   ansible-playbook playbooks/vm-deploy.yml -e vm_state=absent -e vm_name=<vm-name> --ask-become-pass
   ```

# Additional Notes

## Tailscale ACL Setup

It is **strongly** recommended to ensure your Tailscale ACL configuration is setup to deny-by-default (no ACL rules => allow-by-default).

### Option 1: Personal Devices -> Host/VM

Create two tags - `personal` and `openclaw`. Assign `personal` to devices you use to access the host/VM. Assign `openclaw` to the host/VM (can be done when generating auth key).

Apply an ACL rule:
```json
{
    "src": ["tag:personal"],
    "dst": ["tag:zeroklaw"],
    "ip":  ["7777", "443"] // 7777 - non standard SSH port, 443 - Caddy HTTPS
}
```

### Option 2: All -> Host/VM

Create one tag (eg. `openclaw`), and assign it to the Host/VM (can be done when generating auth key).

```json
{
    "src": ["*"],
    "dst": ["tag:zeroklaw"],
    "ip":  ["7777", "443"] // 7777 - non standard SSH port, 443 - Caddy HTTPS
}
```