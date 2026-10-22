# PHP Security RCE Mitigation (`CVE-2026-109618` & `CVE-2026-109619`)

This repository contains Ansible playbooks to apply immediate, low-risk configuration mitigations against the critical unauthenticated Remote Code Execution (RCE) vulnerabilities detailed in the October 2026 PHP Security Advisory.

## Vulnerability Overview
* **CVE-2026-109618 (RCE via OOM corruption):** Out-of-memory handling interrupts PHP while internal request data is partially initialized, leading cleanup routines to treat invalid data as valid.
* **CVE-2026-109619 (RCE via Stale Global State):** Request-owned internal state survives after backing memory is freed, allowing subsequent HTTP requests handled by the same worker process to reuse garbage state.

## Playbook Strategy

### 1. Primary Playbook (`php_rce_2026_mitigation.yml`)
Designed for **supported OS environments** (Ubuntu 22.04+, Debian 12) where standard Linux package paths and systemd services are present.

### 2. Legacy Playbook (`php_rce_2026_mitigation_legacy_servers.yml`)
Designed specifically for **EOL / Legacy OS environments** (CentOS 7, Ubuntu 20.04/14.04). Because these operating systems no longer receive upstream PHP package updates, this playbook:
* Handles both `apt` (Debian/Ubuntu) and `yum` (CentOS/RHEL) path variations (`/etc/php.ini`, `/etc/php.d/`, `/etc/php/`).
* Gracefully handles legacy web server service units (`httpd` vs `apache2`).
* Enforces strict `post_max_size = 32M` request limits to reduce OOM surfaces on unpatchable PHP builds until these hosts can be upgraded or decommissioned.

---

## Core Mitigation Steps
Both playbooks apply emergency configuration hotfixes across active PHP environments (`fpm`, `apache2`, `cli`, `httpd`):
1. **Creates Backups:** Takes date-stamped backups of all existing `php.ini` files before modification.
2. **Disables Upload Progress (`CVE-2026-109619`):** Sets `session.upload_progress.enabled = 0`.
3. **Removes PHP Soft Memory Ceiling (`CVE-2026-109618`):** Sets `memory_limit = -1`.
4. **Limits Request Body Sizes:** Restricts `post_max_size = 32M` to minimize OOM attack surfaces.
5. **Smart Service Restarts:** Dynamically detects active systemd services and restarts running `php-fpm`, `apache2`, or `httpd` units.
6. **Runtime Verification:** Checks live INI settings via PHP CLI and validates HTTP availability.

---

## Setup & Usage

### 1. Prerequisites
* Ansible 2.9+
* SSH access via non-root user (`jenkins`) with passwordless `sudo` privileges on target hosts.

### 2. Configure Inventory
Copy the template file to create your local active inventory:
```bash
cp inventory.ini-sample inventory.ini

```

Edit `inventory.ini` to add your target server IP addresses. *(Note: `inventory.ini` is ignored by git).*

### 3. Execution Workflow

#### Supported OS Servers (Category B)

```bash
# Dry-run
ansible-playbook -i inventory.ini php_rce_2026_mitigation.yml --limit dev_hosts --check --diff

# Live Run
ansible-playbook -i inventory.ini php_rce_2026_mitigation.yml --limit category_b_web

```

#### Legacy / EOL OS Servers (Category A)

```bash
# Dry-run
ansible-playbook -i inventory.ini php_rce_2026_mitigation_legacy_servers.yml --limit category_a --check --diff

# Live Run
ansible-playbook -i inventory.ini php_rce_2026_mitigation_legacy_servers.yml --limit category_a

```

---

## Repository Structure

```text
.
├── .gitignore
├── ansible.cfg                                   # Apply local ansible settings
├── README.md                                     # Repository documentation
├── inventory.ini-sample                          # Template inventory file for Git
├── php_rce_2026_mitigation.yml                   # Mitigation playbook for supported OS releases
└── php_rce_2026_mitigation_legacy_servers.yml    # Mitigation playbook for legacy/EOL OS releases

```
