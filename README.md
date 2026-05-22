# Homelab Infra Actions

Common GitHub Actions workflows for homelab automation.

## SSH via Tailscale

Workflow: `.github/workflows/ssh-tailscale.yml`

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
