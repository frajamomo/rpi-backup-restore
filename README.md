# RPi Backup & Restore

Ansible playbooks to back up and restore a Raspberry Pi 5 running Docker containers, OpenVPN, and bastion-hardening configuration to a NAS via CIFS.

## What gets backed up

| Directory | Contents | Size |
|-----------|----------|------|
| `system/` | SSH keys/config, iptables, sysctl, fail2ban, auditd, AIDE config, OpenVPN + PKI, fstab, cmdline.txt, systemd units, cron jobs, Synology mount monitor | ~2 MB |
| `stacks/` | `/opt/stacks/` — Docker Compose files and container config/data (excludes qBittorrent downloads) | ~2.8 GB |
| `user/`   | `.smbcredentials`, `.ssh/authorized_keys` | ~8 KB |

Total: ~2.8 GB per backup.

## What is NOT backed up

- qBittorrent downloads (`/opt/stacks/qbittorrent/downloads/`)
- NAS mount contents
- Docker images/layers (re-pulled on restore)
- AIDE database (regenerated after restore)
- Logs

## Prerequisites

- Ansible with `ansible.posix` collection:
  ```
  ansible-galaxy collection install -r requirements.yml
  ```
- SMB credentials file at `~/.smbcredentials`
- NAS share accessible via CIFS

## Setup

1. Copy the vault example and fill in your values:
   ```
   cp inventory/group_vars/all/vault.yml.example inventory/group_vars/all/vault.yml
   vi inventory/group_vars/all/vault.yml
   ```

2. Encrypt the vault file:
   ```
   ansible-vault encrypt inventory/group_vars/all/vault.yml
   ```

3. To edit the vault later:
   ```
   ansible-vault edit inventory/group_vars/all/vault.yml
   ```

## Encryption

Snapshots are stored as a single AES-256 archive, `<timestamp>.tar.gz.gpg`.
The tree contains the OpenVPN CA and client keys, the SSH host keys,
`.smbcredentials` and application tokens, and CIFS derives file modes from the
mount options, so permissions alone cannot protect it on the NAS.

Both playbooks **prompt for the passphrase**; it is never stored in the repo
or the vault. Backup asks twice, because a typo would produce an archive
nobody can open. Keep the passphrase somewhere other than the Pi — without it
the backups are unreadable.

Snapshots taken before encryption was introduced are plain directories;
restore still reads those if no archive of that name exists.

## Usage

### Backup

From a remote workstation:
```
ansible-playbook playbooks/backup.yml --ask-vault-pass
```

From the Raspberry Pi itself:
```
ansible-playbook playbooks/backup.yml -e ansible_connection=local --ask-vault-pass
```

Skip stopping containers (faster, but data may be inconsistent):
```
ansible-playbook playbooks/backup.yml --ask-vault-pass -e backup_stop_docker=false
```

> **Tip:** To avoid typing the vault password each time, create a file containing the password and reference it with `--vault-password-file ~/.vault_pass` or set `vault_password_file` in `ansible.cfg`.

### Restore

Restore from the latest backup:
```
ansible-playbook playbooks/restore.yml --ask-vault-pass
```

Restore a specific snapshot:
```
ansible-playbook playbooks/restore.yml --ask-vault-pass -e restore_snapshot=20260301_115357
```

Restore only specific components using tags:
```
ansible-playbook playbooks/restore.yml --ask-vault-pass --tags docker_stacks
ansible-playbook playbooks/restore.yml --ask-vault-pass --tags openvpn
ansible-playbook playbooks/restore.yml --ask-vault-pass --skip-tags packages
```

Available tags: `packages`, `system_config`, `docker_stacks`, `openvpn`, `services`, `verify`.

### After restore

1. Run the bastion-hardening playbook to re-apply security policies
2. Regenerate the AIDE database: `sudo aide --init`
3. Verify OpenVPN clients can connect
4. Verify all Docker containers are healthy

## Backup workflow

1. Prompt for the encryption passphrase
2. Mount NAS share via CIFS
3. Create timestamped directory (`YYYYMMDD_HHMMSS`)
4. Stop Docker containers for data consistency
5. Rsync system config, user files, and Docker stacks
6. Restart Docker containers
7. Pack and encrypt the tree into `<timestamp>.tar.gz.gpg`
8. Test-decrypt the archive, then delete the plaintext tree
9. Update `latest` marker file
10. Prune backups beyond retention limit (default: 5)
11. Print report with size, duration, and container status
12. Unmount NAS share

## Restore workflow

1. Prompt for the passphrase
2. Mount the NAS share, resolve the snapshot (`latest` by default)
3. Decrypt and unpack into a local staging directory — a wrong passphrase
   fails here, before anything on the host is modified
4. Restore packages, system config, Docker stacks, OpenVPN, services
5. Remove the staging directory and the temporary passphrase file, always

## Key variables

Sensitive variables are stored in `inventory/group_vars/all/vault.yml` (encrypted). See `vault.yml.example` for the full list.

Non-sensitive settings in `inventory/group_vars/all/all.yml`:

| Variable | Default | Description |
|----------|---------|-------------|
| `backup_retention` | `5` | Number of backups to keep |
| `backup_stop_docker` | `true` | Stop containers before rsync |
| `restore_snapshot` | `latest` | Snapshot to restore (`latest` or `YYYYMMDD_HHMMSS`) |
| `restore_install_packages` | `true` | Install packages on fresh OS |

## Project structure

```
rpi-backup-restore/
├── ansible.cfg
├── requirements.yml
├── .gitignore
├── inventory/
│   ├── hosts.yml                        # host definitions
│   └── group_vars/all/
│       ├── all.yml                      # non-sensitive variables
│       ├── vault.yml                    # sensitive variables (encrypted)
│       └── vault.yml.example            # placeholder template
├── playbooks/
│   ├── backup.yml
│   └── restore.yml
└── roles/
    ├── backup/
    │   ├── defaults/main.yml
    │   └── tasks/main.yml
    └── restore/
        ├── defaults/main.yml
        ├── handlers/main.yml
        └── tasks/
            ├── main.yml
            ├── packages.yml
            ├── system_config.yml
            ├── docker_stacks.yml
            ├── openvpn.yml
            ├── services.yml
            └── verify.yml
```
