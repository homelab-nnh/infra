# Nginx Gateway Action Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a direct GitHub Actions workflow that applies nginx config from this repo to a gateway server through the existing Tailscale SSH workflow.

**Architecture:** A new workflow runs directly with `workflow_dispatch`. It checks out the repo, packages the local `nginx/` directory into a generated remote command, then calls `.github/workflows/ssh-tailscale.yml` to run that command on the server.

**Tech Stack:** GitHub Actions YAML, existing Tailscale SSH workflow, nginx, bash.

---

### Task 1: Add Nginx Gateway Workflow

**Files:**
- Create: `.github/workflows/setup-nginx-gateway.yml`
- Create: `nginx/sites-available/gateway.conf`

- [ ] **Step 1: Create workflow file**

```yaml
name: Setup Nginx Gateway

on:
  workflow_dispatch:

jobs:
  prepare:
    name: Prepare nginx config
    runs-on: ubuntu-latest
    timeout-minutes: 10
    outputs:
      command: ${{ steps.command.outputs.command }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build remote command
        id: command
        run: |
          set -euo pipefail

          tar -C nginx -czf nginx-config.tar.gz .
          payload="$(base64 -w 0 nginx-config.tar.gz)"

          {
            echo "command<<REMOTE_SCRIPT"
            cat <<'REMOTE_SCRIPT'
            set -euo pipefail

            CONFIG_ROOT=/tmp/nginx-gateway/nginx
            rm -rf /tmp/nginx-gateway
            mkdir -p "$CONFIG_ROOT"

            cat >/tmp/nginx-gateway/nginx-config.tar.gz.b64 <<'NGINX_PAYLOAD'
          REMOTE_SCRIPT
            echo "$payload"
            cat <<'REMOTE_SCRIPT'
          NGINX_PAYLOAD
            base64 -d /tmp/nginx-gateway/nginx-config.tar.gz.b64 > /tmp/nginx-gateway/nginx-config.tar.gz
            tar -xzf /tmp/nginx-gateway/nginx-config.tar.gz -C "$CONFIG_ROOT"

            if [ ! -d "$CONFIG_ROOT" ]; then
              echo "Missing copied nginx config directory: $CONFIG_ROOT"
              exit 1
            fi

            if ! command -v nginx >/dev/null 2>&1; then
              sudo apt-get update
              sudo DEBIAN_FRONTEND=noninteractive apt-get install -y nginx
            fi

            sudo mkdir -p /etc/nginx/sites-available /etc/nginx/sites-enabled /etc/nginx/conf.d

            if [ -d "$CONFIG_ROOT/conf.d" ]; then
              sudo cp -R "$CONFIG_ROOT/conf.d/." /etc/nginx/conf.d/
            fi

            if [ -d "$CONFIG_ROOT/sites-available" ]; then
              sudo cp -R "$CONFIG_ROOT/sites-available/." /etc/nginx/sites-available/

              for site_path in "$CONFIG_ROOT"/sites-available/*; do
                [ -f "$site_path" ] || continue
                site_name="$(basename "$site_path")"
                sudo ln -sfn "/etc/nginx/sites-available/$site_name" "/etc/nginx/sites-enabled/$site_name"
              done
            fi

            sudo nginx -t
            sudo systemctl enable nginx
            sudo systemctl reload nginx
          REMOTE_SCRIPT
            echo "REMOTE_SCRIPT"
          } >> "$GITHUB_OUTPUT"

  setup:
    name: Setup nginx gateway
    needs: prepare
    uses: ./.github/workflows/ssh-tailscale.yml
    with:
      command: ${{ needs.prepare.outputs.command }}
    secrets:
      TAILSCALE_AUTHKEY: ${{ secrets.TAILSCALE_AUTHKEY }}
      SSH_HOST: ${{ secrets.SSH_HOST }}
      SSH_PORT: ${{ secrets.SSH_PORT }}
      SSH_USER: ${{ secrets.SSH_USER }}
      SSH_PASS: ${{ secrets.SSH_PASS }}
```

- [ ] **Step 2: Create default gateway config**

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name _;

    client_max_body_size 100m;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

- [ ] **Step 3: Verify YAML parses**

Run: `python3 -c 'import yaml; yaml.load(open(".github/workflows/setup-nginx-gateway.yml"), Loader=yaml.BaseLoader); print("yaml ok")'`

Expected: `yaml ok`

- [ ] **Step 4: Verify workflow content**

Run: `rg -n "workflow_dispatch|actions/checkout|ssh-tailscale.yml|base64 -d|apt-get install -y nginx|nginx -t|systemctl reload nginx" .github/workflows/setup-nginx-gateway.yml`

Expected: output includes all seven patterns.

### Task 2: Document Usage

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add README section**

Add a section named `Nginx Gateway` that lists the workflow path, config directory behavior, required secrets, and manual run steps.

- [ ] **Step 2: Verify documentation mentions workflow**

Run: `rg -n "Nginx Gateway|setup-nginx-gateway.yml|nginx/sites-available|nginx/conf.d" README.md`

Expected: output includes all four patterns.
