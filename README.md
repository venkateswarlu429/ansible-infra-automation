# ansible-infra-automation

Production-style Ansible automation for provisioning and operating
Debian/Ubuntu servers: a hardened baseline, Docker from the official
repository, Prometheus `node_exporter`, and templated Docker Compose
application deployments — all variable-driven, idempotent, and split into
small single-purpose roles.

Built to be read as much as run: every role works from its own defaults,
every environment difference lives in inventory variables, and secrets are
kept in Ansible Vault, never in the repo.

## Layout

```
.
├── ansible.cfg                  # sensible defaults (inventory, become, ssh)
├── requirements.yml             # Galaxy collections actually used by the roles
├── site.yml                     # entry point — one play per concern
├── inventories/
│   ├── dev/                     # hosts.ini, group_vars/, host_vars/
│   └── prod/                    # stricter group_vars than dev
└── roles/
    ├── common/                  # baseline: packages, timezone, NTP, admin user
    ├── hardening/               # sshd, fail2ban, sysctl, unattended upgrades
    ├── docker/                  # Docker Engine + Compose plugin (official repo)
    ├── monitoring/              # Prometheus node_exporter + systemd unit
    └── app_deploy/              # templated Docker Compose app stacks
```

Each role follows the standard layout — `tasks/`, `defaults/`, `handlers/`,
`templates/`, `meta/` — so it can be lifted into another project or published
to Galaxy unchanged.

## Roles

### `common` — baseline
- Installs a base package set (`curl`, `jq`, `tmux`, `htop`, …) via apt.
- Sets the timezone (`community.general.timezone`) and runs **chrony** for
  NTP with a templated `chrony.conf`.
- Creates an admin user (default `deploy`) in the `sudo` group with
  passwordless sudo via a validated `/etc/sudoers.d/` drop-in, and installs
  SSH keys with `ansible.posix.authorized_key` from the
  `common_admin_user_ssh_keys` variable.

### `hardening` — security baseline
- Renders an **sshd drop-in** (`/etc/ssh/sshd_config.d/60-hardening.conf`,
  validated with `sshd -t` before it can take effect): root login off,
  key-only authentication, `MaxAuthTries`, idle timeouts, optional
  `AllowUsers`.
- Installs and configures **fail2ban** from a templated `jail.local`
  (bantime/findtime/maxretry/ignoreip are variables).
- Applies a **sysctl baseline** (redirect/source-route rejection, SYN
  cookies, ptrace scope, protected links, …) with `ansible.posix.sysctl`
  into `/etc/sysctl.d/60-hardening.conf`.
- Enables **unattended security updates**: templated
  `50unattended-upgrades` + `20auto-upgrades` apt config and the
  `apt-daily-upgrade` timer. Automatic reboot is a variable, off by default.

### `docker` — container runtime
- Removes conflicting distribution packages, installs Docker's official
  GPG key into `/etc/apt/keyrings`, and adds the **official Docker apt
  repository** (architecture and codename derived from facts).
- Installs `docker-ce`, `docker-ce-cli`, `containerd.io`, buildx and the
  **Compose plugin**.
- Renders `/etc/docker/daemon.json` from the `docker_daemon_options`
  variable — JSON log rotation (`10m × 3`) and `live-restore` by default.
- Adds users from `docker_users` to the `docker` group.

### `monitoring` — Prometheus node_exporter
- Creates a dedicated system user, downloads the **pinned release tarball**
  (version and optional SHA-256 checksum are variables), and installs the
  binary to `/usr/local/bin`.
- Creates the textfile-collector directory and a **hardened systemd unit**
  (`ProtectSystem=strict`, `NoNewPrivileges`, `PrivateTmp`) rendered from a
  template; listen address and collectors are variables.

### `app_deploy` — application stacks
- Renders a **Docker Compose file from the `app_deploy_stack` variable**
  (`to_nice_yaml`) and an optional `.env` from `app_deploy_env_vars` (mode `0600`),
  so the whole stack definition lives in inventory data.
- Deploys with `community.docker.docker_compose_v2`: pull/recreate/orphan
  behaviour is variable-driven, and an `.env` change restarts the stack via
  a handler.

## Quick start

```bash
# 1. Install Ansible + the collections this project uses
pip install "ansible-core>=2.16"
ansible-galaxy collection install -r requirements.yml

# 2. Point an inventory at your hosts (examples ship in inventories/)
$EDITOR inventories/dev/hosts.ini

# 3. Always syntax-check, then dry-run with a diff before applying
ansible-playbook -i inventories/dev site.yml --syntax-check
ansible-playbook -i inventories/dev site.yml --check --diff

# 4. Apply (dev, then prod when you're confident)
ansible-playbook -i inventories/dev  site.yml
ansible-playbook -i inventories/prod site.yml --ask-vault-pass

# Run a single concern with tags
ansible-playbook -i inventories/dev site.yml --tags hardening
ansible-playbook -i inventories/dev site.yml --tags docker,app
```

### Trying it safely
Spin up a throwaway Ubuntu 24.04 VM (Vagrant, Multipass, a cloud free-tier
instance, …), add it to `inventories/dev/hosts.ini`, and run with
`--check --diff` first. Everything in this repo is designed so the dry run
is meaningful: templates validate (`sshd -t`, `visudo -cf`), and handlers
only fire on real changes.

## Variables overview

Precedence: **role defaults < inventory `group_vars` < `host_vars` < vault**.

| Area | Key variables | Where |
| --- | --- | --- |
| Baseline | `common_base_packages`, `common_timezone`, `common_ntp_servers` | `roles/common/defaults` |
| Admin user | `common_admin_user_name`, `common_admin_user_ssh_keys`, `common_admin_user_password_hash` | `roles/common/defaults`, vault |
| sshd | `hardening_sshd_permit_root_login`, `hardening_sshd_password_auth`, `hardening_sshd_allow_users` | `roles/hardening/defaults` |
| fail2ban | `hardening_fail2ban_bantime`, `hardening_fail2ban_maxretry`, `hardening_fail2ban_jails` | `roles/hardening/defaults` |
| sysctl | `hardening_sysctl_settings` (whole dict) | `roles/hardening/defaults` |
| Updates | `hardening_unattended_upgrades_enabled`, `hardening_unattended_automatic_reboot` | `roles/hardening/defaults` |
| Docker | `docker_daemon_options`, `docker_users`, `docker_ce_packages` | `roles/docker/defaults` |
| Monitoring | `monitoring_node_exporter_version`, `monitoring_node_exporter_checksum`, `monitoring_node_exporter_listen_address` | `roles/monitoring/defaults` |
| App | `app_deploy_name`, `app_deploy_stack`, `app_deploy_env_vars`, `app_deploy_pull` | `roles/app_deploy/defaults`, `group_vars` |

The `prod` inventory shows the pattern in action: same roles, stricter
variables (longer fail2ban bans, resource limits on the app stack).

## Idempotency

Every task uses declarative modules (`apt`, `template`, `sysctl`,
`authorized_key`, `docker_compose_v2`, …) with change detection, so a
second run against a converged host reports `ok` instead of `changed`.
Templates that can break a service are validated before landing
(`sshd -t` for sshd, `visudo -cf` for sudoers), and service restarts only
happen through handlers when the underlying config actually changes.

## Secrets — Ansible Vault

No real secrets are committed. Each inventory ships a
`group_vars/vault.yml.example` documenting the expected vault variables —
copy it to `vault.yml`, fill in real values, and encrypt:

```bash
cp inventories/prod/group_vars/vault.yml.example inventories/prod/group_vars/vault.yml
$EDITOR inventories/prod/group_vars/vault.yml
ansible-vault encrypt inventories/prod/group_vars/vault.yml
ansible-playbook -i inventories/prod site.yml --ask-vault-pass
```

`.gitignore` excludes real `vault.yml` files (the `.example` files stay
tracked), so the encrypted workflow can't leak by accident. Vault-backed
variables use the `vault_` prefix and are mapped onto plain role variables,
which keeps role defaults free of secret references.

## CI

`.github/workflows/ci.yml` runs on every push and PR:

1. Installs `ansible-core` + `ansible-lint` and the Galaxy collections.
2. Runs **ansible-lint** over `site.yml` and `roles/`.
3. Runs **`ansible-playbook --syntax-check`** against both inventories.

## What this demonstrates

- Role-based Ansible design: small roles with real defaults, handlers,
  templates, and Galaxy metadata — not a pile of playbooks.
- Configuration-as-data: environments differ only in inventory variables,
  including rendering an entire Compose stack from a variable.
- Security-first defaults: key-only SSH, fail2ban, kernel hardening,
  automatic security patches, least-privilege service sandboxing, and
  vault-only secrets.
- Operational discipline: syntax checks and lint in CI, validated config
  files, check-mode-friendly tasks, and handlers instead of unconditional
  restarts.

## License

MIT — see [LICENSE](LICENSE).

Author: **Venkateswarlu Maddula** — Senior DevOps Engineer
(AWS/Azure · Kubernetes · Terraform · CI/CD).
