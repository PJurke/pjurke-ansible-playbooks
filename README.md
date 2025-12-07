# PJurke Infrastructure Ansible Playbooks
This repository contains the central Ansible playbooks and roles used to manage my server infrastructure.

## Concept
All reusable logic (tasks, templates, handlers) is extracted into individual Ansible Roles.

This repository defines:

1. **Playbooks:** The orchestration files (e.g., `bootstrap.yml`).
2. **Inventory:** The list of servers (`inventory.ini`).
3. **Secrets:** All encrypted secrets (via Ansible Vault) in the `secrets/` directory.
4. **Dependencies:** The `requirements.yml`, which defines which external roles and collections are needed.

## Requirements (Client)

1. **Ansible** must be installed locally.
```bash
brew install ansible
```

2. **sshpass** must be installed locally.
```bash
brew install sshpass
```

3. **Git** must be installed (to clone the roles).

## First-Time Client Setup
Before you can run any playbooks, you must install all external roles and collections using `ansible-galaxy`:

```bash
ansible-galaxy install -r requirements.yml
```

This command reads the `requirements.yml` file and downloads all dependencies into your local Ansible path.

## Secrets Management
All secrets are managed using Ansible Vault and are stored in the secrets/ directory. The structure is:

```
secrets/
├── loki-secrets.yml
├── philipjurke/
│   └── vault.yml
└── textreview/
    ├── vault.yml
    ├── production-vault.yml
    └── staging-vault.yml
```

The playbooks (e.g., `deploy-app.yml`) load these files automatically. You will be prompted for the password via `--ask-vault-pass` when running a command.

## Server Management
### 1. Bootstrap
   1. Add the new server (with `root` access and IP) to the `inventory.ini` under the `[bootstrap]` group:
   ```ini
   [bootstrap]
   new-server-name ansible_host=1.2.3.4 ansible_user=root
   ```

   2. Run the `initial-setup.yml` playbook. It installs basic security (ufw, fail2ban), changes the SSH port, creates the `adminuser` and `deployeruser`, installs Podman, and sets up monitoring.
   ```bash
   # Run this as "root", using "su" as the become method
   ansible-playbook -i inventory.ini initial-setup.yml --limit new-server-name \
   --private-key ~/.ssh/id_rsa_adminuser_main-server \
   --ask-vault-pass \
   --ask-become-pass \
   --become-method su
   ```

   3. IMPORTANT: After the run succeeds, update the `inventory.ini`:
   - Move the server from `[new_servers]` to `[running_servers]` (and other relevant groups).
   - Update the entry to use the new port (`3344`) and the `adminuser`:
   ```ini
   [running_servers]
   main-server ansible_host=1.2.3.4 ansible_port=3344 ansible_user=adminuser
   ```

### 2. Install Traefik Reverse Proxy
Run this playbook to deploy Traefik to any server in the `[traefik_servers]` group.
```bash
ansible-playbook -i inventory.ini install-traefik.yml \
  --private-key ~/.ssh/id_rsa_adminuser_main-server \
  --become-method sudo
```

### 3. Deploy Applications (Staging/Production)
The `deploy-app.yml` playbook is used to deploy applications like `philipjurke` or `textreview`. Target servers are controlled by groups in `inventory.ini` (e.g., `[philipjurke_servers]`).

The `app_name` and `env` variables control what is deployed where.

```bash
# Example: Deploy 'textreview' to 'staging'
ansible-playbook -i inventory.ini deploy-app.yml \
  --extra-vars "app_name=textreview env=staging" \
  --private-key ~/.ssh/id_rsa_adminuser_main-server \
  --become-method sudo \
  --ask-vault-pass
```

```bash
# Example: Deploy 'philipjurke' to 'production' (image_tag is optional)
ansible-playbook -i inventory.ini deploy-app.yml \
  --extra-vars "app_name=philipjurke env=production image_tag=1.2.3" \
  --private-key ~/.ssh/id_rsa_adminuser_main-server \
  --become-method sudo \
  --ask-vault-pass
```




## Self-Hosted GitHub Runner `not used`
This playbook sets up a runner. Configuration (owner, repo, PAT) must be passed as `extra-vars`.

**For a Repository (PAT needs `repo` scope):**
```bash
ansible-playbook -i inventory.ini deploy-runner.yml \
  --private-key ~/.ssh/id_rsa_adminuser_main-server \
  --become-method sudo \
  --extra-vars "github_owner=PJurke" \
  --extra-vars "github_repository=philipjurke" \
  --extra-vars "github_pat=your_pat_here"
```

**For an Organization (PAT needs admin:org scope):**
```bash
ansible-playbook -i inventory.ini deploy-runner.yml \
  --private-key ~/.ssh/id_rsa_adminuser_main-server \
  --become-method sudo \
  --extra-vars "github_owner=text-review" \
  --extra-vars "github_pat=your_pat_here"
```