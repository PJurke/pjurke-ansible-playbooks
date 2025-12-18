# Infrastructure Playbooks

This repository manages server infrastructure using Ansible, focusing on
- security hardening,
- containerization (Podman & Quadlets),
- and observability (Grafana Alloy).

## Requirements

- Ansible (installed locally)
- Ansible Vault Password (for `secrets/`)
- SSH Access to the target servers

## Workflow

Server provisioning follows three logical stages:

### 1. Bootstrap (Initialization)

Prepares a fresh server (with root access) for management by Ansible.

- Playbook: `bootstrap.yml`
- Target: `[bootstrap]` group in `inventory.ini`
- Actions:
  - Installs Python 3 & Sudo.
  - Creates the administrative user `admin-user`.
  - Deploys the SSH key for `admin-user`.

```bash
ansible-playbook -i inventory.ini bootstrap.yml -k
```

### 2. Setup (Base Configuration)

Applies the baseline configuration to the server. This is the main maintenance playbook.

- Playbook: `setup.yml`
- Target: `[running_servers] group`
- Roles:
  - `common`: Packages, Timezone (Europe/Berlin), ACLs, Unattended Upgrades.
  - `security`: SSH Port to 3344, UFW Firewall, Fail2Ban, SSH Hardening.
  - `podman`: Installation and Sysctl adjustments (Port 80 for Rootless).
  - `monitoring`: Grafana Alloy Agent (Logs & Metrics).

```bash
ansible-playbook -i inventory.ini setup.yml --ask-vault-pass
```

### 3. Services & Routing

Deploys the reverse proxy infrastructure and specific services.

- Traefik Proxy:
  - Playbook: `install-traefik.yml`
  - Function: Sets up Traefik v3 as a Quadlet service, configures Entrypoints (80/443) and ACME (LetsEncrypt).

- PhilipJurke Website:
  - Playbook: `deploy-philipjurke.yml`
  - Function: Deploys the website container and the Traefik router configuration.

```bash
# Install Traefik
ansible-playbook -i inventory.ini install-traefik.yml --ask-vault-pass

# Deploy Website
ansible-playbook -i inventory.ini deploy-philipjurke.yml --ask-vault-pass

# Deploy Text Review
ansible-playbook -i inventory.ini deploy-textreview.yml --extra-vars "env=staging" --ask-vault-pass
```

## Secrets Management

Sensitive data is encrypted with Ansible Vault. The files are located in the `secrets/` directory and loaded automatically by the playbooks.

- `secrets/loki-secrets.yml`: Credentials for Logging/Monitoring.
- `secrets/philipjurke/vault.yml`: App-specific secrets (e.g., GHCR Token).

## Structure & Roles

| Role	      | Description                                       |
|-------------|---------------------------------------------------|
| bootstrap	  | Initial user & SSH setup.                         |
| common	    | Basic system tools, ACLs, and updates.            |
| security	  | Firewall (UFW), Fail2Ban config, SSH hardening.   |
| podman	    | Container engine setup.                           |
| monitoring	| Grafana Alloy setup (Loki/Prometheus Exporter).   |
| traefik	    | Reverse Proxy setup as Systemd Service (Quadlet). |
| philipjurke	| Deployment of the personal website.               |
| textreview	| Deployment of the Text Review web app.            |