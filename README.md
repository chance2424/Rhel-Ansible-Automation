# 🧠 RHEL Ansible Automation Portfolio

This repository contains a collection of **enterprise-grade Ansible playbooks** I’ve written to automate and maintain **Red Hat Enterprise Linux (RHEL 8/9)** systems across multiple environments (Vegas & Dallas clusters).  
All playbooks have been sanitized — no internal hostnames, credentials, or company-specific details are included.

These automations were originally designed to **simplify repetitive sysadmin tasks, enforce configuration compliance, and minimize human error** in patching, validation, and service management processes.

---

## ⚙️ Playbooks Overview

### 🩵 1. `fix-deprecated-ssh-settings.yml`
**Purpose:**  
Updates cryptographic settings in `/etc/ssh/sshd_config` to remove deprecated ciphers, key exchange algorithms, and MACs.  
Also handles immutable file attributes and safely reloads `sshd` only after syntax validation.

**Key Features:**
- Validates new config with `sshd -t -f %s` before applying  
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

🧾 4. elk_rotated_log_cleanup.yml

Purpose:
Automates the cleanup of rotated system logs and Falcon sensor logs on ELK or RHEL servers. Designed to safely identify and remove old log copies, it includes a dry-run mode by default to preview deletions before performing any actual cleanup.

Key Features:

Targets both /var/log and /var/log/falcon-sensor directories

Detects rotated or compressed log files such as:

messages-*, messages-*.gz

falcon-sensor.log-*, falcon-sensor.log-*.gz

Supports configurable retention periods via retention_days

Defaults to safe mode (perform_delete: false) — can be overridden using -e perform_delete=true for real deletion

Provides a detailed summary of files found or deleted for audit purposes

Ensures the Falcon log directory exists before scanning

Example use case:
This playbook is ideal for environments where log rotation retention must be managed tightly due to limited storage or compliance requirements. It ensures system stability and reduces manual log maintenance across multiple ELK nodes.
---

---
### 🪩 5. `logstash-versionlock.yml`
**Purpose:**  
Locks **Logstash** to an approved version (`8.19.3`) on RHEL 9 systems to maintain environment stability and patch consistency.  

**Key Features:**
- Skips non–RHEL 9 hosts gracefully  
- Installs `dnf-plugins-core` if missing  
- Uses `community.general.dnf_versionlock` to enforce version lock  
- Updates all other packages normally while keeping Logstash fixed  
- Displays versionlock list for verification 
---

🧩 6. Role-Based Playbook Structure for AEM Automation

The Adobe Experience Manager (AEM) patching automation under aem-ops/ is built using a modular, role-based structure designed for clarity, maintainability, and scalability across enterprise environments such as Las Vegas and Dallas.
Each role focuses on a single, clearly defined function, allowing for efficient troubleshooting, reusability, and safe parallel execution across large server fleets.

⚙️ Architectural Overview

This automation framework was designed to replace monolithic patching scripts with a modular, service-aware orchestration model.
The structure ensures that every major AEM component — Author, Publisher, and Dispatcher — is patched, rebooted, and validated in sequence without disrupting other nodes or clusters.

The orchestration logic is defined in the master playbook (aem_master.yml), which calls each role in order, ensuring that every step (stop, patch, start, validate) is executed consistently.

🧱 Roles and Responsibilities
Role	Purpose	Key Responsibilities
stop_aem	Prepares the host for patching by stopping all AEM-related processes	- Detects and terminates AEM Java processes
- Verifies shutdown by checking ports (4502, 4503, 8080)
- Ensures services are fully stopped before patching
patch_reboot	Handles RHEL system updates and reboot sequencing	- Applies security and OS updates via dnf
- Performs controlled reboots
- Waits for SSH re-connection after reboot
start_aem	Restarts AEM after reboot	- Starts Java and supporting AEM services
- Waits for startup to complete and validates service availability
validate_dispatcher	Performs HTTPS health checks for AEM Dispatcher nodes	- Tests port 443 with Ansible’s uri module
- Restarts Apache (httpd) automatically on failure
- Logs validation output for reporting
validate_publisher	Ensures AEM Publisher nodes are healthy post-patch	- Sends HTTP checks to port 4503
- Logs reachability and response codes without stopping the playbook

Each of these roles operates independently, allowing targeted re-runs (for example, validating Dispatchers without repeating patching) while maintaining a consistent orchestration model.

📁 Actual Project Structure
aem-ops/
├── ansible.cfg
├── group_vars/
│   └── all.yml
├── inventories/
│   └── aem.ini
├── playbooks/
│   └── aem_master.yml
└── roles/
    ├── patch_reboot/
    │   ├── defaults/
    │   │   └── main.yml
    │   └── tasks/
    │       └── main.yml
    ├── start_aem/
    │   └── tasks/
    │       └── main.yml
    ├── stop_aem/
    │   └── tasks/
    │       └── main.yml
    ├── validate_dispatcher/
    │   ├── defaults/
    │   │   └── main.yml
    │   ├── handlers/
    │   │   └── main.yml
    │   └── tasks/
    │       └── main.yml
    └── validate_publisher/
        ├── defaults/
        │   └── main.yml
        └── tasks/
            └── main.yml
            

🔁 Execution Logic

The aem_master.yml playbook orchestrates the full automation pipeline by executing each role in sequence:

- name: Full AEM Patching Workflow
  hosts: all
  become: yes
  serial: 1
  any_errors_fatal: false
  max_fail_percentage: 100

  roles:
    - stop_aem
    - patch_reboot
    - start_aem
    - validate_dispatcher
    - validate_publisher


Execution highlights:

serial: 1 — ensures one host is processed at a time (safe for production).

max_fail_percentage: 100 — ensures all hosts are attempted even if some fail.

Each task is self-contained with ignore_errors and descriptive logging to prevent partial run failures from halting the pipeline.

🧠 Design Rationale

This modular architecture was developed with enterprise-scale automation in mind. It adheres to industry best practices:

Reliability — Each stage validates its own success or gracefully handles failure.

Reusability — Roles can be called from other playbooks (e.g., staging or prod patching).

Transparency — Structured logs per role make debugging and auditing simple.

Scalability — Easily expandable to add new roles (e.g., backup validation, post-patch reports).

Safety-First Design — Sequential execution minimizes service downtime and ensures controlled rollouts.

🚀 Operational Flow Summary

Stop AEM → Graceful shutdown of all running AEM processes

Patch & Reboot → Apply RHEL 8/9 updates and reboot

Start AEM → Restart and verify AEM components

Validate Dispatcher → Confirm HTTPS (443) and Apache responsiveness

Validate Publisher → Confirm AEM Publisher health (port 4503)

This structure ensures full automation coverage while allowing flexibility to run individual components independently.

## 🧩 Repository Structure

| Folder | Description |
|--------|--------------|
| `playbooks/` | Core automation playbooks |
| `roles/` | Modular roles for patching, validation, and service control |
| `group_vars/` | Environment-specific variables (e.g., Vegas, Dallas) |
| `inventories/` | Example inventory files for prod/non-prod |
| `docs/` | Reference material (AEM flow diagrams, patching logic, etc.) |

---

## 🧠 Tools & Technologies

- **Red Hat Enterprise Linux 8/9**
- **Ansible Automation Platform**
- **YAML
- **Systemctl / SSH / DNF**
- **Bash for command automation**
- **Python for supplemental scripts**

---

