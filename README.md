# Inventory

## Requirements (Client)

1. Install Ansible
```bash
brew install ansible
```

2. Install sshpass
```bash
brew install sshpass
```

## Initial Setup

The initial setup does the following:
- Harden the server
- Implement the Grafana Loki Cloud logging
- Add the central user `adminuser`

1. Add the server to the `inventory.ini` file, directly below `[new_servers]`.
```yml
[new_servers]
<servername> ansible_host=<ip> ansible_user=root
```

2. Run the initialization script on the client.
```bash
ansible-playbook -i inventory.ini initial-setup.yml \
  --tags "cicd" \
  --private-key ~/.ssh/id_rsa_adminuser_main-server \
  --ask-vault-pass
  --ask-become-pass
  --become-method su
```

3. To complete the step, please adapt the `inventory.ini`:<br>
Put the initialized server directly under `[running_servers]` and adapt the line:

```yml
[running_servers]
<server name> ansible_host=<ip> ansible_port=3344 ansible_user=adminuser ansible_ssh_private_key_file=<path>
```

## Install Docker

`inventory.ini`
```yml
[docker_servers]
<server name> ansible_host=<ip> ansible_port=3344 ansible_user=adminuser ansible_ssh_private_key_file=<path>
```

`bash`
```bash
ansible-playbook -i inventory.ini install-docker.yml --ask-become-pass --become-method su
```

## Install Traefik
`inventory.ini`
```yml
[traefik_servers]
<server name> ansible_host=<ip> ansible_port=3344 ansible_user=adminuser ansible_ssh_private_key_file=<path>
```

`bash`
```bash
ansible-playbook -i inventory.ini install-traefik.yml \
  --private-key ~/.ssh/id_rsa_adminuser_main-server
```

## Automated Deployment

Deployments are triggered automatically by a push to the `main` branch of the respective application's repository (e.g., `pjurke/philipjurke`).

The process is managed by a GitHub Actions workflow (`.github/workflows/deploy.yml` in the application repository). It performs the following steps:

1. Builds a multi-architecture Docker image (`linux/amd64` and `linux/arm64`).
2. Pushes the image to the GitHub Container Registry.
3. Triggers this repository's `deploy-app.yml` playbook to deploy the new image to both the staging and production environments.

`bash`
```bash
ansible-playbook -i inventory.ini deploy-app.yml \
  --extra-vars "app_name=philipjurke env=staging image_tag=your_image_tag_here" \
  --ask-vault-pass

ansible-playbook -i inventory.ini deploy-app.yml \
  --extra-vars "app_name=philipjurke env=staging" \
  --private-key ~/.ssh/id_rsa_adminuser_main-server \
  --ask-vault-pass
```

## Self-Hosted GitHub Runner `not used`
This setup deploys a self-hosted, ARM64-compatible GitHub Actions runner inside a Podman container. Each runner is tied to a specific repository.

1. Add the server to the inventory

```ini
[github_runner_servers]
<server name> ansible_host=<ip> ansible_port=3344 ansible_user=adminuser ansible_ssh_private_key_file=<path>
```

2. Run the playbook

**For Organizations**
The PAT needs the scope "admin:org".
```bash
ansible-playbook -i inventory.ini deploy-runner.yml \
  --private-key ~/.ssh/id_rsa_adminuser_main-server \
  --extra-vars "github_owner=Text-Review" \
  --extra-vars "github_pat=pat"
```

**For Repos**
The PAT needs the scope "repo".
```bash
ansible-playbook -i inventory.ini deploy-runner.yml \
  --private-key ~/.ssh/id_rsa_adminuser_main-server \
  --extra-vars "github_owner=PJurke" \
  --extra-vars "github_repository=philipjurke" \
  --extra-vars "github_pat=pat"
```