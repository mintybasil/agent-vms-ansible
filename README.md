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

## Step 1 - Setup host user
> If you already have a hardened non-root user, you can skip this step.

1. Set `host_user_manage: true` in `inventory/group_vars/all.yml`
2. Set `host_user` to desired username
3. Run playbook, temporarily overriding user as `root`:
    ```shell
    ansible-playbook playbooks/host-setup.yml --tags user -e ansible_user=root
    ```
4. SSH as `root` and set a strong password (`passwd <user>`),
5. SSH as the new user to confirm the setup, verify `sudo ls` works

## Step 2 - Setup Tailscale on Host
> Visit [tailscale.com/admin/settings/keys](https://tailscale.com/admin/settings/keys) to create an Auth Key.
1. Install and configure Tailscale
    ```shell
    ansible-playbook playbooks/host-setup.yml --tags tailscale -e tailscale_auth_key=<key> --ask-become-pass
    ```
2. Update `hosts.yml` (or your SSH config) to use the Tailscale IP/Hostname of the host machine.

## Step 3 - SSH hardening (optional)
> Run after the non-root user is working. Ensure the firewall allows `host_ssh_port` before changing the port.
1. Set `host_ssh_port` (and optionally `ssh_permit_root_login`, `ssh_password_authentication`) in `inventory/group_vars/all.yml`
2. If you have an external firewall, make sure to allow the new port.
2. Run:
   ```shell
   ansible-playbook playbooks/host-setup.yml --tags ssh_hardening --ask-become-pass
   ```
## Step 4 - Setup libvirt
1. Install and configure libvirt:
   ```shell
   ansible-playbook playbooks/host-setup.yml --tags libvirt --ask-become-pass
   ```

## Step 5 - Deploy Firewall
1. Review [default allowed ports](roles/host_setup/defaults/main.yml) (`vm_allowed_outbound_ports`).
2. Install and configure nftables:
   ```shell
   ansible-playbook playbooks/host-setup.yml --tags nftables --ask-become-pass
   ```
3. (Recommended) Review external firewall to ensure minimal ports are open.

## Step 6 - Deploy VM
1. Define a new VM config in `inventory/group_vars/all.yml`
2. Acquire another Tailscale auth key (**WARNING: key will remain in VM and could become compromised. Avoid reusable keys, if possible.**)
3. Configure and deploy VM
    ```shell
   ansible-playbook playbooks/vm-deploy.yml -e tailscale_auth_key=<key> --ask-become-pass
    ```
4. Verify you can SSH into VM via Tailscale. You may need to wait a few minutes for the VM to fully boot.
5. Ensure hosts.yml includes the VM

## Step 7 - Caddy
1. Set `caddy_basicauth_user` and `caddy_basicauth_hash`
2. Deploy Caddy with Tailscale HTTPS:
```shell
   ansible-playbook playbooks/vm-setup.yml --tags caddy
   # OR for a specific VM host
   ansible-playbook playbooks/vm-setup.yml --tags caddy --limit <host>
```
3. Confirm authentication works on `https://<vm_tailscale_hostname>`

## Step 8 - Install OpenClaw
> Note: Installation is not automated as the OpenClaw interactive setup is quite useful
1. SSH into the VM and run the install script (https://docs.openclaw.ai/install)

## Optional - Promtail
1. Set `promtail_enabled` (true/false)
2. Run promtail setup
    ```shell
    ansible-playbook playbooks/host-setup.yml --tags promtail
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