# Rhel-Ansible-Automation
Enterprise-style Ansible playbooks for automating RHEL 8/9 patching, registration, and validation.

Fix Deprecated SSH Cryptographic Settings (RHEL 8/9)
Updates Ciphers, KexAlgorithms, and MACs in /etc/ssh/sshd_config to secure defaults.

Validates configuration with sshd -t before reloading.

Temporarily removes and then restores the immutable bit if it was set.

Run on a test host first and keep an active SSH session while applying.
