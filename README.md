# vgreens-ai-infra

Infrastructure-as-code (Ansible) for the vGreens AI platform. Run from your Mac
over the VPN; it configures `gpu-worker-01` (and later the CI runner on `sv01`).

**Ansible does not run training or touch GPUs** — it manages configuration:
packages, directories, systemd units, Docker, monitoring, reverse proxy,
GitLab Runner, backups. Training stays on the platform.

## Adoption phases (progressive, low-risk)

- **A — audit only (this repo now):** a read-only playbook that inspects
  production and changes nothing.
- **B — new components:** manage additions (dataset worker unit, reverse proxy,
  GitLab Runner, new monitoring) via Ansible.
- **C — import existing:** bring the API/worker units under Ansible after
  diffing against production.
- **D — documented deployment path.**

Never let Ansible overwrite `/opt/vgreens-ai/.env` or `jobs.db` without a
deliberate migration.

## Layout

```
ansible.cfg
requirements.yml
inventories/production/hosts.yml          # gpu-worker-01 via ProxyJump sv01
inventories/production/group_vars/all.yml # non-secret facts
inventories/production/group_vars/vault.yml   # ansible-vault encrypted (create locally)
playbooks/audit.yml                       # Phase A read-only audit
```

## Setup (Mac)

```bash
pipx install --include-deps ansible        # or: brew install ansible
cd ~/Desktop/vgreens-ai-infra
ansible-galaxy collection install -r requirements.yml
```

SSH must already work: `ssh -J vgreens@10.8.0.30 vgreens@172.16.20.89`.

## Run the audit (read-only)

```bash
ansible platform -m ping                   # connectivity via the jump host
ansible-playbook playbooks/audit.yml       # inspect; changes nothing
```

## Secrets

Encrypted with Ansible Vault in `inventories/production/group_vars/vault.yml`.
See `docs/secrets.md`. The vault password lives in `.vault-password` (git-ignored),
never committed.
