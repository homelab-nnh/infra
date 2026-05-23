# Common SSH Tailscale Action Design

## Goal

Create a reusable GitHub composite action that lets any workflow run remote shell commands on the homelab server through Tailscale.

## User Experience

Workflows should be able to run independent remote commands as normal steps:

```yaml
- name: Check remote host
  uses: ./.github/actions/ssh-tailscale
  with:
    command: hostname
    tailscale_authkey: ${{ secrets.TAILSCALE_AUTHKEY }}
    ssh_host: ${{ secrets.SSH_HOST }}
    ssh_port: ${{ secrets.SSH_PORT }}
    ssh_user: ${{ secrets.SSH_USER }}
    ssh_pass: ${{ secrets.SSH_PASS }}
```

The caller should not need to create a whole script file when a few commands are enough.

## Architecture

Add `.github/actions/ssh-tailscale/action.yml` as a composite action. The action connects the current Ubuntu runner to Tailscale, checks SSH reachability, then runs the provided command through `appleboy/ssh-action`.

Keep `.github/workflows/ssh-tailscale.yml` as a compatibility wrapper for manual dispatch and workflow-call users. Internally, it should call the composite action so the SSH logic has one implementation.

## Inputs

The action accepts:

- `command`: remote shell script to execute.
- `tailscale_authkey`: Tailscale auth key.
- `ssh_host`: target host or Tailscale IP/name.
- `ssh_port`: SSH port.
- `ssh_user`: SSH username.
- `ssh_pass`: SSH password.
- `connect_tailscale`: optional, defaults to `true`; set to `false` when the current job already connected to Tailscale.

## Error Handling

The action fails if Tailscale authentication fails, the SSH port is unreachable within the timeout, or the remote command exits non-zero.

## Documentation

Update `README.md` to show the composite action usage for step-by-step remote commands and keep the reusable workflow example for compatibility.

## Verification

Static verification should confirm all YAML files parse. The composite action should be checked by updating the existing `test-ssh-tailscale.yml` workflow to call it directly.
