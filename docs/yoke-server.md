# Yoke Server Setup

## Pre-requisites

- Hermes agent deployed with API and inter-VM networking enabled (Yoke -> Hermes)
- Gtihub PAT

## Basic Usage

> [!IMPORTANT]
> Ensure you run `ansible-galaxy collection install -r requirements.yml -p ./collections` to fetch the required collections.

### 1. Install Docker

```shell
ansible-playbook playbooks/agents/yoke-server.yml --tags install_docker
```

### 2. Setup Webhooks

```shell
 GH_TOKEN=<token> WEBHOOK_SECRET=<secret> ansible-playbook playbooks/agents/yoke-server.yml --tags setup
```

### 3. Start Yoke Server
```shell
 GH_TOKEN=<token> WEBHOOK_SECRET=<secret> HERMES_API_KEY=<key> ansible-playbook playbooks/agents/yoke-server.yml --tags run
```

## Additional Commands

### Update Workflows
```shell
 GH_TOKEN=<token> ansible-playbook playbooks/agents/yoke-server.yml --tags update_workflows
```

### Stop Yoke Server
```shell
 ansible-playbook playbooks/agents/yoke-server.yml --tags stop
```

### Pull Latest Docker Image
```shell
 ansible-playbook playbooks/agents/yoke-server.yml --tags pull
```

### Remove Webhooks
```shell
 GH_TOKEN=<token> ansible-playbook playbooks/agents/yoke-server.yml --tags remove_webhooks
```

### List Webhooks
```shell
 GH_TOKEN=<token> ansible-playbook playbooks/agents/yoke-server.yml --tags list_webhooks
```