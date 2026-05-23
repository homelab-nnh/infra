# Homelab Infra Actions

Common GitHub Actions workflows for homelab automation.

## SSH via Tailscale Action

Action: `.github/actions/ssh-tailscale`

Use this when a workflow needs to run one or more independent remote SSH steps through Tailscale.

Required secrets:

- `TAILSCALE_AUTHKEY`
- `SSH_HOST`
- `SSH_PORT`
- `SSH_USER`
- `SSH_PASS`

Example:

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v4

  - name: Check remote host
    uses: homelab-nnh/infra/.github/actions/ssh-tailscale@main
    with:
      command: hostname
      tailscale_authkey: ${{ secrets.TAILSCALE_AUTHKEY }}
      ssh_host: ${{ secrets.SSH_HOST }}
      ssh_port: ${{ secrets.SSH_PORT }}
      ssh_user: ${{ secrets.SSH_USER }}
      ssh_pass: ${{ secrets.SSH_PASS }}

  - name: Reload nginx
    uses: homelab-nnh/infra/.github/actions/ssh-tailscale@main
    with:
      command: |
        sudo nginx -t
        sudo systemctl reload nginx
      tailscale_authkey: ${{ secrets.TAILSCALE_AUTHKEY }}
      ssh_host: ${{ secrets.SSH_HOST }}
      ssh_port: ${{ secrets.SSH_PORT }}
      ssh_user: ${{ secrets.SSH_USER }}
      ssh_pass: ${{ secrets.SSH_PASS }}
```

## SSH via Tailscale Reusable Workflow

Workflow: `.github/workflows/ssh-tailscale.yml`

This remains available for manual dispatch or workflows that prefer one reusable SSH job.

Required secrets:

- `TAILSCALE_AUTHKEY`
- `SSH_HOST`
- `SSH_PORT`
- `SSH_USER`
- `SSH_PASS`

Manual run:

1. Open the workflow in GitHub Actions.
2. Choose **Run workflow**.
3. Enter the remote command to run, or keep the default connection check.

Reusable workflow example:

```yaml
name: Deploy

on:
  workflow_dispatch:

jobs:
  deploy:
    uses: homelab-nnh/infra/.github/workflows/ssh-tailscale.yml@main
    with:
      command: |
        hostname
        whoami
        pwd
    secrets:
      TAILSCALE_AUTHKEY: ${{ secrets.TAILSCALE_AUTHKEY }}
      SSH_HOST: ${{ secrets.SSH_HOST }}
      SSH_PORT: ${{ secrets.SSH_PORT }}
      SSH_USER: ${{ secrets.SSH_USER }}
      SSH_PASS: ${{ secrets.SSH_PASS }}
```

## Nginx Gateway

Workflow: `.github/workflows/setup-nginx-gateway.yml`

This workflow runs directly from this repo. It checks whether nginx is installed on the remote server, installs it when missing, uploads the local `nginx/` directory, copies config into `/etc/nginx`, validates nginx, and reloads the service.

Config:

- `nginx/sites-available/*.conf` is copied to `/etc/nginx/sites-available/`.
- Each file in `nginx/sites-available/` is symlinked into `/etc/nginx/sites-enabled/`.
- `nginx/conf.d/` is copied to `/etc/nginx/conf.d/` when the directory exists.

Required secrets:

- `TAILSCALE_AUTHKEY`
- `SSH_HOST`
- `SSH_PORT`
- `SSH_USER`
- `SSH_PASS`

Manual run:

1. Open the workflow in GitHub Actions.
2. Choose **Run workflow**.
3. Run the workflow.
