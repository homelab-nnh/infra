# Simplify Nginx Gateway Workflow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Simplify the nginx gateway workflow so it uses normal workflow steps instead of generating a large remote script.

**Architecture:** `setup-nginx-gateway.yml` will become one Ubuntu job. It checks out the repo, uses the common SSH Tailscale action to install nginx when missing, uses `appleboy/scp-action` to upload the local `nginx/` directory, then uses the common SSH action again to copy configs into `/etc/nginx`, validate, enable, and reload nginx.

**Tech Stack:** GitHub Actions YAML, local composite action `.github/actions/ssh-tailscale`, `appleboy/scp-action@v0.1.7`, shell on `ubuntu-latest`.

---

### Task 1: Replace Generated Script Workflow

**Files:**
- Modify: `.github/workflows/setup-nginx-gateway.yml`

- [ ] **Step 1: Replace the two-job prepare/setup workflow with one step-based job**

Use this workflow:

```yaml
name: Setup Nginx Gateway

on:
  workflow_dispatch:

jobs:
  setup:
    name: Setup nginx gateway
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Ensure nginx is installed
        uses: ./.github/actions/ssh-tailscale
        with:
          command: |
            set -euo pipefail

            if ! command -v nginx >/dev/null 2>&1; then
              sudo apt-get update
              sudo DEBIAN_FRONTEND=noninteractive apt-get install -y nginx
            fi
          tailscale_authkey: ${{ secrets.TAILSCALE_AUTHKEY }}
          ssh_host: ${{ secrets.SSH_HOST }}
          ssh_port: ${{ secrets.SSH_PORT }}
          ssh_user: ${{ secrets.SSH_USER }}
          ssh_pass: ${{ secrets.SSH_PASS }}

      - name: Upload nginx config
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SSH_HOST }}
          port: ${{ secrets.SSH_PORT }}
          username: ${{ secrets.SSH_USER }}
          password: ${{ secrets.SSH_PASS }}
          source: nginx
          target: /tmp/nginx-gateway
          rm: true

      - name: Apply nginx config
        uses: ./.github/actions/ssh-tailscale
        with:
          command: |
            set -euo pipefail

            CONFIG_ROOT=/tmp/nginx-gateway/nginx

            if [ ! -d "$CONFIG_ROOT" ]; then
              echo "Missing uploaded nginx config directory: $CONFIG_ROOT"
              exit 1
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
          tailscale_authkey: ${{ secrets.TAILSCALE_AUTHKEY }}
          ssh_host: ${{ secrets.SSH_HOST }}
          ssh_port: ${{ secrets.SSH_PORT }}
          ssh_user: ${{ secrets.SSH_USER }}
          ssh_pass: ${{ secrets.SSH_PASS }}
          connect_tailscale: "false"
```

### Task 2: Update Documentation

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Rewrite the nginx gateway description**

Update the Nginx Gateway section so it says the workflow:

```markdown
This workflow runs directly from this repo. It checks whether nginx is installed on the remote server, installs it when missing, uploads the local `nginx/` directory, copies config into `/etc/nginx`, validates nginx, and reloads the service.
```

### Task 3: Verify

**Files:**
- No source changes.

- [ ] **Step 1: Parse GitHub YAML files**

Run:

```bash
python3 -c 'import yaml, glob; files=glob.glob(".github/**/*.yml", recursive=True)+glob.glob(".github/**/*.yaml", recursive=True); [print(f) or yaml.safe_load(open(f)) for f in files]'
```

Expected: command exits 0 and prints all GitHub YAML files.

- [ ] **Step 2: Review workflow**

Run:

```bash
sed -n '1,220p' .github/workflows/setup-nginx-gateway.yml
```

Expected: workflow has one `setup` job and no base64/tar command-generation step.
