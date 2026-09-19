# AAP Proxmox K3s Automation

This repository contains three AAP-ready playbooks:

1. `create-proxmox-template.yml` downloads an Ubuntu 24.04 cloud image on Proxmox and converts it into a reusable cloud-init template.
2. `clone-k3s-nodes.yml` clones the template into K3s virtual machines.
3. `install-k3s.yml` installs one K3s server and joins agent nodes.

K3s is deliberately installed after cloning. Do not bake an initialized K3s installation into a VM template because each node must have a unique identity, hostname, certificates and cluster state.

## Requirements

- Ansible Automation Platform 2.7
- Proxmox VE reachable from the AAP execution environment over SSH
- A Proxmox account able to run `qm`, write to the image directory and manage VMs
- DNS or IP connectivity from AAP to the newly created K3s VMs
- An SSH public key for cloud-init
- A DHCP reservation or static addresses for the K3s VMs

The playbooks use `ansible.builtin` modules only, so the default supported AAP execution environment should be sufficient.

## Repository layout

```text
.
├── ansible.cfg
├── inventories/
│   ├── proxmox/hosts.yml
│   └── k3s/hosts.yml
├── playbooks/
│   ├── create-proxmox-template.yml
│   ├── clone-k3s-nodes.yml
│   └── install-k3s.yml
└── vars/
    ├── template.yml
    └── k3s-nodes.yml
```

## 1. Configure Proxmox inventory

Edit `inventories/proxmox/hosts.yml` or create the equivalent inventory and host in AAP:

```yaml
proxmox:
  hosts:
    pve01.example.com:
      ansible_user: ansible
```

Create an AAP Machine credential for the Proxmox SSH account. The account needs passwordless sudo for the required `qm`, image-download and file-management commands. For an initial lab, using `root` is simpler, but a restricted automation account is recommended afterward.

## 2. Configure the template

Review `vars/template.yml`. The main values are:

```yaml
proxmox_storage: local-lvm
proxmox_bridge: vmbr0
template_vmid: 9000
template_name: ubuntu-2404-k3s-base
```

Provide your public SSH key at launch instead of committing it:

```yaml
cloud_init_ssh_public_key: "ssh-ed25519 AAAA..."
```

In AAP, make `cloud_init_ssh_public_key` a Survey question with the answer type **Text**.

## 3. AAP template: create base template

Create an AAP Job Template with:

```text
Name: Create Proxmox K3s Base Template
Inventory: Proxmox
Project: this Git repository
Playbook: playbooks/create-proxmox-template.yml
Credential: Proxmox SSH Machine credential
Extra variables file: loaded automatically by the playbook
```

Launch it. Re-running it is safe: if VM ID `9000` is already a Proxmox template, it performs no rebuild.

To intentionally rebuild the template, launch with:

```yaml
rebuild_template: true
```

This deletes and recreates only the configured `template_vmid`, so verify that value carefully.

## 4. Configure and clone K3s VMs

Review `vars/k3s-nodes.yml`. The sample creates:

- `k3s-server-01` at `192.168.10.101`
- `k3s-agent-01` at `192.168.10.111`
- `k3s-agent-02` at `192.168.10.112`

Create another AAP Job Template:

```text
Name: Clone K3s Nodes
Inventory: Proxmox
Playbook: playbooks/clone-k3s-nodes.yml
Credential: Proxmox SSH Machine credential
Survey: cloud_init_ssh_public_key
```

The playbook clones, configures and starts each VM. Existing VM IDs are skipped unless `rebuild_nodes: true` is explicitly supplied.

## 5. Add the new VMs to AAP

Create an AAP inventory matching `inventories/k3s/hosts.yml`. Replace the sample addresses with your own.

Create a Machine credential for the cloud-init user (`ubuntu` by default) using the private key matching `cloud_init_ssh_public_key`.

## 6. Install K3s

Create the final AAP Job Template:

```text
Name: Install K3s Cluster
Inventory: K3s
Playbook: playbooks/install-k3s.yml
Credential: K3s Node SSH Machine credential
Privilege escalation: enabled
```

Launch the job. It installs the server first, reads the protected join token using `no_log`, and then joins all agents.

## Workflow

Create an AAP Workflow Job Template in this order:

```text
Create base template -> Clone K3s nodes -> Install K3s cluster
```

The first job is normally run only when refreshing the base image. Day-to-day cluster creation starts with cloning.

## Security notes

- Never commit private keys, API tokens or K3s join tokens.
- Pin `k3s_version` before using this outside a lab.
- Replace Proxmox's self-signed certificates if you later move to API-based modules.
- Restrict the Proxmox automation account to the target VM ID range and storage.
- Put the K3s API behind a firewall and permit TCP 6443 only from trusted networks.
- Use separate AAP credentials for Proxmox administration and K3s node SSH.

