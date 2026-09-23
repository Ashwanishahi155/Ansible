<<<<<<< HEAD
# Ansible
=======
# Ansible Setup & Nginx Automation Lab

## 📌 Project Overview

This lab demonstrates a practical Ansible setup using:

- 1 Ansible Control Node
- Multiple Managed Nodes
- SSH key-based authentication
- Ansible Inventory
- Ansible Playbook  
- Nginx installation and configuration


The objective was to configure the control node, establish SSH key-based connectivity with managed nodes, generate inventory entries, test Ansible connectivity, and install Nginx on managed nodes.

---

## 1. Ansible Installation

Check Ansible:

```bash
ansible --version
```

If Ansible is not installed:

```bash
sudo apt update
sudo apt install ansible
```

Verify:

```bash
ansible --version
```

Example:

```text
ansible [core 2.16.3]
config file = None
executable location = /usr/bin/ansible
python version = 3.12.3
jinja version = 3.1.2
libyaml = True
```

---

## 2. Generate SSH Key on Control Node

Check SSH directory:

```bash
ls -ls ~/.ssh/
```

Generate an ED25519 key:

```bash
ssh-keygen -t ed25519
```

Default files:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

Verify:

```bash
ls -ls ~/.ssh/
```

### Important

```text
id_ed25519
    ↓
Private Key — keep secure on Control Node

id_ed25519.pub
    ↓
Public Key — copy to Managed Nodes
```

---

## 3. Create Ansible Working Directory

```bash
mkdir -p ansible-test
cd ansible-test
```

The working directory contains:

```text
ansible-test/
├── vm-list.txt
├── inventory.ini
└── playbook.yaml
```

---

## 4. Get Azure VM IP Addresses and Usernames

From the Windows machine:

```powershell
az vm list -g ash-rg --query "[].name" -o tsv | ForEach-Object {
    $vm = $_
    $ip = az vm show -d -g ash-rg -n $vm --query "publicIps" -o tsv
    $user = az vm show -g ash-rg -n $vm --query "osProfile.adminUsername" -o tsv
    "$ip $user"
}
```

Example:

```text
20.189.95.79 azureuser
20.235.124.58 azureuser
20.219.133.235 azureuser
```

Save the required entries in `vm-list.txt`.

---

## 5. Create vm-list.txt

Example:

```text
20.189.95.79 azureuser
20.219.133.235 azureuser
```

For many VMs, keep one IP and username per line.

---

## 6. Copy SSH Public Key to Managed Nodes

Use:

```bash
while read -r ip user
do
    [ -z "$ip" ] && continue

    echo "===== $ip / $user ====="

    ssh-copy-id \
      -i ~/.ssh/id_ed25519.pub \
      "$user@$ip"

done < vm-list.txt
```

On the first connection, SSH may ask for host authenticity confirmation:

```text
Are you sure you want to continue connecting?
```

Enter:

```text
yes
```

Then enter the VM user's password.

Successful output:

```text
Number of key(s) added: 1
```

---

## 7. Generate Inventory Automatically

For multiple VMs, AWK can generate Ansible inventory entries:

```bash
awk '{printf "managed%02d ansible_host=%s ansible_user=%s\n", NR, $1, $2}' vm-list.txt
```

Example:

```text
managed01 ansible_host=20.189.95.79 ansible_user=azureuser
managed02 ansible_host=20.219.133.235 ansible_user=azureuser
```

---

## 8. inventory.ini

Example:

```ini
[webservers]
managed01 ansible_host=20.189.95.79 ansible_user=azureuser
managed02 ansible_host=20.219.133.235 ansible_user=azureuser

[webservers:vars]
ansible_ssh_private_key_file=/home/azureuser/.ssh/id_ed25519
```

### Important

Use:

```ini
[webservers:vars]
```

Not:

```ini
[webservers:var]
```

The incorrect `var` syntax causes an inventory parsing warning.

---

## 9. Test Ansible Connectivity

```bash
ansible webservers -i inventory.ini -m ping
```

Expected:

```text
managed01 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

managed02 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

## 10. Nginx Playbook

Create `playbook.yaml`:

```yaml
---
- name: Install Nginx on all web servers
  hosts: webservers
  become: yes

  tasks:

    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: Start Nginx
      ansible.builtin.service:
        name: nginx
        state: started

    - name: Enable Nginx at boot
      ansible.builtin.service:
        name: nginx
        enabled: yes
```

---

## 11. Validate Playbook

Syntax check:

```bash
ansible-playbook -i inventory.ini playbook.yaml --syntax-check
```

Dry run:

```bash
ansible-playbook -i inventory.ini playbook.yaml --check
```

Execute:

```bash
ansible-playbook -i inventory.ini playbook.yaml
```

---

## 12. Verify Nginx

Check from Ansible:

```bash
ansible webservers -i inventory.ini -b -m command -a "systemctl is-active nginx"
```

Or connect directly:

```bash
ssh azureuser@<VM-IP>
```

Then:

```bash
sudo systemctl status nginx
```

Expected:

```text
Active: active (running)
```

The service should also be enabled at boot.

---

## 13. Complete Automation Flow

```text
                    Azure
                      |
          +-----------+-----------+
          |                       |
    Control Node             Managed Nodes
          |                       |
    Ansible installed        Ubuntu VMs
          |                       |
    SSH key generated             |
          |                       |
    id_ed25519.pub                |
          |                       |
          +---- ssh-copy-id ------+
                      |
              authorized_keys
                      |
                      v
              Ansible Inventory
                      |
                      v
                ansible ping
                      |
                      v
                  Playbook
                      |
                      v
               Install Nginx
                      |
                      v
             Start + Enable Nginx
```

---

## 14. Important Files

```text
ansible-test/
├── vm-list.txt
├── inventory.ini
├── playbook.yaml
└── README.md
```

### vm-list.txt

Contains:

```text
IP USER
```

### inventory.ini

Defines:

- Managed nodes
- Ansible group
- SSH username
- Private key path

### playbook.yaml

Defines the automation tasks.

### README.md

Contains the complete lab documentation.

---

## 15. Top Ansible Commands

| Command | Purpose |
|---|---|
| `ansible --version` | Check Ansible installation/version |
| `ansible-inventory -i inventory.ini --list` | Validate/read inventory |
| `ansible webservers -i inventory.ini -m ping` | Test managed-node connectivity |
| `ansible webservers -i inventory.ini -m command -a "hostname"` | Get managed-node hostname |
| `ansible-playbook -i inventory.ini playbook.yaml --syntax-check` | Check playbook syntax |
| `ansible-playbook -i inventory.ini playbook.yaml --check` | Dry run |
| `ansible-playbook -i inventory.ini playbook.yaml` | Execute playbook |
| `ansible webservers -i inventory.ini -b -m command -a "systemctl is-active nginx"` | Check Nginx status |

---

## 🎯 Final Result

The lab successfully demonstrated:

- Ansible installation
- SSH ED25519 key generation
- Public-key distribution using `ssh-copy-id`
- Azure VM information collection
- Automated inventory generation using AWK
- Ansible inventory configuration
- Ansible connectivity testing
- Playbook execution
- Nginx installation
- Nginx service start
- Nginx service enablement
- Nginx verification

**Control Node → SSH → Managed Nodes → Ansible → Nginx Automation** ✅
>>>>>>> ac72820 (Add Ansible Nginx automation lab)
