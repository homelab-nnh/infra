# Nginx Gateway GitHub Action Design

## Goal

Create a direct GitHub Actions workflow that packages nginx config from this repo and applies it to an Ubuntu or Debian gateway server through the existing Tailscale SSH workflow.

## Triggers

The workflow supports manual execution with `workflow_dispatch`.

## Inputs And Secrets

The workflow reads nginx config files from the repo:

- `nginx/sites-available/*.conf` is copied to `/etc/nginx/sites-available/`.
- Each copied site file is symlinked into `/etc/nginx/sites-enabled/`.
- `nginx/conf.d/` is copied to `/etc/nginx/conf.d/` when present.

The workflow reads these GitHub secrets:

- `TAILSCALE_AUTHKEY`
- `SSH_HOST`
- `SSH_PORT`
- `SSH_USER`
- `SSH_PASS`

## Execution Flow

1. Check out this repo on the GitHub-hosted runner.
2. Connect the runner to the tailnet with `tailscale/github-action`.
3. Package the local `nginx/` directory as a base64 tar payload in a generated remote command.
4. Call `.github/workflows/ssh-tailscale.yml` with the generated command.
5. On the remote server, decode the payload into `/tmp/nginx-gateway/nginx`.
6. Install nginx if it is missing.
7. Copy repo config into `/etc/nginx`.
8. Symlink files from `/etc/nginx/sites-available` into `/etc/nginx/sites-enabled`.
9. Run `nginx -t`.
10. Reload nginx through `systemctl`.

## Error Handling

The remote script uses `set -euo pipefail`, validates that the decoded config directory exists, fails if nginx config validation fails, and relies on the shared SSH workflow to fail when Tailscale or SSH fails.

## Verification

Static verification checks that workflow YAML parses and that the workflow contains manual trigger, local SSH workflow call, nginx payload packaging, nginx install, `nginx -t`, and reload steps.
