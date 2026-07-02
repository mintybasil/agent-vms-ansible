# Yoke Agents Setup

## Setup

```shell
 PROFILE= \
 API_PORT= \
 API_HOST= \
 API_SECRET= \
 GH_TOKEN= \
 MODEL_API_KEY= \
 MODEL_API_FB_KEY= \
 EXA_API_KEY= \
 ansible-playbook playbooks/agents/yoke-agent-setup.yml
```

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