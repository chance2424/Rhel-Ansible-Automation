# 🧠 RHEL Ansible Automation Portfolio

This repository contains a collection of **enterprise-grade Ansible playbooks** I’ve written to automate and maintain **Red Hat Enterprise Linux (RHEL 8/9)** systems across multiple environments (Las Vegas & Dallas clusters).  
All playbooks have been sanitized — no internal hostnames, credentials, or company-specific details are included.

These automations were designed to **simplify repetitive sysadmin tasks, enforce configuration compliance, and minimize human error** in patching, validation, and service management processes.

---

## ⚙️ Playbooks Overview

### 🩵 1. `fix-deprecated-ssh-settings.yml`

**Purpose:**  
Updates cryptographic settings in `/etc/ssh/sshd_config` to remove deprecated ciphers, key exchange algorithms, and MACs.  
Also handles immutable file attributes and safely reloads `sshd` only after syntax validation.

**Key Features:**
- Validates configuration with `sshd -t -f %s` before applying  
- Automatically re-applies immutable bit if it was originally set  
- Enforces modern secure defaults for SSH hardening  

---

### 🛰️ 2. `register-rhel-satellite.yml`

**Purpose:**  
Automates **re-registration of RHEL systems** with the correct Red Hat Satellite activation keys and environments.

**Key Features:**
- Dynamically detects and removes stale registration files  
- Registers using activation keys defined per environment (non-prod, prod, etc.)  
- Ensures hosts appear correctly in Satellite inventory  
- Logs registration results for auditing  

---

### 🔒 3. `podman-versionlock.yml`

**Purpose:**  
Applies or updates **version locks** for `podman` to ensure compliance with approved versions.

**Key Features:**
- Ensures `yum-plugin-versionlock` is installed  
- Locks to the correct version specified in group variables  
- Fully idempotent and safe for repeat runs  

---

### 🧾 4. `elk_rotated_log_cleanup.yml`

**Purpose:**  
Automates the cleanup of rotated system logs and Falcon sensor logs on ELK or RHEL servers. Designed to safely identify and remove old log copies, it includes a dry-run mode by default to preview deletions before performing any actual cleanup.

**Key Features:**
- Targets both `/var/log` and `/var/log/falcon-sensor` directories  
- Detects rotated or compressed log files (e.g. `messages-*`, `falcon-sensor.log-*`)  
- Supports configurable retention periods via `retention_days`  
- Defaults to safe mode (`perform_delete: false`) — override with `-e perform_delete=true` to perform deletions  
- Provides detailed summaries of files found or deleted for auditing  
- Ensures Falcon log directories exist before scanning  

**Use Case:**  
Ideal for environments where storage retention must be tightly managed or where compliance requires automated log cleanup across multiple ELK nodes.  

---

### 🪩 5. `logstash-versionlock.yml`

**Purpose:**  
Locks **Logstash** to a specific version (`8.19.3` in this case) on RHEL 9 systems to maintain environment stability and patch consistency.  

**Key Features:**
- Skips non–RHEL 9 hosts gracefully  
- Installs `dnf-plugins-core` if missing  
- Uses `community.general.dnf_versionlock` to enforce version lock  
- Updates all other packages normally while keeping Logstash fixed  
- Displays versionlock list for verification  

---

### 🧩 6. Role-Based Playbook Structure for AEM Automation

The **Adobe Experience Manager (AEM)** patching automation under `aem-ops/` is built using a **modular, role-based structure** designed for clarity, maintainability, and scalability across enterprise environments such as Las Vegas and Dallas.  
Each role focuses on a single, clearly defined function, allowing for efficient troubleshooting, reusability, and safe sequential execution across large server fleets.

---

#### ⚙️ Architectural Overview

This automation framework replaces legacy monolithic patching scripts with a **modular, service-aware orchestration model**.  
It ensures that all major AEM components — Author, Publisher, and Dispatcher — are patched, rebooted, and validated in sequence without disrupting other clusters.

The orchestration logic is defined in the **master playbook (`aem_master.yml`)**, which calls each role in a controlled sequence.

---

#### 🧱 Roles and Responsibilities

| Role | Purpose | Key Responsibilities |
|------|----------|----------------------|
| **stop_aem** | Prepares hosts for patching by stopping AEM processes | Detects and terminates AEM Java processes; validates port shutdown (4502, 4503, 8080) |
| **patch_reboot** | Handles system updates and reboots | Applies RHEL patches via DNF; performs controlled reboots; waits for SSH reconnect |
| **start_aem** | Restarts AEM services | Starts Java/AEM components; confirms readiness |
| **validate_dispatcher** | Validates Dispatcher functionality | Performs HTTPS checks (port 443); restarts Apache on failure; logs results |
| **validate_publisher** | Validates AEM Publisher nodes | Performs port 4503 checks; logs results; ignores transient failures |

Each role can be executed independently, allowing flexibility to re-run validation steps without redoing patching.

---

#### 📁 Project Structure

For the actual directory layout, see the image below (or refer to `Aem Playbook Structure.png`):

<p align="center">
  <img src="images/aem_structure.png" width="500" alt="AEM Role-Based Playbook Structure">
</p>

---

#### 🔁 Execution Logic

The **`aem_master.yml`** orchestrates the entire patching workflow:

```yaml
---
# Run with:
# ansible-playbook -i inventories/aem.ini playbooks/aem_master.yml
# Tags: stop, patch, reboot, start, publisher, dispatcher
 
- name: Stop AEM servers
  hosts: aut:pub:dis
  gather_facts: false
  serial: 1
  max_fail_percentage: 100
  roles:
    - { role: stop_aem, tags: ["stop"] }
 
- name: Patch and reboot servers
  hosts: aut:pub:dis
  gather_facts: true
  serial: 1
  max_fail_percentage: 100
  roles:
    - { role: patch_reboot, tags: ["patch","reboot"] }
 
- name: Start AEM servers
  hosts: aut:pub:dis
  gather_facts: true
  serial: 1
  max_fail_percentage: 100
  roles:
    - { role: start_aem, tags: ["start"] }
 
- name: Validate AEM Publisher
  hosts: pub
  gather_facts: false
  serial: 1
  max_fail_percentage: 100
  roles:
    - { role: validate_publisher, tags: ["publisher"] }
 
- name: Validate AEM Dispatcher
  hosts: dis
  gather_facts: false
  serial: 1
  max_fail_percentage: 100
  roles:
    - { role: validate_dispatcher, tags: ["dispatcher"] }


