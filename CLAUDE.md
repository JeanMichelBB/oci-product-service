# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Run playbook locally (uses ansible/inventory.ini with static IPs + SSH keys)
ansible-playbook -i ansible/inventory.ini ansible/playbook.yml \
  --extra-vars "tailscale_auth_key=<YOUR_KEY>"

# Ping all hosts to verify connectivity
ansible k3s -i ansible/inventory.ini -m ping
```

The CI/CD pipeline runs the playbook automatically on push to `main` when files under `ansible/*` change.

## Cluster Layout (3 machines)

| Host | Group | User | SSH Key |
|------|-------|------|---------|
| `sacenpapier.org` | `master` | `jmbberube` | `~/.ssh/id_ed25519_elitedesk` |
| OCI instance 1 | `workers` | `ubuntu` | `~/.ssh/sacenplastique.key` |
| OCI instance 2 | `workers` | `ubuntu` | `~/.ssh/sacenplastique.key` |

## Inventory Flow

- `ansible/inventory_template.ini` — source template with `__OCI_WORKERS__` placeholder
- CI generates `ansible/inventory.ini` at runtime by replacing the placeholder with live OCI instance IPs
- Local development uses a static `ansible/inventory.ini` with hardcoded IPs

## Playbook Execution Order (`ansible/playbook.yml`)

1. **System prerequisites** — installs curl/gnupg/iptables, loads `br_netfilter` + `overlay` kernel modules, applies sysctl settings for k3s networking
2. **Tailscale** — adds apt repo, installs `tailscale`, connects to Tailscale network (idempotent: only runs `tailscale up` if not already connected)
3. **k3s server (master)** — installs k3s server on `sacenpapier.org` bound to its Tailscale IP, waits for node token
4. **k3s agents (workers)** — installs k3s agent on both OCI instances, joining via the master's Tailscale IP and token

## CI/CD Pipeline (`.github/workflows/main.yaml`)

Trigger: push to `main` with changes in `ansible/*`

Steps:
1. Configure OCI CLI from secrets
2. Poll all instances in compartment until `RUNNING`
3. Collect public IPs → `ansible/hosts.tmp`
4. Update Cloudflare DNS A record for the first instance IP (only if changed)
5. Build `ansible/inventory.ini` from template
6. Configure SSH keys (workers + master)
7. Run playbook via SSH

## Required Secrets

| Secret | Purpose |
|--------|---------|
| `OCI_USER_OCID`, `OCI_TENANCY_OCID`, `OCI_REGION`, `OCI_KEY_FINGERPRINT`, `OCI_PRIVATE_KEY` | OCI CLI auth |
| `OCI_COMPARTMENT_ID` | Instance discovery scope |
| `SSH_PRIVATE_KEY` | Ansible SSH to OCI workers |
| `MASTER_SSH_PRIVATE_KEY` | Ansible SSH to master (`sacenpapier.org`) |
| `TAILSCALE_AUTH_KEY` | Tailscale node auth key |
| `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ZONE_ID`, `RECORD_ID` | DNS update |
