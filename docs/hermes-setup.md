# Hermes Agent Setup

To assist in setting up an environment for Hermes, this repo includes `playbooks/hermes-setup.yml` to install Hermes and a collection of services it depends upon, including:
- Github CLI
- Github MCP server
- Obsidian (headless sync)

Additionally, it installs `docker` and `node` which are sub-dependencies for these services.

# Full Setup

> Make sure to update your hosts.yml to include `agent_vms` with your target hosts.

## 1. Install Docker & Nodejs
```shell
ansible-playbook playbooks/hermes-setup.yml --tags install_docker,install_nodejs
```

## 2. Install Github CLI & MCP server

This will install and authenticate `gh`, and deploy the MCP server as a docker container.
```shell
GH_TOKEN=<TOKEN> ansible-playbook playbooks/hermes-setup.yml --tags gh_mcp,gh_cli
```

## 3. Install/Configure Obsidian

This will setup a remote vault on the VM, and configure syncing as systemd user service.

You will need an auth token and the vault id, which can retrieved on your local machine:
```shell
mkdir /tmp/ob-token-gen
HOME=/tmp/ob-token-gen npx obsidian-headless login
cat /tmp/ob-token-gen/.config/obsidian-headless/auth_token
HOME=/tmp/ob-token-gen npx obsidian-headless sync-list-remote
```
>[!NOTE]
> The vault ID or name can be used for OB_VAULT_ID. It will also be used to name the vault directory (`~/<OB_VAULT_ID>`). 

```shell
OB_TOKEN=<TOKEN> \
OB_VAULT_ID=<ID> \
OB_VAULT_PW=<PASSWORD> \
ansible-playbook playbooks/hermes-setup.yml --tags obsidian
```

## 4. Install Hermes

This simply runs the Hermes install script.

```shell
ansible-playbook playbooks/hermes-setup.yml --tags hermes
```
