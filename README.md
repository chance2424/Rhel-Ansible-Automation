# RHEL Ansible Automation

This repository contains a collection of Ansible playbooks and roles used to automate Red Hat Enterprise Linux (RHEL) operations such as patching, SSH hardening, Satellite registration, and service validation.
All examples are sanitized — no real hostnames, credentials, or IPs are included.

Fix Deprecated SSH Cryptographic Settings (RHEL 8/9)
Updates Ciphers, KexAlgorithms, and MACs in /etc/ssh/sshd_config to secure defaults.

Validates configuration with sshd -t before reloading.

Temporarily removes and then restores the immutable bit if it was set.

Run on a test host first and keep an active SSH session while applying.
