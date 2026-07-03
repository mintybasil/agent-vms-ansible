# Yoke Agents Setup

## Setup

```shell
 PROFILE= \
 GH_TOKEN= \
 MODEL_API_KEY= \
 MODEL_API_FB_KEY= \
 EXA_API_KEY= \
 ansible-playbook playbooks/agents/yoke-agent-setup.yml --tags setup
```
> [!NOTE]
> `MODEL_FB_API_KEY is optional.`

## Additional Steps

### Setup API

```shell
 API_PORT= \
 API_HOST= \
 API_SECRET= \
 ansible-playbook playbooks/agents/yoke-agent-setup.yml --tags api
```

### Setup Dashboard

```shell
 DASH_USER= \
 DASH_PW= \
 ansible-playbook playbooks/agents/yoke-agent-setup.yml --tags dashboard
```

### Create Profile
```shell
 PROFILE= \
 ansible-playbook playbooks/agents/yoke-agent-setup.yml --tags profile
```

### Update Config
```shell
 PROFILE= \
 ansible-playbook playbooks/agents/yoke-agent-setup.yml --tags config
```

### Update Skills
```shell
 PROFILE= \
 ansible-playbook playbooks/agents/yoke-agent-setup.yml --tags profile
```

### Deploy SOUL.md
```shell
 PROFILE= \
 ansible-playbook playbooks/agents/yoke-agent-setup.yml --tags soul
```

### Update Model Config
```shell
 PROFILE= \
 MODEL_API_KEY= \
 MODEL_FB_API_KEY= \
 ansible-playbook playbooks/agents/yoke-agent-setup.yml --tags model
```

### Configure Exa Search
```shell
 PROFILE= \
 EXA_API_KEY= \
 ansible-playbook playbooks/agents/yoke-agent-setup.yml --tags exa_search
```

### Configure GH MCP
```shell
 PROFILE= \
 GH_TOKEN= \
 ansible-playbook playbooks/agents/yoke-agent-setup.yml --tags gh_mcp
```