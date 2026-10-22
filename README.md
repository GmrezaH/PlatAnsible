# Ansible Automation for Platform Engineering

This directory contains **Ansible playbooks, roles, collections and inventory structures** used to provision, harden and configure platform infrastructure components (VMs, bare-metal, cloud instances).

Main purposes:

- OS & SSH hardening (CIS-inspired baseline + company policy)
- Container runtime installation (Docker)
- Reverse proxy setup (Traefik)
- Deployment of core platform services: GitLab, Nexus, GitLab Runner, HAProxy, Keepalived, ELK stack (Elasticsearch + Kibana), observability components

## Directory Structure

```text
ansible/
├── collections/                    # Downloaded Galaxy collections (supports offline/air-gapped)
│   ├── community.docker/
│   └── devsec.hardening/
├── inventory/
│   ├── sample/                     # Template – never run directly from here
│   │   ├── inventory.ini
│   │   ├── group_vars/
│   │   │   ├── all.yml
│   │   │   ├── lb.yml
│   │   │   └── ...
│   │   └── host_vars/
│   └── {project_name}/             # Real environments – created by copying sample
│       ├── inventory.ini
│       ├── group_vars/
│       └── host_vars/
├── playbooks/                      # Main entry points – one purpose per playbook
│   ├── bootstrap.yml
│   ├── hardening.yml
│   ├── docker.yml
│   ├── traefik.yml
│   ├── gitlab.yml
│   ├── runner.yml
│   ├── nexus.yml
│   ├── elastic.yml
│   ├── kibana.yml
│   └── observability.yml
├── roles/                          # Reusable, idempotent roles
│   ├── os_hardening/
│   ├── ssh_hardening/
│   ├── docker/
│   ├── traefik/
│   ├── gitlab/
│   ├── gitlab_runner/
│   ├── nexus/
│   ├── elasticsearch/
│   ├── kibana/
│   └── ...
└── requirements.txt                # Python packages needed by control node

```

## 1. Creating a new environment / project

Never modify files directly in `inventory/sample/`.

Instead — for every new environment or project:

```bash
cp -rfp inventory/sample inventory/production
cp -rfp inventory/sample inventory/staging
cp -rfp inventory/sample inventory/customer-xyz
```

Then edit only files inside your new copy:

- `inventory/{project_name}/inventory.ini`
- `inventory/{project_name}/group_vars/*.yml`
- `inventory/{project_name}/host_vars/*.yml`

This keeps the template clean and reusable.

## 2. **MANDATORY** — Encrypt all sensitive variables

The file inventory/{env}/group_vars/all/vault.yml is reserved for secrets:

- passwords
- API tokens
- private keys
- database credentials
- any other sensitive data

Encrypt it immediately after creating the environment:

```bash
ansible-vault encrypt inventory/production/group_vars/all/vault.yml
```

You will be prompted to enter a vault password (or use `--vault-password-file` / `--vault-id` if using a file or keyring).

After encryption the file content will look like:

```yaml
$ANSIBLE_VAULT;1.1;AES256
613238653... (encrypted content)
```

From this moment forward **never commit the plaintext version**.

## 3. Control Node (Ansible server) Preparation

### System packages (RHEL / AlmaLinux / Rocky / CentOS Stream)

```bash
sudo dnf install -y \
  python3.12 \
  python3.12-pip \
  python3.12-devel \
  gcc \
  libselinux-python3
```

### Virtual environment & Ansible installation

```bash
python3.12 -m venv .venv
source .venv/bin/activate

pip install --upgrade pip setuptools wheel
pip install ansible-core>=2.16 ansible>=10.0
pip install -U -r requirements.txt
```

### Galaxy collections (online)

```bash
ansible-galaxy collection install \
  devsec.hardening \
  community.docker \
```

### Galaxy collections (offline / air-gapped)

1. On internet-connected machine:

```bash
ansible-galaxy collection download \
  devsec.hardening community.docker \
  -p ./collections
```

2. Transfer folder to air-gapped control node

3. Install:

```bash
cd collections
ansible-galaxy collection install --requirements-file requirements.yml --offline
```

Verify:

```bash
ansible-galaxy collection list | grep -E 'devsec.hardening|community.docker'
```

## 4. Important Customizations that should be Applied

- `roles/os_hardening/tasks/user_accounts.yml`  
  → All lines with `password_expire_warn` must be commented out.

- `roles/ssh_hardening/files/banner.txt`  
  → Custom restricted-access legal banner with ASCII art

  ```bash
  cat <EOF > roles/ssh_hardening/files/banner.txt
  #################################################################
  #                   _    _           _   _                      #
  #                  / \  | | ___ _ __| |_| |                     #
  #                 / _ \ | |/ _ \ '__| __| |                     #
  #                / ___ \| |  __/ |  | |_|_|                     #
  #               /_/   \_\_|\___|_|   \__(_)                     #
  #                                                               #
  #  You are entering into a secured area! Your IP, Login Time,   #
  #   Username has been noted and has been sent to the server     #
  #                       administrator!                          #
  #   This service is restricted to authorized users only. All    #
  #            activities on this system are logged.              #
  #  Unauthorized access will be fully investigated and reported  #
  #        to the appropriate law enforcement agencies.           #
  #################################################################
  EOF
  ```

## 5. Target Nodes Preparation (bootstrap)

Most playbooks expect Python + basic packages on managed hosts.

Run once per new host (uses raw module – works even without Python):

```bash
ansible all -i inventory/production/inventory.ini \
  -m raw -a "dnf install -y python3.12 python3.12-pip python3-libselinux python3.12-dnf"
```

## 6. Variable & Configuration Best Practices

**Golden rule – never change variables inside `roles` for environment-specific settings!**

Correct approach (Ansible variable precedence best practice):

- **Role defaults** (`roles/*/defaults/main.yml`)  
  → Only sensible global defaults (rarely overridden)

- **Role vars** (`roles/*/vars/main.yml`)  
  → Fixed logic / internal constants (almost never overridden)

- **Environment / project specific values**  
  → **Only** in `inventory/{env}/group_vars/` and `host_vars/`

**Allowed changes inside `roles/`**

- Change role **logic** (tasks, handlers, templates, conditionals)
- Add / remove **default variables** (in `defaults/main.yml`)
- Fix bugs or improve idempotency

**Forbidden / bad practice**

- Hardcoding environment-specific values (URLs, IPs, credentials, versions, etc.) inside roles
- Creating separate roles per environment

**Why this matters:**

- Keeps roles **pure** and reusable across environments/projects
- Prevents merge conflicts when updating roles
- Follows Ansible precedence: inventory vars > role vars > role defaults
- Makes auditing & compliance easier (config separated from logic)

## 7. Recommended Execution Flow

```bash
# 1. Activate environment
source .venv/bin/activate

# 2. Syntax check
ansible-playbook --syntax-check -i inventory/production/inventory.ini playbooks/os_hardening.yml

# 3. Dry-run + diff (strongly recommended!)
ansible-playbook -i inventory/production/inventory.ini playbooks/os_hardening.yml --check --diff

# 4. Apply
ansible-playbook -i inventory/production/inventory.ini playbooks/os_hardening.yml --ask-vault-pass

# Optional verbosity levels: -v  -vv  -vvv
```

Typical order example (dependencies):

```text
bootstrap.yml → hardening.yml → docker.yml
              → traefik.yml → gitlab.yml → runner.yml → observability.yml
```

## 8. Quick Reference – Main Playbooks

| Playbook            | Purpose                             |
| ------------------- | ----------------------------------- |
| `bootstrap.yml`     | Prepare target nodes                |
| `os_hardening.yml`  | OS security baseline                |
| `ssh_hardening.yml` | SSH daemon + banner + key-only auth |
| `docker.yml`        | Docker + docker-compose plugin      |
| `traefik.yml`       | Traefik reverse proxy               |
| `gitlab.yml`        | GitLab CE/EE via docker compose     |
| `runner.yml`        | GitLab Runner                       |
| `nexus.yml`         | Sonatype Nexus Repository OSS       |
| `elastic.yml`       | Elasticsearch                       |
| `kibana.yml`        | Kibana                              |
| `observability.yml` | Observability components            |

## 9. Contributing

- Use meaningful commit messages
- Add new roles under `roles/`
- Document new variables in `defaults/main.yml` with comments
- Keep roles idempotent and OS-agnostic where possible
- Test roles in check mode before merging

## 10. See Also

- [Ansible Best Practices (official docs)](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [Variable Precedence](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#ansible-variable-precedence)
