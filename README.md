# 🖥 Homelab

[![Validate](https://github.com/0xlexie/choochooserver/actions/workflows/validate.yml/badge.svg)](https://github.com/0xlexie/choochooserver/actions/workflows/validate.yml)

Very simple setup with utility software, mostly information management. Glance is used as the main dashboard. I use wireguard to connect from the ouside.

*Ansible deploys services to two hosts:*

| Host | Services |
|---|---|
| `local_apps` | [Glance](https://github.com/glanceapp/glance), [Securo](https://github.com/securo-finance/securo), [Anki sync server](https://github.com/ankitects/anki), [Postiz](https://github.com/gitroomhq/postiz-app), Nginx |
| `torrents` | [qBittorrent](https://github.com/qbittorrent/qBittorrent), Nginx |

## Requirements

- Ansible with Vault support and the `ansible.posix` collection on the control machine
- Docker on the control machine to generate Glance’s authentication values
- SSH access and sudo privileges on both hosts
- `dnf`-based hosts with Docker Compose available
- `local_apps` and `torrents` resolvable through private DNS or SSH config

## Repository map

- **`hosts.yml`** — Inventory; host names resolve through private DNS or SSH config.
- **`playbooks/main.yml`** — Runs the deployment playbooks.
- **`playbooks/local_apps.yml`** — Validates settings and deploys local apps.
- **`playbooks/torrents.yml`** — Deploys qBittorrent.
- **`playbooks/roles/`** — Shared Docker, app deployment, and Nginx tasks.
- **`playbooks/tasks/`** — Local app validation and one-time Kaneo cleanup.
- **`playbooks/vars/local_apps.yml`** — App stack and Nginx site definitions.
- **`images/<app>/`** — Each app’s Compose files and supporting configuration.
- **`env/local.apps.vault.yml.example`** — Template for private app settings and secrets.

## Getting started

Follow these steps on your Ansible control machine.

1. **Create your private settings file:**

   ```sh
   cp env/local.apps.vault.yml.example env/local.apps.vault.yml
   ```

2. **Install the required Ansible collection:**

   ```sh
   ansible-galaxy collection install -r collections/requirements.yml
   ```

3. **Generate and enter the secrets.** Generate random secrets with `openssl rand -hex 32`, then replace the matching `replace-with-...` values in the new file.

   For Glance, run:

   ```sh
   docker run --rm glanceapp/glance:v0.8.7 secret:make
   docker run --rm glanceapp/glance:v0.8.7 password:hash YOUR_PASSWORD
   ```

   Put the outputs in `GLANCE_AUTH_SECRET_KEY` and `GLANCE_AUTH_PASSWORD_HASH`. Replace `YOUR_PASSWORD` with the password you want to use.

   Set `SYNC_USER1` to `username:password` for Anki. Leave `OPENEXCHANGERATES_APP_ID` empty if you do not use that Securo integration.
   Set `PROXMOXVE_URL` to the Proxmox hostname and port (without `https://`) and `PROXMOXVE_KEY` to a dedicated API token with read-only `PVEAuditor` permissions. Proxmox must use a TLS certificate trusted by the host. Set `TORRENTS_MONITOR_URL` to qBittorrent's private URL, reachable from the `local_apps` host.

4. **Keep the default access private.** Until WireGuard is configured, leave `nginx_listen_addresses` as `127.0.0.1` and `nginx_allowed_cidrs` as `127.0.0.1/32`.

5. **Encrypt the settings file.** Choose a Vault password and keep it outside this repository:

   ```sh
   ansible-vault encrypt env/local.apps.vault.yml
   ```

6. **Deploy the services:**

   ```sh
   ansible-playbook --ask-vault-pass -i hosts.yml playbooks/main.yml
   ```

   Enter the Vault password when prompted.

To edit the encrypted settings later, run `ansible-vault edit env/local.apps.vault.yml`.

## Service notes

<details>
<summary>Show service notes</summary>

- **Glance** — Compose file: `images/glance/compose.yml`; dashboard configuration is rendered from a template. It uses host networking to monitor loopback-bound services but listens only on `127.0.0.1`. Proxmox access uses a read-only API token and requires a trusted TLS certificate.
- **Dashboard widgets** — Kanagawa Dark theme, local service monitor, Proxmox VE/nodes, markets, Twitch channels, and Steam Specials (EUR prices).
- **Community widgets** — Proxmox and Steam widget examples are based on [glanceapp/community-widgets](https://github.com/glanceapp/community-widgets).
- **Securo** — Compose file: `images/securo/compose.yml`. Uses a pinned release tag; update it deliberately.
- **Anki sync server** — Compose file and Dockerfile: `images/anki/`. Set `ANKI_VERSION` in Vault; changing it rebuilds the image.
- **Postiz** — Compose files: `images/postiz/compose.yml` and `images/postiz/temporal.yml`. Uses a pinned release tag; update it deliberately.
- **Persistent data** — App data is stored in Docker volumes. Replacing or stopping a stack does not delete its volumes.

</details>

## Private access (Wireguard)

<details>
<summary>Show private access instructions</summary>

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

</details>
