# SSH Tailscale Action Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a common GitHub Actions workflow that SSHs into a server through Tailscale.

**Architecture:** A single workflow file exposes both `workflow_call` and `workflow_dispatch`. It joins the tailnet first, then runs a remote command over SSH using repository or caller-provided secrets.

**Tech Stack:** GitHub Actions YAML, `tailscale/github-action@v4`, `appleboy/ssh-action`.

---

### Task 1: Add Workflow

**Files:**
- Create: `.github/workflows/ssh-tailscale.yml`

- [ ] **Step 1: Create workflow file**

```yaml
name: SSH via Tailscale

on:
  workflow_call:
    inputs:
      command:
        description: Command to run on the remote server.
        required: false
        type: string
        default: hostname && whoami && pwd
    secrets:
      TAILSCALE_AUTHKEY:
        required: true
      SSH_HOST:
        required: true
      SSH_PORT:
        required: true
      SSH_USER:
        required: true
      SSH_PASS:
        required: true
  workflow_dispatch:
    inputs:
      command:
        description: Command to run on the remote server.
        required: false
        type: string
        default: hostname && whoami && pwd

jobs:
  ssh:
    name: SSH via Tailscale
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Connect to Tailscale
        uses: tailscale/github-action@v4
        with:
          authkey: ${{ secrets.TAILSCALE_AUTHKEY }}

      - name: Run remote command
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          port: ${{ secrets.SSH_PORT }}
          username: ${{ secrets.SSH_USER }}
          password: ${{ secrets.SSH_PASS }}
          script_stop: true
          script: ${{ inputs.command }}
```

- [ ] **Step 2: Verify YAML parses**

Run: `ruby -e 'require "yaml"; YAML.load_file(".github/workflows/ssh-tailscale.yml"); puts "yaml ok"'`

Expected: `yaml ok`

- [ ] **Step 3: Verify triggers exist**

Run: `rg -n "workflow_call|workflow_dispatch|tailscale/github-action@v4|appleboy/ssh-action" .github/workflows/ssh-tailscale.yml`

Expected: output includes all four patterns.
