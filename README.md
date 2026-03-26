# Deploying FortiCNAPP Linux Agent via Ansible on Azure

**Document Owner:*LinuxExploit*
**Last Updated:** March 26, 2026
**Scope:** End-to-end guide for deploying the Lacework FortiCNAPP Linux agent to Ubuntu VMs in Azure using Ansible with static inventory
**Product:** Lacework FortiCNAPP (Fortinet)
**Status:** ✅ Tested successfully in Azure environment

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites](#2-prerequisites)
3. [Set Up the Ansible Control Node](#3-set-up-the-ansible-control-node)
4. [Create the Static Inventory](#4-create-the-static-inventory)
5. [Prepare the FortiCNAPP Agent Config](#5-prepare-the-forticnapp-agent-config)
6. [The Playbook — FortiCNAPP Agent Deployment](#6-the-playbook--forticnapp-agent-deployment)
7. [Run the Playbook](#7-run-the-playbook)
8. [Verify the Deployment](#8-verify-the-deployment)
9. [Operational Playbooks](#9-operational-playbooks)
10. [Troubleshooting](#10-troubleshooting)
11. [References](#11-references)

---

## 1. Architecture Overview

```
┌──────────────────────┐
│  Ansible Control     │
│  Node                │
│  (your workstation   │
│   or Azure VM)       │
│                      │
│  - ansible           │
│  - config.json       │
│  - playbook.yml      │
│  - hosts inventory   │
└──────────┬───────────┘
           │ SSH (port 22)
           │
     ┌─────┴─────────────────────────────────┐
     │         Azure Resource Group           │
     │                                        │
     │  ┌──────────┐  ┌──────────┐  ┌─────┐  │
     │  │ Ubuntu   │  │ Ubuntu   │  │ ... │  │
     │  │ VM-01    │  │ VM-02    │  │     │  │
     │  │          │  │          │  │     │  │
     │  │ Agent ✓  │  │ Agent ✓  │  │  ✓  │  │
     │  └──────────┘  └──────────┘  └─────┘  │
     │                                        │
     └────────────────────┬───────────────────┘
                          │ HTTPS (port 443)
                          ▼
                ┌──────────────────┐
                │  FortiCNAPP      │
                │  Cloud Platform  │
                │  (Lacework)      │
                └──────────────────┘
```

Ansible connects to Azure VMs over SSH, installs the agent package via APT, deploys the config.json with your access token, and starts the datacollector service. The agent then communicates with the FortiCNAPP platform over HTTPS.

---

## 2. Prerequisites

### 2.1 Azure Side

- [ ] Ubuntu VMs deployed in a known resource group
- [ ] VMs have SSH (port 22) accessible from the Ansible control node
- [ ] NSG allows outbound HTTPS (port 443) from the VMs to Lacework URLs
- [ ] SSH key pair — public key on the VMs, private key on the control node
- [ ] You know the private IPs (or public IPs) of your target VMs

### 2.2 FortiCNAPP Side

- [ ] FortiCNAPP Console access
- [ ] A Linux agent access token (Settings → Configuration → Agent Tokens → Copy)
- [ ] Your Lacework server URL (default US: `https://api.lacework.net`)

### 2.3 Ansible Control Node

- [ ] A Linux or macOS machine (can be an Azure VM or your local workstation)
- [ ] Python 3.8+
- [ ] Ansible 2.10+
- [ ] SSH connectivity to target VMs

---

## 3. Set Up the Ansible Control Node

### 3.1 Install Ansible

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install -y python3 python3-pip
pip3 install ansible

# Verify
ansible --version
```

### 3.2 Create the Project Directory

```bash
mkdir -p ~/forticnapp-ansible/{inventory,playbooks,files}
cd ~/forticnapp-ansible
```

Your directory structure:

```
forticnapp-ansible/
├── inventory/
│   └── hosts                        # Static inventory
├── playbooks/
│   ├── deploy-lacework-agent.yml    # Install playbook
│   ├── update-lacework-agent.yml    # Update playbook
│   └── remove-lacework-agent.yml   # Uninstall playbook
├── files/
│   └── config.json                  # FortiCNAPP agent config
└── ansible.cfg                      # Ansible configuration
```

### 3.3 Create ansible.cfg

Create `~/forticnapp-ansible/ansible.cfg`:

```ini
[defaults]
inventory = ./inventory/hosts
remote_user = azureuser
private_key_file = ~/.ssh/id_rsa
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = True
become_method = sudo
become_user = root
```

---

## 4. Create the Static Inventory

Create `inventory/hosts`:

```ini
[lacework_servers]
vm-linux-01  ansible_host=10.0.0.4
vm-linux-02  ansible_host=10.0.0.5
vm-linux-03  ansible_host=10.0.0.6

[lacework_servers:vars]
ansible_user=azureuser
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

Replace the VM names and IPs with your actual values.

**Get your VM private IPs from Azure CLI:**

```bash
az vm list-ip-addresses \
  --resource-group <Your_Resource_Group> \
  --output table
```

**Test connectivity:**

```bash
ansible all -m ping
```

Expected output:

```
vm-linux-01 | SUCCESS => { "ping": "pong" }
vm-linux-02 | SUCCESS => { "ping": "pong" }
vm-linux-03 | SUCCESS => { "ping": "pong" }
```

---

## 5. Prepare the FortiCNAPP Agent Config

### 5.1 Get Your Access Token

1. Log in to the **FortiCNAPP Console**
2. Go to **Settings → Configuration → Agent Tokens**
3. Click **+ Add New** → name it (e.g. `linux-azure-prod`) → select **Linux** → **Save**
4. Click the **...** icon → **Copy** the token

### 5.2 Create config.json

Create `files/config.json`:

```json
{
  "tokens": {
    "accesstoken": "<Your_Access_Token>"
  },
  "serverurl": "https://api.lacework.net"
}
```

Replace `<Your_Access_Token>` with the token you copied. Update `serverurl` if you are outside the US region:

| Region | Server URL |
|---|---|
| US (default) | `https://api.lacework.net` |
| EU | `https://api.fra.lacework.net` |
| ANZ / APRODUS2 | Check your Console URL or contact support |

---

## 6. The Playbook — FortiCNAPP Agent Deployment

### 6.1 Install Playbook

Create `playbooks/deploy-lacework-agent.yml`:

```yaml
---
- name: Deploy FortiCNAPP Linux Agent
  hosts: lacework_servers
  become: yes

  tasks:
    - name: Install prerequisites
      apt:
        name:
          - gpg
          - ca-certificates
          - curl
          - lsb-release
        state: present
        update_cache: yes

    - name: Add Lacework APT signing key
      shell: |
        curl -s 'https://packages.lacework.net/keys/RPM-GPG-KEY-lacework' | gpg --dearmor > /etc/apt/trusted.gpg.d/lacework-agent.gpg
      args:
        creates: /etc/apt/trusted.gpg.d/lacework-agent.gpg

    - name: Get distribution name
      shell: lsb_release -i | cut -f2 | tr '[:upper:]' '[:lower:]'
      register: lsb_distro
      changed_when: false

    - name: Get release codename
      shell: lsb_release -c | cut -f2
      register: lsb_rel
      changed_when: false

    - name: Remove old repo file if exists
      file:
        path: /etc/apt/sources.list.d/lacework.list
        state: absent

    - name: Add Lacework latest repository
      shell: echo "deb [arch=amd64] https://packages.lacework.net/latest/DEB/{{ lsb_distro.stdout }} {{ lsb_rel.stdout }} main" >> /etc/apt/sources.list.d/lacework.list

    - name: Add Lacework established repository
      shell: echo "deb [arch=amd64] https://packages.lacework.net/established/DEB/{{ lsb_distro.stdout }} {{ lsb_rel.stdout }} main" >> /etc/apt/sources.list.d/lacework.list

    - name: Add Lacework archived repository
      shell: echo "deb [arch=amd64] https://packages.lacework.net/archived/DEB/{{ lsb_distro.stdout }} {{ lsb_rel.stdout }} main" >> /etc/apt/sources.list.d/lacework.list

    - name: Pin Lacework repository
      copy:
        dest: /etc/apt/preferences.d/lacework
        content: |
          Package: lacework*
          Pin: origin "packages.lacework.net"
          Pin-Priority: 999
        owner: root
        group: root
        mode: 0644

    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install Lacework datacollector
      apt:
        name: lacework
        state: latest

    - name: Wait until config directory is created
      wait_for:
        path: /var/lib/lacework/config/

    - name: Copy config.json
      copy:
        src: ../files/config.json
        dest: /var/lib/lacework/config/config.json
        owner: root
        group: root
        mode: 0644
      notify: Restart Lacework agent

    - name: Ensure datacollector service is enabled and started
      systemd:
        name: datacollector
        enabled: yes
        state: started

    - name: Verify agent is running
      command: systemctl is-active datacollector
      register: agent_status
      changed_when: false

    - name: Show agent status
      debug:
        msg: "Agent status on {{ inventory_hostname }}: {{ agent_status.stdout }}"

  handlers:
    - name: Restart Lacework agent
      systemd:
        name: datacollector
        state: restarted
```

### 6.2 What the Playbook Does (Step by Step)

| Step | Task | What Happens |
|---|---|---|
| 1 | Install prerequisites | Ensures gpg, ca-certificates, curl, lsb-release are present |
| 2 | Add APT signing key | Imports the Lacework GPG key from packages.lacework.net |
| 3 | Get distribution name | Detects OS (e.g. `ubuntu`) for repo URL |
| 4 | Get release codename | Detects codename (e.g. `jammy`) for repo URL |
| 5 | Remove old repo file | Cleans up any previous repo file to avoid duplicates |
| 6–8 | Add three repositories | Creates `/etc/apt/sources.list.d/lacework.list` with latest, established, and archived repos |
| 9 | Pin repository | Creates `/etc/apt/preferences.d/lacework` to prioritize Lacework packages |
| 10 | Update apt cache | Refreshes package lists after repo setup |
| 11 | Install datacollector | Installs the latest Lacework agent package via apt |
| 12 | Wait for config dir | Waits for `/var/lib/lacework/config/` to be created by the installer |
| 13 | Copy config.json | Deploys your access token and server URL to the agent |
| 14 | Enable and start service | Ensures the datacollector service starts on boot and is running |
| 15 | Verify | Confirms the service is active and prints the status |

---

## 7. Run the Playbook

### 7.1 Dry Run (Check Mode)

Always do a dry run first to see what will change:

```bash
cd ~/forticnapp-ansible
ansible-playbook playbooks/deploy-lacework-agent.yml --check
```

### 7.2 Full Run

```bash
ansible-playbook playbooks/deploy-lacework-agent.yml
```

Expected output (per host):

```
TASK [Install prerequisites] *************************************** ok
TASK [Add Lacework APT signing key] ******************************** ok
TASK [Add Lacework repository] ************************************* changed
TASK [Pin Lacework repository] ************************************* changed
TASK [Install Lacework datacollector] ****************************** changed
TASK [Wait until config directory is created] *********************** ok
TASK [Copy config.json] ******************************************** changed
TASK [Ensure datacollector service is enabled and started] ********** ok
TASK [Verify agent is running] ************************************* ok
TASK [Show agent status] ******************************************* ok
  msg: "Agent status on vm-linux-01: active"

PLAY RECAP *********************************************************
vm-linux-01  : ok=10  changed=4  unreachable=0  failed=0
vm-linux-02  : ok=10  changed=4  unreachable=0  failed=0
vm-linux-03  : ok=10  changed=4  unreachable=0  failed=0
```

> **✅ Tested successfully in Azure environment — March 26, 2026.** Playbook ran to completion on Ubuntu 22.04 VMs deployed via Terraform. GPG key imported from `packages.lacework.net`, all three repository channels (latest, established, archived) configured, agent installed and datacollector service started successfully.

### 7.3 Target Specific Hosts

```bash
# Single host
ansible-playbook playbooks/deploy-lacework-agent.yml --limit vm-linux-01

# Multiple hosts
ansible-playbook playbooks/deploy-lacework-agent.yml --limit "vm-linux-01,vm-linux-02"
```

### 7.4 Verbose Output (for Debugging)

```bash
ansible-playbook playbooks/deploy-lacework-agent.yml -vvv
```

---

## 8. Verify the Deployment

### 8.1 From Ansible (Ad-hoc Commands)

```bash
# Check service status on all hosts
ansible lacework_servers -m command -a "systemctl status datacollector"

# Check agent version
ansible lacework_servers -m command -a "/var/lib/lacework/datacollector --version"

# Verify config.json is in place
ansible lacework_servers -m command -a "cat /var/lib/lacework/config/config.json"

# Check agent connectivity logs
ansible lacework_servers -m command -a "journalctl -u datacollector --no-pager -n 10"
```

### 8.2 From FortiCNAPP Console

1. Log in to the FortiCNAPP Console
2. Navigate to **Agents**
3. VMs should appear within **10–15 minutes** of successful deployment
4. Sort by **Agent OS** to find your Linux hosts

---

## 9. Operational Playbooks

### 9.1 Update Agent

Create `playbooks/update-lacework-agent.yml`:

```yaml
---
- name: Update FortiCNAPP agent to latest
  hosts: lacework_servers
  become: yes

  tasks:
    - name: Update apt cache and upgrade Lacework agent
      apt:
        name: lacework
        state: latest
        update_cache: yes

    - name: Restart datacollector
      systemd:
        name: datacollector
        state: restarted

    - name: Verify agent is running
      command: systemctl is-active datacollector
      register: agent_status
      changed_when: false

    - name: Show status
      debug:
        msg: "{{ inventory_hostname }}: {{ agent_status.stdout }}"
```

Run:

```bash
ansible-playbook playbooks/update-lacework-agent.yml
```

### 9.2 Uninstall Agent

Create `playbooks/remove-lacework-agent.yml`:

```yaml
---
- name: Remove FortiCNAPP agent
  hosts: lacework_servers
  become: yes

  tasks:
    - name: Stop datacollector service
      systemd:
        name: datacollector
        state: stopped
        enabled: no
      ignore_errors: yes

    - name: Remove Lacework package
      apt:
        name: lacework
        state: absent
        purge: yes

    - name: Remove config directory
      file:
        path: /var/lib/lacework
        state: absent

    - name: Remove APT repository
      apt_repository:
        repo: "deb [arch=amd64] https://packages.lacework.net/latest/DEB/{{ ansible_distribution | lower }} {{ ansible_distribution_release }} main"
        filename: lacework
        state: absent

    - name: Remove APT pin file
      file:
        path: /etc/apt/preferences.d/lacework
        state: absent
```

Run:

```bash
ansible-playbook playbooks/remove-lacework-agent.yml
```

### 9.3 Quick Fleet Status Check

No playbook needed — run ad-hoc:

```bash
ansible lacework_servers -m command -a "systemctl is-active datacollector"
```

---

## 10. Troubleshooting

### 10.1 Common Issues

| Issue | Cause | Resolution |
|---|---|---|
| `ping` fails — unreachable | SSH blocked or wrong IP | Verify NSG allows port 22, check `ansible_host` IP in inventory |
| `Permission denied (publickey)` | Wrong username in inventory | Run `az vm show --query "osProfile.adminUsername"` to get the correct username — update `ansible_user` in inventory |
| `apt_key` task fails with keyserver | Ubuntu keyserver unreachable from the VM | Use the alternate method: pull GPG key from `packages.lacework.net/keys/RPM-GPG-KEY-lacework` instead |
| `No package matching 'lacework'` with `apt_repository` module | Module may format the repo URL differently than expected | Use shell commands matching the official FortiCNAPP docs to create `/etc/apt/sources.list.d/lacework.list` |
| `No package matching 'lacework'` — only `latest` repo added | Package may only exist in `established` or `archived` repos | Add all three repos: latest, established, and archived — as per official docs |
| Playbook stops partway — not all tasks run | Playbook file was truncated during copy/paste | Verify with `wc -l` — full playbook is ~88 lines. Use base64 encoding to transfer if paste is unreliable |
| No inventory parsed warning | Running ansible from wrong directory | Run from the project root (`~/forticnapp-ansible/`) where `ansible.cfg` is located |
| Agent not appearing in Console | Wrong token or server URL | Check `files/config.json`, run `journalctl -u datacollector` on the VM |
| Permission denied on config.json | File not owned by root | Re-run the playbook — it sets `owner: root` |
| Package install fails | APT cache stale | Playbook includes a separate `update_cache` task after repo setup |
| Service won't start | config.json missing or malformed | SSH to the VM and check `/var/lib/lacework/config/config.json` |
| Host key verification failed | First SSH to a new VM | `ansible.cfg` has `host_key_checking = False` — verify it's set |
| Package install fails | APT cache stale | Playbook includes `update_cache: yes` — run again |
| Service won't start | config.json missing or malformed | SSH to the VM and check `/var/lib/lacework/config/config.json` |

### 10.2 Debug Commands

```bash
# Verbose playbook run
ansible-playbook playbooks/deploy-lacework-agent.yml -vvv

# Manual SSH test
ssh -i ~/.ssh/id_rsa azureuser@10.0.0.4

# Check what Ansible sees for a host
ansible vm-linux-01 -m setup | head -50

# Agent logs on a specific VM
ansible vm-linux-01 -m command -a "journalctl -u datacollector --no-pager -n 30"

# Check if agent binary exists
ansible lacework_servers -m stat -a "path=/var/lib/lacework/datacollector"
```

---

## 11. References

### FortiCNAPP Official Documentation

| Topic | URL |
|---|---|
| Ansible Installation (official playbook) | https://docs.fortinet.com/document/forticnapp/latest/administration-guide/206223/installing-with-ansible |
| APT Package Repositories | https://docs.fortinet.com/document/forticnapp/25.4.0/administration-guide/78302/install-linux-agent-from-package-repositories |
| Agent Configuration File | https://docs.fortinet.com/document/forticnapp/25.4.0/administration-guide/628180/configure-linux-agent-using-agent-configuration-file |
| Agent Access Tokens | https://docs.fortinet.com/document/forticnapp/latest/administration-guide/030085/download-linux-agent-installer |
| Connectivity Requirements | https://docs.fortinet.com/document/forticnapp/latest/administration-guide/059862/required-connectivity-proxies-and-certificates-for-agents |
| Linux Agent FAQs | https://docs.fortinet.com/document/forticnapp/latest/administration-guide/792160/linux-agent-faqs |

### Ansible Documentation

| Topic | URL |
|---|---|
| Ansible Installation Guide | https://docs.ansible.com/ansible/latest/installation_guide/ |
| apt_key Module | https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_key_module.html |
| apt_repository Module | https://docs.ansible.com/ansible/latest/collections/ansible/builtin/apt_repository_module.html |

---

**Security Note:** This guide stores the access token in plaintext in `files/config.json` for simplicity. For production deployments, consider using Ansible Vault to encrypt the token or retrieving it from Azure Key Vault at runtime.

---

*Document generated from FortiCNAPP official documentation. The playbook structure is based on the official FortiCNAPP Ansible example. Verify release-specific details against the current version of the Admin Guide.*
