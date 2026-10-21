# 🤖 Lab 07: Ansible Architecture, Configuration & Automated Inventory

## 🧠 Concept & Architecture ("What & Why")

### What is Ansible Configuration & Dynamic Inventory?
**Ansible** is an open-source, agentless configuration management and automation platform. Unlike legacy agent-based systems (such as Chef, Puppet, or SaltStack) that require a background daemon agent installed on every target node, Ansible operates over standard **SSH (TCP Port 22)** on Linux or **WinRM** on Windows.

- **Control Node**: The central workstation or CI server running Ansible binaries. It reads playbooks, parses dynamic inventory inputs, compiles Python module payloads, opens SSH sessions, and executes modules in-memory on remote target hosts.
- **Target Managed Node**: The target server (our Jenkins EC2 host deployed in Lab 06). It requires zero custom agent software — only an SSH daemon (`sshd`) and a standard Python 3 runtime environment.
- **Dynamic Inventory**: In modern elastic cloud infrastructure (AWS), EC2 instance IP addresses change dynamically whenever instances are recreated. A **dynamic inventory script** queries infrastructure outputs (e.g. `terraform output -raw jenkins_public_ip`) and dynamically generates Ansible's inventory file (`inventory/hosts.ini`), completely eliminating manual IP editing.

```text
                                ANSILE AGENTLESS INVENTORY ARCHITECTURE
                                
  +-------------------------------------+                 +-------------------------------------+
  |         ANSIBLE CONTROL NODE        |                 |      AWS EC2 TARGET MANAGED NODE    |
  |                                     |                 |  (Jenkins Server — Public Subnet A) |
  |  [ master config: ansible.cfg ]     |                 |                                     |
  |  [ script: update-inventory.sh ]    |   SSH Port 22   |  - Standard OpenSSH Daemon (sshd)   |
  |        | (Fetches TF Output IP)     | ==============> |  - Python 3 Runtime Environment     |
  |        v                            |  Pushes Python  |  - Zero Agent Software Installed    |
  |  [ inventory: hosts.ini ]           |  Module Payload |  - Systemd Service Manager          |
  +-------------------------------------+                 +-------------------------------------+
```

---

## 🎯 Objectives

- Configure master Ansible settings in `ansible.cfg` (SSH keys, host key checking, privilege escalation).
- Build automated inventory updater script `scripts/update-inventory.sh` to dynamically populate `inventory/hosts.ini` with Terraform IP outputs.
- Test SSH connectivity and Python module execution using Ansible ad-hoc modules (`ansible -m ping`).

---

## 📂 Project Structure

```text
02-Configuration-Management-Ansible/Lab07-Ansible-Inventory-Setup/
├── ansible.cfg             # Master Ansible configuration blueprint
├── README.md               # Exhaustive step-by-step lab documentation
├── inventory/
│   └── hosts.ini           # Dynamically generated target host inventory
└── scripts/
    └── update-inventory.sh # Shell script automating inventory IP generation
```

---

## 🛠 Comprehensive Code & Configuration Breakdown

### 1. Master Ansible Configuration (`ansible.cfg`)

```ini
[defaults]
inventory            = ./inventory/hosts.ini
host_key_checking    = False
remote_user          = ubuntu
private_key_file     = ~/.ssh/id_rsa
timeout              = 30
deprecation_warnings = False
stdout_callback      = yaml

[privilege_escalation]
become          = True
become_method   = sudo
become_user     = root
become_ask_pass = False
```

#### Line-by-Line Breakdown:
- **`inventory = ./inventory/hosts.ini`**: Defines default path to the host target inventory file if `-i` is omitted in CLI calls.
- **`host_key_checking = False`**: Bypasses the interactive SSH fingerprint prompt (`Are you sure you want to continue connecting (yes/no)?`), enabling fully automated non-interactive CI/CD executions.
- **`remote_user = ubuntu`**: Specifies default SSH login username matching standard Ubuntu AMIs on AWS EC2.
- **`private_key_file = ~/.ssh/id_rsa`**: Path to the SSH private key corresponding to the public key injected into the EC2 instance during Lab 06 Terraform deployment.
- **`timeout = 30`**: Sets SSH connection timeout limit in seconds.
- **`stdout_callback = yaml`**: Formats Ansible CLI output as readable YAML structure instead of raw JSON.
- **`become = True`**: Automatically escalates privileges to root for playbook execution.
- **`become_method = sudo`**: Uses `sudo` privilege escalation.
- **`become_user = root`**: Escalates target execution context to `root`.

---

### 2. Dynamic Inventory Updater Script (`scripts/update-inventory.sh`)

```bash
#!/bin/bash
set -e

JENKINS_IP=$1

if [ -z "$JENKINS_IP" ]; then
    echo "❌ Error: Missing Jenkins Public IP address."
    echo "Usage: ./scripts/update-inventory.sh <JENKINS_PUBLIC_IP>"
    exit 1
fi

echo "🔄 Updating Ansible inventory with Jenkins Public IP: ${JENKINS_IP}..."

cat <<EOF > inventory/hosts.ini
[jenkins_servers]
jenkins_host ansible_host=${JENKINS_IP} ansible_user=ubuntu

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
EOF

echo "✅ Inventory updated successfully in inventory/hosts.ini"
```

#### Line-by-Line Breakdown:
- **`set -e`**: Terminates script execution immediately if any command exits with a non-zero status.
- **`JENKINS_IP=$1`**: Captures first CLI parameter passed to the script.
- **`[jenkins_servers]`**: Defines host group alias `jenkins_servers`.
- **`jenkins_host`**: Target host alias name within Ansible playbooks.
- **`ansible_host=${JENKINS_IP}`**: Maps host alias to actual public IP address.
- **`ansible_python_interpreter=/usr/bin/python3`**: Directs Ansible to target Ubuntu's System Python 3 binary for executing pushed module payloads.

---

## 📋 Step-by-Step Execution Guide

### Step 1: Make Script Executable
```bash
cd 02-Configuration-Management-Ansible/Lab07-Ansible-Inventory-Setup
chmod +x scripts/update-inventory.sh
```

### Step 2: Extract Public IP from Lab 06 & Generate Inventory
```bash
JENKINS_IP=$(terraform -chdir=../../01-Infrastructure-Terraform/Lab06-Jenkins-EC2-AWSBackup output -raw jenkins_public_ip)
./scripts/update-inventory.sh $JENKINS_IP
```

### Step 3: Test Ad-Hoc Ping Connectivity
```bash
ansible jenkins_servers -m ping -i inventory/hosts.ini
```

---

## 🧪 Verification & Expected Terminal Output

```bash
ansible jenkins_servers -m ping -i inventory/hosts.ini
```

**Expected Terminal Output:**
```yaml
jenkins_host:
  changed: false
  ping: pong
```

Verify `inventory/hosts.ini` contents:
```bash
cat inventory/hosts.ini
```

**Expected File Output:**
```ini
[jenkins_servers]
jenkins_host ansible_host=54.210.142.88 ansible_user=ubuntu

[all:vars]
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

---

## 💡 Best Practices & Common Troubleshooting

### Best Practices:
1. **Never Commit SSH Keys**: Ensure private keys are stored securely in `~/.ssh/` with strict permissions (`chmod 400`).
2. **Use Dynamic Scripts**: Always automate inventory generation from IaC outputs rather than hardcoding static IP addresses.

### Troubleshooting Matrix:

| Error Symptom | Root Cause | Remediation Command |
|---------------|------------|---------------------|
| `Permission denied (publickey)` | Incorrect SSH key path or invalid SSH user | Verify key path in `ansible.cfg` and ensure SSH user is set to `ubuntu`. |
| `UNREACHABLE! Host connection timed out` | EC2 Security Group blocking SSH Port 22 | Check Lab 02 Security Group rules and verify Port 22 ingress is open. |

---

## 🔗 Connection to Next Lab
With Ansible SSH connectivity and dynamic host inventory verified, **Lab 08** executes Ansible playbooks across 8 modular roles to provision the full DevOps toolchain!
