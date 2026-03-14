# OpenClaw VM Deployments

Hardened VM deployments for OpenClaw.

# Features
- Tailscale setup on host and inside VMs
- Promtail setup for log collection
- Firewall configuration on host and VMs
- Caddy proxy for authenticated access to APIs
- Minimal OpenClaw config for quick start

# How To Use

## Requirements
- Host with fresh Debian 13 install
- Tailscale account (+ auth key)
- `inventory/hosts.yml` is set with the IP/hostname of your host machine

## Step 1 - Setup host user
> If you already have a hardened non-root user, you can skip this step.

1. Set `host_user_manage: true` in `inventory/group_vars/all.yml`
2. (Optional) Change `host_user` to desired username (default: `admin`)
3. (Recommended) Set `host_user_password_hash` to apply user password
   - `mkpasswd -m sha-512` (Debian, Ubuntu)
   - `python3 -c "import crypt; print(crypt.crypt('YOUR_PASSWORD', crypt.METHOD_SHA512))"` (Python)
4. Run playbook, temporarily overriding user as `root`:
    ```shell
    ansible-playbook playbooks/host-setup.yml --tags user -e ansible_user=root
    ```
5. SSH into the host and set a strong password,

## Step 2 - Setup Tailscale on Host
> Visit [tailscale.com/admin/settings/keys](https://tailscale.com/admin/settings/keys) to create an Auth Key.
1. Install and configure Tailscale
    ```shell
    ansible-playbook playbooks/host-setup.yml --tags tailscale -e tailscale_auth_key=<key>
    ```
2. Update `hosts.yml` to use the Tailscale IP/Hostname of the host machine

##  Step 3 - Deploy VM
1. Define VM config in `inventory/group_vars/all.yml`
2. Configure and deploy VM
    ```shell
   ansible-playbook playbooks/host-deploy.yml 
    ```

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