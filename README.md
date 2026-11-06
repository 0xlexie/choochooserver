# Local services

Ansible deploys Glance, Securo, a self-hosted Anki sync server, Postiz, and Nginx on `local_apps`, plus qBittorrent and Nginx on `torrents`.

## Deploy

Requirements: Ansible with Vault support on your control machine, SSH access to both hosts, and sudo privileges. The hosts use `dnf`. Ensure `local_apps` and `torrents` resolve through your private DNS or SSH config.

1. Copy `env/local.apps.vault.yml.example` to `env/local.apps.vault.yml`.
2. Replace each `replace-with-...` value with a unique value. Generate application secrets with `openssl rand -hex 32`. For Glance, generate its secret key and password hash with:

   ```sh
   docker run --rm glanceapp/glance secret:make
   docker run --rm glanceapp/glance password:hash YOUR_PASSWORD
   ```

   Set the Anki sync credential as `username:password`. Leave `nginx_listen_addresses` at `127.0.0.1` and `nginx_allowed_cidrs` at `127.0.0.1/32` until WireGuard is set up.
3. Encrypt the variables file with a vault password kept outside this repository:

   ```sh
   ansible-vault encrypt env/local.apps.vault.yml
   ```

4. Securely remove the obsolete plaintext `env/local.apps.env` file; it contains the retired Kaneo settings.
5. Run:

   ```sh
   ansible-playbook --ask-vault-pass -i hosts.yml playbooks/main.yml
   ```

The first deployment stops the old Kaneo Compose stack without deleting its Docker volume. The old database volume is left on the host for backup or manual cleanup.

Glance, Securo, and Postiz use pinned upstream release tags (Glance `v0.8.7`, Securo `0.17.0`, Postiz `v2.25.3`). Review and update those tags deliberately when upgrading rather than relying on `latest`.

## Private access

The apps are private by default: their container ports bind to loopback, and Nginx listens only on `127.0.0.1` and allows only configured client addresses. They are not published to the internet. Until WireGuard is configured, remote devices cannot access them.

After setting up WireGuard, update the encrypted Vault variables:

- Add the server's WireGuard interface address to `nginx_listen_addresses`.
- Add only your device address(es) as `/32` IPv4 or `/128` IPv6 entries in `nginx_allowed_cidrs`.
- Configure private DNS to resolve the app names to the server's WireGuard address while connected to the VPN.
- If the host firewall is enabled, allow TCP port 80 only on the WireGuard interface. Do not forward app ports or HTTP from your router.

For an IPv6 WireGuard listener, put the address in brackets (for example, `[fd00::1]`). Some Postiz social integrations may require OAuth callback domains that are not available with the private `.train` names; this setup does not add public ingress to work around that.

Redeploy after changing Vault. Keep network addresses in your private notes, not in this repository. Configure private DNS with:

- `glance.choochoo.train` → `local_apps`
- `securo.choochoo.train` → `local_apps`
- `anki.choochoo.train` → `local_apps`
- `postiz.choochoo.train` → `local_apps`
- `torrents.choochoo.train` → `torrents`

For Anki clients, configure the self-hosted sync server URL as `http://anki.choochoo.train`.

## Software by host

**Torrents:**
- qBittorrent

**Private apps (WireGuard-only):**
- Glance
- Securo
- Anki sync server
- Postiz
