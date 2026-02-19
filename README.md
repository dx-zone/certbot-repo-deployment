# Certbot Repo Deployment - Ansible Role

🚀 **Automate your containerized TLS certificate management with ease.**

This Ansible role simplifies the deployment of a Certbot-based mTLS (mutual TLS) stack, a containerized Let’s Encrypt and RPM repository solution for automated TLS certificate issuance and secure distribution over HTTPS. It handles everything from Docker installation to container lifecycle orchestration and DNS-01 challenge configuration.

---

## 📋 Table of Contents
- [🔍 Overview](#-overview)
- [🛠 Prerequisites](#-prerequisites)
- [🚀 Getting Started](#-getting-started)
    - [Cloning the Role](#cloning-the-role)
    - [Deployment Strategy](#deployment-strategy)
- [📦 Installation & Deployment](#-installation--deployment)
    - [1. Prepare Ansible Environment](#1-prepare-ansible-environment)
    - [2. Configure Secrets (Ansible Vault)](#2-configure-secrets-ansible-vault)
    - [3. Define Certificates & DNS Providers](#3-define-certificates--dns-providers)
    - [4. Run the Playbook](#4-run-the-playbook)
- [🧩 How the Stack Works](#-how-the-stack-works)
- [🏗 Advanced Usage](#-advanced-usage)
- [🛠 Operational Scripts](#-operational-scripts)
- [⚠️ Critical: Validation](#️-critical-validation)
- [⚙️ Configuration Reference](#-configuration-reference)

---

## 🔍 Overview
The `certbot-repo-deployment` role performs the following:
1.  **Docker Setup**: Installs Docker Engine and Compose v2.
2.  **Repo Sync**: Clones the core [Certbot RPM mTLS repository](https://github.com/dx-zone/certbot-rpm-mtls-repo).
3.  **Config Generation**: Renders `certificates.csv` and DNS provider `.ini` files using Jinja2 templates.
4.  **Orchestration**: Starts the stack using `docker compose`.

---

## 🛠 Prerequisites
- **Control Node**: Ansible 2.15+ installed.
- **Target Host**: Linux (Ubuntu/Debian/RHEL/CentOS) with SSH access.
- **Permissions**: The user running the Ansible role must have `become` privileges.

---

## 🚀 Getting Started

### Cloning the Role
To use this role, you should clone it into your Ansible project's `roles` directory.

```bash
# Navigate to your Ansible project
mkdir -p roles
cd roles

# Clone the role
git clone https://github.com/YOUR_USERNAME/certbot-repo-deployment.git
```

### Deployment Strategy
This role is flexible and can be used in different ways:

*   **Localhost Deployment**: Run everything on the same machine where you have the code.
*   **Remote Server Deployment**: Use your local machine (Control Node) to deploy the stack to a remote server.
*   **CI/CD Pipelines**: Integrate into GitHub Actions, GitLab CI, Jenkins, etc.
*   **Ansible Tower / AWX**: Use it as part of a Job Template for automated enterprise deployment.

---

## 📦 Installation & Deployment

### 1. Prepare Ansible Environment

Create an `ansible.cfg` in your project root to ensure Ansible knows where to find the role:

```ini
[defaults]
# Directory where Ansible looks for roles
roles_path = ./roles
# Default inventory file
inventory = ./inventory/hosts.ini
# Prevent gathering facts automatically if not needed (optional)
inject_facts_as_vars = False
```

Define your target host in `inventory/hosts.ini`:

```ini
[certbot_hosts]
# For localhost deployment:
# localhost ansible_connection=local
# For remote deployment:
192.168.1.100 ansible_user=root
```

### 2. Configure Secrets (Ansible Vault)

Sensitive information like API tokens should NEVER be stored in plain text.

1.  Create a vault password file (and add it to `.gitignore`!):
    ```bash
    echo "your_secure_password" > .vault_pass
    ```

2.  Create and encrypt your secrets:
    ```bash
    mkdir -p inventory/group_vars/certbot_hosts
    ansible-vault create --vault-id dev@.vault_pass inventory/group_vars/certbot_hosts/vault.yml
    ```

3.  Inside the vault file, add your credentials:
    ```yaml
    cloudflare_api_token: "your-secret-token"
    # Add other providers as needed (rfc2136_tsig_key_secret, etc.)
    ```
    
    **NOTE:** Refer to `vault.yml.example` for a detailed description of the required variables for each supported DNS provider, including their purpose and expected format.

### 3. Define Certificates & DNS Providers

Create a `vars/main.yml` or add to your `group_vars` to define which certificates to issue.

```yaml
certbot_dns_providers:

  # ---------------------------------------------------------------------------
  # Cloudflare DNS provider
  # ---------------------------------------------------------------------------
  cloudflare:
    enabled: True
    template: "cloudflare.ini.j2"
    filename: "cloudflare.ini"
    certificate:
      # List of domain/email pairs to be written into certificates.csv
      # Note: template logic supports multiple FQDNs and/or emails per entry.
      domains:
        - fqdn:
            - "repo.example.net"
          email:
            - "admin@example.net"
        - fqdn:
            - "app.example.net"
          email:
            - "admin@example.net"
        - fqdn:
            - "web.example.net"
          email:
            - "admin@example.net"
        - fqdn:
            - "email.example.net"
          email:
            - "admin@example.net"
        - fqdn:
            - "test.example.net"
          email:
            - "admin@example.net"

  # ---------------------------------------------------------------------------
  # AWS Route53 (placeholder)
  # NOTE: certbot Route53 auth typically uses AWS credentials/roles rather than
  # a single "token". Keep disabled unless implemented end-to-end.
  # ---------------------------------------------------------------------------
  route53:
    enabled: False
    template: "route53.ini.j2"
    filename: "route53.ini"
    certificate:
      domains:
        - fqdn:
            - "example.com"
          email:
            - "admin@example.com"

  # ---------------------------------------------------------------------------
  # RFC2136 dynamic DNS updates (TSIG) - placeholder
  # ---------------------------------------------------------------------------
  rfc2136:
    enabled: False
    template: "rfc2136.ini.j2"
    filename: "rfc2136.ini"
    certificate:
      domains:
        - fqdn:
            - "example.com"
          email:
            - "admin@example.com"
```

### 4. Run the Playbook

Create a simple playbook `deploy.yml` to reference the role for execution.

**NOTE:** Facts gathering is required for template header rendering.

```yaml
- name: Deploy certbot-rpm-mtls-repo via Ansible role
  hosts: certbot_hosts
  become: true
  gather_facts: true  # Facts gathering required for template rendering
  vars:
    # Optional: Only gather subset of facts to speed up execution
    gather_subset:
      - '!all'
      - 'min'
      - 'date_time'

  roles:
    - role: certbot-repo-deployment
```

Execute the deployment:

```bash
ansible-playbook -i inventory/hosts.ini deploy.yml --vault-id dev@.vault_pass
```

---



## 🧩 How the Stack Works

New to **Ansible Role, Docker, Certbot,** or **DNS-01**?
 Here’s the “Magic Path” explained simply.

------

### What is an Ansible Role?

Think of a **Role** as a reusable automation package.

You don’t rewrite infrastructure logic.
 You only provide configuration (variables), and the role handles:

- Installing Docker
- Cloning the repository
- Creating directories
- Rendering templates
- Starting containers
- Validating the stack

👉 You provide intent.
 👉 The role executes it consistently.

------

### What is DNS-01 (Let’s Encrypt Validation)?

Let’s Encrypt must verify you control a domain before issuing a certificate.

There are multiple methods — this project uses **DNS-01**:

- Certbot creates a temporary TXT record in your DNS zone.
- Let’s Encrypt checks that record.
- If valid → certificate is issued.

That’s why DNS API credentials are required.

Examples:

- Cloudflare API Token
- AWS Route53 credentials
- RFC2136 TSIG keys

⚠️ These must be stored securely (Ansible Vault recommended).

------

### Can I Test This Locally?

Yes.

In your inventory:

```ini
[certbot_hosts]
localhost ansible_connection=local
```

Then run:

```bash
ansible-playbook playbooks/deploy-certbot-repo.yml
```

This will deploy the full stack on your own machine using Docker.

------

### Where Are My Certificates?

After deployment completes:

#### 🔐 Let’s Encrypt Certificates

```bash
/opt/certbot/datastore/certbot-data/letsencrypt
```

#### 📦 Generated RPM Packages

```bash
/opt/certbot/datastore/rpmrepo-data/rpms
```

The certificates are also packaged automatically into RPMs for distribution.

------

### What Do the Containers Actually Do?

This stack deploys **three services**:

------

#### 🟢 1. `certbot` (Certificate Factory)

- Reads `certificates.csv`
- Issues or renews certificates
- Re-checks inventory every 60 minutes (default)
- Packages certificates into RPM files
- Stores them in shared RPM directory

Think of it as an automated certificate pipeline.

------

#### 🔵 2. `rpmrepo` (Secure Distribution Server)

- Serves RPM packages over HTTPS
- Uses mTLS (mutual TLS)
- Generates internal CA + client certificates
- Only authorized Linux clients can download RPMs

This simulates a production-grade internal certificate distribution model.

------

#### 🟣 3. `client-test` (Linux Client Simulator)

- Acts like a real Linux machine
- Has:
  - `/etc/yum.repos.d/cert.repo`
  - client certificate
  - private key
  - trusted CA
- Validates:
  - TLS trust
  - mTLS authentication
  - RPM download integrity

It exists purely to validate the entire PKI + distribution pipeline.

------

## 🔍 Deployment Validation & Operational Control

After deployment completes, use the operational scripts below to **manage, validate, and audit** the entire mTLS certificate distribution pipeline.

These scripts are designed for:

- 🏗 Infrastructure control
- 🔐 PKI lifecycle management
- 📦 RPM repository validation
- 🧪 End-to-end trust verification

------

## 🧭 Stack Lifecycle Manager

**Script:** `manage-certbo-repo-client-stack.sh`

Primary control interface for the entire stack.

```
sudo ./manage-certbo-repo-client-stack.sh [TARGET]
```

### 🎯 Available Targets

| Target    | Purpose                                         | When to Use                    |
| --------- | ----------------------------------------------- | ------------------------------ |
| `init`    | Prepare directories, permissions, PKI workspace | First-time deployment          |
| `pki`     | Generate or rotate mTLS client certificates     | Manual CA/client rotation      |
| `up`      | Start stack and wait for health checks          | Normal startup                 |
| `rebuild` | Recreate containers                             | After `.env` or config changes |
| `status`  | Show container state + certificate info         | Quick health snapshot          |
| `check`   | Run diagnostics                                 | Troubleshooting                |
| `logs`    | Follow container logs                           | Debugging                      |
| `down`    | Stop containers                                 | Maintenance                    |
| `purge`   | Delete all persistent data                      | Full reset (destructive)       |
| `clean`   | Remove data + images + orphans                  | Lab teardown                   |

> 🔎 Use `-v` for verbose mode:
>
> ```bash
> sudo ./manage-certbo-repo-client-stack.sh -v status
> ```

This script is your **control plane**.

------

## 🧪 Full Pipeline Validation

**Script:** `validate-pki-pipeline.sh`

Performs a full-spectrum validation of:

- Container health
- Certbot lifecycle state
- Internal DNS resolution
- PKI regeneration & rotation
- Apache reload
- mTLS authentication
- RPM repository access

```bash
sudo ./validate-pki-pipeline.sh
```

### What It Verifies

| Validation Stage      | What It Confirms                             |
| --------------------- | -------------------------------------------- |
| Infrastructure Health | All services are running                     |
| Certbot Provisioning  | Certificates are valid and processed         |
| Internal Networking   | FQDN resolves inside Docker network          |
| PKI Rotation          | CA and client material regenerated correctly |
| Apache Reload         | Server picks up new trust material           |
| mTLS Session          | Client authenticates successfully            |
| Repository Access     | RPM endpoint returns HTTP 200                |

If this script finishes with:

```bash
✅ DEPLOYMENT READY
```

Your system is cryptographically functional end-to-end.

------

## 📦 RPM Repository Verification

**Script:** `verify-rpm-repo.sh`

Validates that:

- Repository metadata exists
- RPM packages are indexable
- Clients can retrieve packages
- Repo configuration is valid

Use when:

- RPMs were newly generated
- Repo metadata was refreshed
- Clients report yum/dnf errors

------

## 🔄 mTLS Audit & Rotation Testing

**Script:** `rpmrepo-mtls-audit-rotation.sh`

Focused security audit script.

Validates:

- CA trust chain integrity
- Client certificate expiration
- Proper file permissions
- Successful Apache reload after rotation
- mTLS authentication enforcement

Use when:

- Rotating client identities
- Rotating CA
- Auditing trust chain integrity
- Preparing for production hardening

------

## 🧠 Recommended Operational Flow

For production validation:

```bash
sudo ./manage-certbo-repo-client-stack.sh up
sudo ./validate-pki-pipeline.sh
```

For mTLS client certificate rotation and testing:

```bash
sudo ./manage-certbo-repo-client-stack.sh pki
# or
sudo rpmrepo-mtls-audit-rotation.sh
sudo ./validate-pki-pipeline.sh

```

For stack configuration changes:

```bash
sudo ./manage-certbo-repo-client-stack.sh init
sudo ./manage-certbo-repo-client-stack.sh rebuild
```

For troubleshooting:

```bash
sudo ./manage-certbo-repo-client-stack.sh status
sudo ./manage-certbo-repo-client-stack.sh logs
sudo ./manage-certbo-repo-client-stack.sh check
```

------

## 🏁 When Are You “Done”?

You are production-ready when:

- ✅ Certbot shows VALID status
- ✅ rpmrepo is healthy
- ✅ mTLS session returns HTTP 200
- ✅ Client can access `/rpms/`
- ✅ No expired certificates
- ✅ DNS resolution matches expected network path

---

## 🏗 Advanced Usage

### CI/CD Pipelines
You can run this role in a pipeline by providing the Vault password as an environment variable or a secret.
*   **GitHub Actions**: Use `ansible-playbook` with `--vault-password-file` pointing to a temporary file created from a GitHub Secret.

### Ansible Tower / AWX
1.  Add this repo as a **Project**.
2.  Create **Machine Credentials** for SSH access to your target.
3.  Create **Vault Credentials** for your secrets.
4.  Create a **Job Template** using the `deploy.yml` playbook.

---

## 🛠 Operational Scripts

After deployment, the cloned repository at:

```
{{ certbot_repo_dir }}   # default: /opt/certbot
```

contains lifecycle and validation scripts used to operate and audit the stack.

> ⚠️ These scripts run **on the target host**, not from your Ansible control node.

> 📘 For deep implementation details, read the `README.md` inside the cloned repository on the target host or visit the upstream repository:
>  https://github.com/dx-zone/certbot-rpm-mtls-repo

------

### 🧰 Stack Lifecycle & Validation Tools

| Script                               | Purpose                                                      | When to Use                                                  | Example                                            |
| ------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | -------------------------------------------------- |
| `manage-certbo-repo-client-stack.sh` | 🔧 **Unified Stack Manager** – Controls container lifecycle, health, rebuilds, and PKI workspace prep. | Daily operations, restarts, rebuild after `.env` or config changes | `sudo ./manage-certbo-repo-client-stack.sh status` |
| `validate-pki-pipeline.sh`           | 🧪 **Full-Spectrum Validation** – End-to-end check of containers, DNS resolution, certificate state, mTLS handshake, and repository access. | After first deployment or before production cutover          | `sudo ./validate-pki-pipeline.sh`                  |
| `verify-rpm-repo.sh`                 | 📦 **Repository Functional Test** – Verifies RPM metadata and authenticated client access via DNF. | Quick repo sanity check                                      | `./verify-rpm-repo.sh`                             |
| `rpmrepo-mtls-audit-rotation.sh`     | 🔐 **mTLS Audit & Rotation Tool** – Regenerates client CA and certificates, reloads Apache safely, validates secure access. | Scheduled rotation, security hardening, incident response    | `./rpmrepo-mtls-audit-rotation.sh`                 |

------

### 🧠 Operational Flow (Recommended Order)

For a fresh deployment validation:

1. `manage-certbo-repo-client-stack.sh up`
2. `validate-pki-pipeline.sh`
3. (Optional) `verify-rpm-repo.sh`

For certificate or CA rotation:

1. `rpmrepo-mtls-audit-rotation.sh`
2. `validate-pki-pipeline.sh`

---

## ⚠️ Critical: Validation

After deployment, it is highly recommended to run the validation script to ensure everything is working correctly.

```bash
# Navigate to the deployment directory on the target host
cd /opt/certbot

# Check the containers' deployment status
sudo ./manage-certbo-repo-client-stack.sh status

# Run full deployment validation with sudo
sudo ./validate-pki-pipeline.sh

# Validate the rpmrepo connetion by testing RPM access with mTLS material
sudo ./verify-rpm-repo.sh

# Optional: Regenerate new mTLS material for client authentication, relod and test the rpmrepo
rpmrepo-mtls-audit-rotation.sh
```

---

## ⚙️ Configuration Reference

All variables can be overridden at **playbook**, **inventory**, **group_vars**, or **host_vars** level unless otherwise stated.

> ✅ **Precedence reminder:** `roles/<role>/vars/main.yml` has **high precedence** and will override many other sources. Use it for **role structure defaults**, not environment-specific secrets or domains.

------

## 🐳 Docker & Runtime Infrastructure

| Variable                    | Default | Description                                                  |
| --------------------------- | ------- | ------------------------------------------------------------ |
| `docker_install_flavor`     | `"ce"`  | Install Docker Community Edition (`ce`) or use distro packages (`distro`, future). |
| `docker_run_container_test` | `True`  | Runs a container lifecycle validation after Docker install.  |
| `docker_compose_plugin`     | `True`  | Ensures Docker Compose v2 plugin is installed.               |

------

## 📦 Repository Deployment Settings

| Variable               | Default          | Description                                            |
| ---------------------- | ---------------- | ------------------------------------------------------ |
| `certbot_repo_url`     | GitHub URL       | Git repository containing the certbot + rpmrepo stack. |
| `certbot_repo_version` | `"main"`         | Branch, tag, or commit to deploy.                      |
| `certbot_repo_dir`     | `"/opt/certbot"` | Target directory where the repository is cloned.       |

------

## 📁 Filesystem & Ownership

| Variable             | Default  | Description                                         |
| -------------------- | -------- | --------------------------------------------------- |
| `certbot_repo_owner` | `"1000"` | Owner UID applied to repo + bind-mount directories. |
| `certbot_repo_group` | `"1000"` | Group GID applied to repo + bind-mount directories. |

> 💡 Default assumes containers run as **UID/GID 1000**.

------

## 💾 Persistent Host Paths

These directories live on the **target host** and are bind-mounted into containers.

| Variable           | Default                                                      | Purpose                                    |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------ |
| `certbot_data_dir` | `{{ certbot_repo_dir }}/datastore/certbot-data/letsencrypt`  | Let’s Encrypt storage (certbot container). |
| `rpmrepo_rpms_dir` | `{{ certbot_repo_dir }}/datastore/rpmrepo-data/rpms`         | RPM package storage (shared repo content). |
| `certbot_ini_dir`  | `{{ certbot_repo_dir }}/secrets/certbot-secrets/ini`         | DNS provider credential files (`*.ini`).   |
| `rpmrepo_pki_dir`  | `{{ certbot_repo_dir }}/secrets/rpmrepo-secrets/pki_mtls_material` | mTLS CA + client PKI material.             |

------

## 🧾 Template Output Paths

| Variable                      | Default                                   | Description                                                  |
| ----------------------------- | ----------------------------------------- | ------------------------------------------------------------ |
| `certbot_cert_inventory_file` | `{{ certbot_data_dir }}/certificates.csv` | Inventory CSV consumed by certbot manager (fqdn, provider, email). |

------

## 🌐 Stack Environment File (.env)

The upstream docker-compose stack expects a `.env` file in the repository root.
 This role renders it so deployment is **reproducible** and **non-interactive**.

| Variable                    | Default                       | Description                                                  |
| --------------------------- | ----------------------------- | ------------------------------------------------------------ |
| `certbot_stack_repo_fqdn`   | `"repo.example.com"`          | Primary HTTPS hostname for `rpmrepo` (Apache). Must appear in `certificates.csv` for LE issuance. |
| `certbot_stack_ca_name`     | `"Internal-RPM-Repo-CA"`      | Display/common-name label for the internal mTLS client CA (Linux trust store). |
| `certbot_stack_client_name` | `"client-identity"`           | Identity prefix for generated client cert/key (used by `client-test` and client config). |
| `certbot_stack_env_file`    | `{{ certbot_repo_dir }}/.env` | Path where the role writes the rendered `.env` file.         |

------

## 🔧 Docker Compose Behavior Controls

Controls how `community.docker.docker_compose_v2` behaves.

| Variable           | Default    | Options                     | Description                  |
| ------------------ | ---------- | --------------------------- | ---------------------------- |
| `compose_build`    | `"policy"` | `policy`, `always`, `never` | Image build behavior.        |
| `compose_pull`     | `"policy"` | `policy`, `always`, `never` | Image pull behavior.         |
| `compose_recreate` | `"auto"`   | `auto`, `always`, `never`   | Container recreation policy. |

------

## 🧪 Post-Deployment Validation

| Variable                  | Default | Description                                                 |
| ------------------------- | ------- | ----------------------------------------------------------- |
| `run_pipeline_validation` | `True`  | Executes `validate-pki-pipeline.sh` after the stack starts. |

------

## 🔐 DNS Provider Secrets (Sensitive)

⚠️ Never store real secrets in `defaults/main.yml`.

Override via:

- `inventory/group_vars/<group>/vault.yml` (recommended)
- `inventory/host_vars/<host>/`
- External secret managers
- Ansible Vault

------

### ☁️ Cloudflare

| Variable               | Description                                     |
| ---------------------- | ----------------------------------------------- |
| `cloudflare_api_token` | Cloudflare API token with DNS edit permissions. |

------

### 🟧 AWS Route53

| Variable              | Description                                                  |
| --------------------- | ------------------------------------------------------------ |
| `route53_creds_token` | Placeholder (Route53 typically uses AWS creds/roles rather than a single token). |

------

### 🧷 RFC2136 TSIG

| Variable                     | Description                           |
| ---------------------------- | ------------------------------------- |
| `rfc2136_tsig_key_name`      | TSIG key name.                        |
| `rfc2136_tsig_key_secret`    | TSIG secret value.                    |
| `rfc2136_tsig_key_algorithm` | TSIG algorithm (e.g., `hmac-sha256`). |
| `rfc2136_tsig_key_server`    | DNS server hostname or IP.            |
| `rfc2136_tsig_key_port`      | DNS server port.                      |

------

### 🧩 BIND TSIG-Style

| Variable              | Description                |
| --------------------- | -------------------------- |
| `bind_tsig_name`      | TSIG key name.             |
| `bind_tsig_secret`    | TSIG secret value.         |
| `bind_tsig_algorithm` | TSIG algorithm.            |
| `bind_tsig_server`    | DNS server hostname or IP. |
| `bind_tsig_port`      | DNS server port.           |

------

## 🌐 DNS Provider Definitions

These define **which providers are enabled**, which templates are rendered, and which **domains/emails** get written to `certificates.csv`.

✅ Recommended home (so users can override cleanly):

- `inventory/group_vars/certbot_hosts/*.yml`

⚠️ If defined in `roles/.../vars/main.yml`, it becomes **hard to override**.

------

## Provider Structure

```
certbot_dns_providers:
  cloudflare:
    enabled: true
    template: "cloudflare.ini.j2"
    filename: "cloudflare.ini"
    certificate:
      domains:
        - fqdn:
            - "repo.example.net"
          email:
            - "admin@example.net"
```

### Field Explanation

| Field                 | Purpose                                                     |
| --------------------- | ----------------------------------------------------------- |
| `enabled`             | Activates provider for CSV generation + template rendering. |
| `template`            | Jinja2 template used to generate provider `.ini`.           |
| `filename`            | Output filename written under `certbot_ini_dir`.            |
| `certificate.domains` | Domain/email pairs written to `certificates.csv`.           |

