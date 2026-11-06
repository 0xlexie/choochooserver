# Local services

[![Validate](https://github.com/0xlexie/choochooserver/actions/workflows/validate.yml/badge.svg)](https://github.com/0xlexie/choochooserver/actions/workflows/validate.yml)

Ansible deploys services to two hosts:

| Host | Services |
|---|---|
| `local_apps` | Glance, Securo, Anki sync server, Postiz, Nginx |
| `torrents` | qBittorrent, Nginx |

## Requirements

- Ansible with Vault support on the control machine
- Docker on the control machine to generate Glance’s authentication values
- SSH access and sudo privileges on both hosts
- `dnf`-based hosts with Docker Compose available
- `local_apps` and `torrents` resolvable through private DNS or SSH config

## Getting started

Follow these steps on your Ansible control machine.

1. **Create your private settings file:**

   ```sh
   cp env/local.apps.vault.yml.example env/local.apps.vault.yml
   ```

2. **Generate and enter the secrets.** Generate random secrets with `openssl rand -hex 32`, then replace the matching `replace-with-...` values in the new file.

   For Glance, run:

   ```sh
   docker run --rm glanceapp/glance:v0.8.7 secret:make
   docker run --rm glanceapp/glance:v0.8.7 password:hash YOUR_PASSWORD
   ```

   Put the outputs in `GLANCE_AUTH_SECRET_KEY` and `GLANCE_AUTH_PASSWORD_HASH`. Replace `YOUR_PASSWORD` with the password you want to use.

   Set `SYNC_USER1` to `username:password` for Anki. Leave `OPENEXCHANGERATES_APP_ID` empty if you do not use that Securo integration.

3. **Keep the default access private.** Until WireGuard is configured, leave `nginx_listen_addresses` as `127.0.0.1` and `nginx_allowed_cidrs` as `127.0.0.1/32`.

4. **Encrypt the settings file.** Choose a Vault password and keep it outside this repository:

   ```sh
   ansible-vault encrypt env/local.apps.vault.yml
   ```

5. **Deploy the services:**

   ```sh
   ansible-playbook --ask-vault-pass -i hosts.yml playbooks/main.yml
   ```

   Enter the Vault password when prompted.

The private Vault file and `.env` files are ignored by Git. Never commit secrets or the Vault password. To edit the encrypted settings later, run `ansible-vault edit env/local.apps.vault.yml`.

## Private access

Apps are **not exposed to the internet**. Their ports bind to loopback, and Nginx initially listens only on `127.0.0.1`.

After setting up WireGuard:

1. Add the server’s WireGuard address to `nginx_listen_addresses`. For an IPv6 address, include brackets, e.g. `[fd00::1]`.
2. Add only your client address(es) to `nginx_allowed_cidrs`, using `/32` for IPv4 or `/128` for IPv6.
3. Configure private DNS on VPN clients to resolve the app names to the server’s WireGuard address.
4. If the host firewall is enabled, allow TCP port 80 only on the WireGuard interface. Do not forward app ports or HTTP from your router.
5. Encrypt the updated Vault file and redeploy.

| Private hostname | Service |
|---|---|
| `glance.choochoo.train` | Glance |
| `securo.choochoo.train` | Securo |
| `anki.choochoo.train` | Anki sync server |
| `postiz.choochoo.train` | Postiz |
| `torrents.choochoo.train` | qBittorrent |

Set Anki clients’ sync server URL to `http://anki.choochoo.train`. Some Postiz social integrations may require public OAuth callback domains; this deployment intentionally does not add public ingress.

## Repository map

| Path | Purpose |
|---|---|
| `hosts.yml` | Inventory; host names resolve through private DNS or SSH config |
| `playbooks/main.yml` | Runs the deployment playbooks |
| `playbooks/local_apps.yml` | Validates settings and deploys local apps |
| `playbooks/torrents.yml` | Deploys qBittorrent |
| `playbooks/roles/` | Shared Docker, app deployment, and Nginx tasks |
| `playbooks/tasks/` | Local app validation and one-time Kaneo cleanup |
| `playbooks/vars/local_apps.yml` | App stack and Nginx site definitions |
| `images/<app>/` | Each app’s Compose files and supporting configuration |
| `env/local.apps.vault.yml.example` | Template for private app settings and secrets |

Each app has its own directory under `images/`. Postiz’s Temporal dependencies are in `images/postiz/temporal.yml`; Anki’s sync server image is built from `images/anki/Dockerfile`.

## Updates and data

- Glance, Securo, and Postiz use pinned release tags in their Compose files. Review and update those tags deliberately.
- Anki’s version is set by `ANKI_VERSION` in the Vault file; updating it rebuilds the sync server image.
- App data is stored in Docker volumes. Replacing or stopping a stack does not delete its volumes.
- The first deployment stops the old Kaneo stack and removes its old remote plaintext `.env`, but preserves its database volume for backup or manual cleanup.
