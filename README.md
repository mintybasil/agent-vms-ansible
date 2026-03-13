# OpenClaw VM Deployments

Hardened VM deployments for OpenClaw.

## Features
- Tailscale setup on host and inside VMs
- Promtail setup for log collection
- Firewall configuration on host and VMs
- Caddy proxy for authenticated access to APIs
- Minimal OpenClaw config for quick start

## How To Use

### Pre-Requisites
- Host with fresh Debian 13 install
- Tailscale account (+ auth key)

### Setup
1. Update inventory/hosts.yml with the IP of your host
2. Set `promtail_enabled` (true/false)
3. Run host setup (Tailscale + Promtail)
```shell
ansible-playbook playbooks/host-setup.yml -e tailscale_auth_key=<key>
```
4. Confirm access via Tailscale, update hosts.yml with Tailscale hostname/IP.
5. Deploy and configure VM. 
> Avoid re-usable tailscale auth keys as this will introduce it to the VM environment and could later be extracted by an adversary.
```shell
ansible-playbook playbooks/host-deploy.yml -e tailscale_auth_key=<key> 
```
