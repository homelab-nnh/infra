# Common SSH Tailscale Action Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a common composite action that lets workflows run independent SSH commands through Tailscale.

**Architecture:** Create `.github/actions/ssh-tailscale/action.yml` as the single implementation for Tailscale connection, SSH reachability, and remote command execution. Keep `.github/workflows/ssh-tailscale.yml` as a reusable/manual wrapper that delegates to the composite action.

**Tech Stack:** GitHub Actions YAML, `tailscale/github-action@v4`, `appleboy/ssh-action@v1`, shell on `ubuntu-latest`.

---

### Task 1: Add Composite Action

**Files:**
- Create: `.github/actions/ssh-tailscale/action.yml`

- [ ] **Step 1: Create the composite action**

Create `.github/actions/ssh-tailscale/action.yml` with:

```yaml
name: SSH via Tailscale
description: Connect an Ubuntu runner to Tailscale and run a remote SSH command.

inputs:
  command:
    description: Command to run on the remote server.
    required: true
  tailscale_authkey:
    description: Tailscale auth key.
    required: true
  ssh_host:
    description: SSH host.
    required: true
  ssh_port:
    description: SSH port.
    required: true
  ssh_user:
    description: SSH username.
    required: true
  ssh_pass:
    description: SSH password.
    required: true

runs:
  using: composite
  steps:
    - name: Connect to Tailscale
      uses: tailscale/github-action@v4
      with:
        authkey: ${{ inputs.tailscale_authkey }}

    - name: Check SSH reachability
      shell: bash
      run: |
        set -euo pipefail

        tailscale status
        timeout 10 bash -c 'cat < /dev/null > /dev/tcp/${{ inputs.ssh_host }}/${{ inputs.ssh_port }}'

    - name: Run remote command
      uses: appleboy/ssh-action@v1
      with:
        host: ${{ inputs.ssh_host }}
        port: ${{ inputs.ssh_port }}
        username: ${{ inputs.ssh_user }}
        password: ${{ inputs.ssh_pass }}
        script: ${{ inputs.command }}
```

### Task 2: Delegate Existing Workflow To The Action

**Files:**
- Modify: `.github/workflows/ssh-tailscale.yml`

- [ ] **Step 1: Replace duplicated job steps with the local action**

Update the `jobs.ssh.steps` section to:

```yaml
    steps:
      - name: SSH via Tailscale
        uses: ./.github/actions/ssh-tailscale
        with:
          command: ${{ inputs.command }}
          tailscale_authkey: ${{ secrets.TAILSCALE_AUTHKEY }}
          ssh_host: ${{ secrets.SSH_HOST }}
          ssh_port: ${{ secrets.SSH_PORT }}
          ssh_user: ${{ secrets.SSH_USER }}
          ssh_pass: ${{ secrets.SSH_PASS }}
```

### Task 3: Update Direct Test Workflow

**Files:**
- Modify: `.github/workflows/test-ssh-tailscale.yml`

- [ ] **Step 1: Convert the test workflow to direct action usage**

Replace the reusable workflow job with a normal Ubuntu job:

```yaml
jobs:
  echo:
    name: Echo on remote server
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Echo via SSH
        uses: ./.github/actions/ssh-tailscale
        with:
          command: |
            echo "hello from github actions via tailscale"
            echo "hostname: $(hostname)"
            echo "user: $(whoami)"
            echo "current directory: $(pwd)"
            echo "hello from github actions via tailscale" >> /tmp/test-nnh.log
          tailscale_authkey: ${{ secrets.TAILSCALE_AUTHKEY }}
          ssh_host: ${{ secrets.SSH_HOST }}
          ssh_port: ${{ secrets.SSH_PORT }}
          ssh_user: ${{ secrets.SSH_USER }}
          ssh_pass: ${{ secrets.SSH_PASS }}
```

### Task 4: Update README

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Document composite action usage**

Add a section showing step-level usage:

```markdown
## SSH via Tailscale Action

Action: `.github/actions/ssh-tailscale`

Use this when a workflow needs to run one or more independent remote SSH steps through Tailscale.

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
```
```

### Task 5: Verify

**Files:**
- No source changes.

- [ ] **Step 1: Parse YAML files**

Run:

```bash
ruby -e 'require "yaml"; Dir[".github/**/*.yml", ".github/**/*.yaml"].each { |f| YAML.load_file(f); puts f }'
```

Expected: command exits 0 and prints the workflow/action YAML files.

- [ ] **Step 2: Review git diff**

Run:

```bash
git diff -- .github README.md docs/superpowers
```

Expected: diff only contains the composite action, workflow delegation, test workflow update, README docs, and these planning docs.
