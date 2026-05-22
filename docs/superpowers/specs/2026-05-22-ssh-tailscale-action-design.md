# SSH Tailscale GitHub Action Design

## Goal

Create a common GitHub Actions workflow that can SSH into a server through Tailscale.

## Triggers

The workflow supports both reusable and manual execution:

- `workflow_call` lets other repositories call the workflow.
- `workflow_dispatch` lets a user run the workflow manually from GitHub.

## Inputs And Secrets

The workflow accepts a `command` input. If omitted, it runs a small connection check:

```bash
hostname && whoami && pwd
```

The workflow reads these GitHub secrets:

- `TAILSCALE_AUTHKEY`
- `SSH_HOST`
- `SSH_PORT`
- `SSH_USER`
- `SSH_PASS`

## Execution Flow

1. Connect the GitHub-hosted runner to the tailnet with `tailscale/github-action`.
2. SSH to the target host with `appleboy/ssh-action`.
3. Run the provided command on the remote server.

## Error Handling

The workflow fails if Tailscale authentication fails, SSH connection fails, or the remote command exits non-zero.

## Verification

Static verification checks that workflow YAML parses and that the workflow contains both trigger modes.
