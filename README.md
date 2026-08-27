# ansible-homelab

Ansible automation for provisioning and hardening a Debian-based homelab server, from a fresh install to a ready-to-use Docker host.

The playbook takes a freshly installed Debian box and: bootstraps SSH access and privilege escalation, applies system baseline configuration, locks down the firewall, and installs and configures Docker.

## What it does

| Role | Purpose |
|------|---------|
| **bootstrap** | Installs `sudo`, grants the admin user sudo rights, deploys the SSH public key, hardens `sshd` (disables password auth, limits auth tries, restricts allowed users), and locks the root password. |
| **base** | Full apt upgrade, installs common packages (git, curl, htop, rsync, nfs-common, etc.), sets timezone (`Europe/Warsaw`) and locale (`en_US.UTF-8`), and deploys an `htop` config. |
| **firewall** | Installs and configures **ufw**: default-deny incoming, allow outgoing, rate-limited SSH from the RFC1918 local network only. |
| **docker** | Installs Docker CE from the official Docker apt repo, deploys a `daemon.json` (json-file logging with rotation, `overlay2`, live-restore), adds the admin user to the `docker` group, and creates a `proxy` bridge network. |

## Requirements

- **Control node:** Ansible with the following collections:
  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```
  (`ansible.posix`, `community.general`, `community.docker`)
- **Managed node:** a fresh Debian install with SSH reachable and a non-root admin user.
- An SSH key pair at `~/.ssh/homelab` / `~/.ssh/homelab.pub` on the control node.

## Setup

The project uses an Ansible Vault and a `become` password read from local files (both git-ignored):

```bash
# vault password used to decrypt group_vars/.../vault.yml
echo 'your-vault-password' > .vault_password_file

# sudo/become password for the managed host
echo 'your-become-password' > .become_password_file

chmod 600 .vault_password_file .become_password_file
```

The vault stores `vault_os_administrator`, which populates `os_administrator` (the admin username used throughout the roles).

## Inventory

Two environments live under `inventory/`:

- `inventory/test/hosts.ini` — default (set in `ansible.cfg`)
- `inventory/prod/hosts.ini`

Edit the host entry to match your server:

```ini
[docker_hosts]
srv01 ansible_host=192.168.72.102 ansible_user=youruser ansible_ssh_private_key_file=~/.ssh/homelab

[linux:children]
docker_hosts
```

## Usage

Run the full playbook against the default (test) inventory:

```bash
ansible-playbook site.yml
```

Target the production inventory:

```bash
ansible-playbook -i inventory/prod/hosts.ini site.yml
```

Run a single role via tags/limits, e.g. only Docker hosts:

```bash
ansible-playbook site.yml --limit docker_hosts
```

Dry run to preview changes:

```bash
ansible-playbook site.yml --check --diff
```

## Layout

```
.
├── site.yml                # main playbook
├── ansible.cfg             # inventory, vault & become config
├── requirements.yml        # Galaxy collections
├── inventory/
│   ├── test/               # default environment (+ group_vars, vault)
│   └── prod/
└── roles/
    ├── bootstrap/          # SSH key, sudo, sshd hardening, lock root
    ├── base/               # packages, timezone, locale
    ├── firewall/           # ufw rules
    └── docker/             # Docker CE install + daemon config
```

## License

MIT — see [LICENSE](LICENSE).
