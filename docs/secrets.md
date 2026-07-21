# Secrets (Ansible Vault)

Infrastructure secrets are stored encrypted with Ansible Vault, never in plain
git. The vault password is kept locally in `.vault-password` (git-ignored) and
referenced by `ansible.cfg` (`vault_password_file`).

## One-time setup (Mac)

```bash
cd ~/Desktop/vgreens-ai-infra
# create a strong vault password locally (not committed)
printf '%s' 'CHOOSE-A-STRONG-PASSPHRASE' > .vault-password && chmod 600 .vault-password
# create the encrypted vars file and add the keys from vault.yml.example
ansible-vault create inventories/production/group_vars/vault.yml
```

Edit later with `ansible-vault edit inventories/production/group_vars/vault.yml`.

## What goes in the vault (Phase B onward)

- `vault_roboflow_api_key`
- `vault_ci_api_token`
- `vault_gitlab_runner_token`
- `vault_nas_password`
- registry pull token, S3 credentials, etc.

## Rules

- Never commit `.vault-password` or a decrypted vault.
- Use `no_log: true` on tasks that pass secret values.
- Deployed secret files get mode `0600`.
- Ansible must not overwrite `/opt/vgreens-ai/.env` in place without a reviewed
  migration — it already contains live tokens.
