# RCV PingIt

**PingIt** is multi-vantage-point network monitoring: a collector hub plus a fleet of
lightweight remote agents that probe your targets and report round-trip time, packet loss and
jitter, with per-agent dashboards, hop-by-hop traceroute and alert delivery by webhook or
e-mail. It is a standalone, self-hosted app.

This repository is the **install bundle**: a Docker Compose stack that pulls the published
PingIt images. Each release attaches the same bundle as a single archive.

## Requirements

- A Linux host (amd64 or arm64) with **Docker Engine 24+** and the **Docker Compose v2 plugin**
  (`docker compose version`). 2 CPU cores, 2 GB RAM and 10 GB of disk are plenty to start.
- Outbound access to `ghcr.io` to pull the images.
- Inbound TCP 8443 (HTTPS) and 8080 (redirect) by default; both are configurable in `.env`.
- `openssl` on the host; the installer uses it to generate the agent-signing keypair.
- **Unprivileged LXC or other restricted containers:** the local worker needs the sysctl
  `net.ipv4.ping_group_range`, which such containers often cannot set to the default
  `0 2147483647` (the worker then fails to start with a sysctl "invalid argument" error). Set
  `PINGIT_PING_GROUP_RANGE="0 65535"` in `.env` and run `docker compose up -d worker`. The
  installer detects this, retries with `0 65535` and saves it to `.env` for you. Alternatively
  run the container privileged, or set the sysctl on the host.
- **One install per host.** The Compose project is named `pingit`, and containers, network and
  volumes use that prefix. To run a second copy on the same host, set `COMPOSE_PROJECT_NAME` (and
  different ports) in its `.env`. Volume names in the backup commands below then use your project
  name instead of `pingit`.

## Quick start

```sh
# 1. Download and extract the bundle from this repo's latest release
mkdir pingit && cd pingit
tar -xzf ~/Downloads/pingit-install-*.tar.gz     # or copy this repo's files here

# 2. Install: generates .env with strong random secrets, pulls images, starts the stack
./install.sh

# 3. Open https://<host>:8443/ and read the one-time setup token
docker compose logs web | grep 'SETUP TOKEN'
```

There is **no default account**. The first page asks for the setup token, then creates the first
admin. After that the wizard offers instance naming, optional single sign-on and default targets.

If the installer did not print the setup token (or you missed it), read it any time before the
first admin exists:

```sh
docker compose logs web | grep 'SETUP TOKEN'
```

On first sign-in, set your company name and logo under **Settings -> Branding** (the "Set up your
organization" prompt appears once the setup wizard is finished).

The installer pins the images to the version in `.env` (`PINGIT_VERSION`) and never overwrites an
existing `.env`.

## Sign-in

- **Local accounts.** The first-run admin, plus users created under **Users**.
  Forgot the admin password, or locked out?
  `docker compose exec web pingit admin reset --email you@example.com` creates or resets a local
  admin and prints a one-time password.
- **Single sign-on (optional).** One OpenID Connect provider of your choice, configured in
  **Settings -> SSO**: issuer URL, client id and secret, scopes, a claim-to-role mapping, PKCE.
  Register PingIt at the provider with the redirect URI `https://<your-host>/api/auth/oidc/callback`.
  - **Microsoft Entra ID:** issuer `https://login.microsoftonline.com/<tenant-id>/v2.0`.
  - **Authentik:** issuer `https://<authentik>/application/o/<slug>/`.
  - **Your NOC Portal (or any other OIDC provider):** use the Portal's public URL as the issuer.
    The Portal only needs PingIt registered as an OIDC client.
  Turn password sign-in off only after SSO is configured and tested. If the provider is down,
  set `PINGIT_ALLOW_LOCAL_LOGIN_OVERRIDE=1` in `.env` and restart `web` to get password login back.

## Remote agents

In the web UI open **Agents -> Add Agent**. PingIt issues a one-time enrollment token and a
one-line install command; run it on any Linux host. The agent installs as a systemd service,
enrolls over outbound HTTPS and starts probing. Agent packages ship inside the collector image
and updates are signed by the collector and applied automatically.

**Certificates and remote agents.** With the default self-signed (internal CA) certificate, a remote
agent cannot verify the collector. Either use the variant of the install command that sets
`PINGIT_INSECURE=1` (the **Add Agent** dialog has an "insecure" option that adds `curl -k` and
`PINGIT_INSECURE=1`; it skips certificate verification, so use it on trusted networks only), or
install a trusted certificate on the collector (see "TLS and your own hostname" below) and use the
command as shown.

The compose stack also runs a **local worker** that enrolls itself, so a single host produces
data immediately.

## TLS and your own hostname

Out of the box Caddy serves HTTPS with its own CA, so browsers warn once. For a real hostname and
a public certificate, edit `caddy/Caddyfile`: replace the `:443 { ... }` block with

```
pingit.example.com {
	encode zstd gzip
	reverse_proxy web:8000
}
```

remove the `on_demand_tls` and `auto_https disable_redirects` lines from the global block at the
top, keep the `:80` block, set `PINGIT_HTTP_PORT=80` and `PINGIT_HTTPS_PORT=443` in `.env`, then
`docker compose up -d`. Caddy obtains the certificate on its own; ports 80 and 443 must be
reachable from the internet for that. Behind your own TLS proxy instead, point it at the `web`
service and set `PINGIT_REQUIRE_HTTPS=1`.

PingIt is served at the root of its host by default. To mount it under a path (for example
`/pingit`), set `PINGIT_ROOT_PATH=/pingit` in `.env` and use a proxy that strips the prefix.

## Upgrade

```sh
# edit .env: PINGIT_VERSION=<new release tag>
docker compose pull
docker compose up -d        # database migrations run on start
```

Take a backup first (below). Do not downgrade across a release that changed the database
schema without restoring a backup taken before the upgrade.

## Backup and restore

Three things make a complete backup. Store the **secrets file separately from the database
dump**.

```sh
# 1. Database
docker compose exec -T db pg_dump -U pingit -d pingit -Fc > pingit-db-$(date -u +%Y%m%d).pgdump

# 2. Secrets: .env holds the database password, the secrets-encryption key and the agent-signing keypair
cp .env pingit-env-$(date -u +%Y%m%d).backup && chmod 600 pingit-env-*.backup

# 3. Optional: the signing volume (the signing key as the collector holds it)
docker run --rm -v pingit_pingit_signing:/v -v "$PWD":/b alpine tar -C /v -czf /b/pingit-signing.tar.gz .
```

Restore into a fresh install: put the old `.env` in place (so the keys match), start only the
database (`docker compose up -d db`), then follow TimescaleDB's restore procedure:

```sh
docker compose exec -T db psql -U pingit -d pingit -c "SELECT timescaledb_pre_restore();"
docker compose exec -T db pg_restore -U pingit -d pingit -Fc --no-owner < pingit-db-YYYYMMDD.pgdump
docker compose exec -T db psql -U pingit -d pingit -c "SELECT timescaledb_post_restore();"
docker compose up -d        # web migrates to the current schema on start
```

**Keep the agent-signing key.** Enrolled agents trust it; a collector with a different key can no
longer update them, and they must be re-enrolled. The secrets-encryption key is equally needed:
without it, stored SSO and integration credentials cannot be decrypted.

## Moving from a NOC Portal-hosted PingIt

Earlier PingIt versions were installed as a NOC Portal module. To move that install to this
standalone stack and keep your data, users and enrolled agents:

1. Back up the database, `.env` and the signing key from the old install.
2. Install this bundle on its own host or directory, but **before starting it**, copy the old
   `PINGIT_SECRETS_ENCRYPTION_KEY`, `PINGIT_AGENT_SIGNING_PRIVKEY_PEM` and
   `PINGIT_AGENT_SIGNING_PUBKEY_B64` values into the new `.env`, then restore the database dump.
3. In **Settings -> SSO**, point PingIt at your Portal as an OIDC provider and set **Legacy Portal
   issuer** to the same URL so existing users keep their accounts, or create a local admin with
   `pingit admin reset`.
4. If the collector's URL changed, re-point each agent: set `collector_url` in
   `/etc/pingit-agent/agent.toml` and run `systemctl restart pingit-agent`.
5. Unregister PingIt from the Portal.

## Uninstall

```sh
docker compose down          # stop, keep data
docker compose down -v       # stop AND delete the database and all volumes
rm -f .env                   # removes the secrets as well
```

## License

**RCV Community License 1.1**: free for personal and home-lab use and for any organization's own
internal operations; a paid commercial license is required to offer PingIt to third parties as a
hosted, managed or SaaS service. See [`LICENSE`](LICENSE). Commercial inquiries:
legal@rootchainventures.com.
