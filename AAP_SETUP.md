# AAP configuration checklist

## Project

- Name: `Proxmox K3s Automation`
- SCM type: Git
- SCM branch: `main`
- Update revision on launch: enabled

## Inventory 1: Proxmox

- Group: `proxmox`
- Host: the Proxmox DNS name
- Host variable: `ansible_user: ansible`
- Credential: Proxmox SSH Machine credential

## Inventory 2: K3s

- Group: `k3s_cluster`
- Child groups: `k3s_server`, `k3s_agents`
- Credential: Ubuntu SSH Machine credential
- Group variables:

```yaml
ansible_user: ubuntu
ansible_become: true
```

## Survey shared by the first two templates

| Prompt | Variable | Type | Required |
| --- | --- | --- | --- |
| Cloud-init SSH public key | `cloud_init_ssh_public_key` | Text | Yes |

## Job templates

### Create Proxmox K3s Base Template

- Playbook: `playbooks/create-proxmox-template.yml`
- Inventory: Proxmox
- Credential: Proxmox SSH
- Survey: SSH public key

### Clone K3s Nodes

- Playbook: `playbooks/clone-k3s-nodes.yml`
- Inventory: Proxmox
- Credential: Proxmox SSH
- Survey: SSH public key

### Install K3s Cluster

- Playbook: `playbooks/install-k3s.yml`
- Inventory: K3s
- Credential: Ubuntu SSH
- Privilege escalation: enabled

## Suggested workflow

Connect the job templates as success nodes:

```text
Clone K3s Nodes -> Install K3s Cluster
```

Keep template rebuilding as a separate manually approved workflow because it can delete and recreate the configured template VM ID.

