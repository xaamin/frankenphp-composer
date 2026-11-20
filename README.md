# frankenphp-composer

Generates a `docker-compose.yml` that serves a PHP app with [FrankenPHP](https://frankenphp.dev) (`dunglas/frankenphp`), using the latest PHP version unless you pick one. The app is served from the project's `public/` folder, and the container runs as your host user, so files it creates belong to you.

## Requirements

- Docker with the Compose plugin
- Bash

## Installation

From the root of your PHP project, download and run the `dockerize` script:

```bash
curl -fsSL https://raw.githubusercontent.com/xaamin/frankenphp-composer/refs/heads/master/dockerize -o dockerize
bash dockerize
```

It downloads `.caddy/.gitignore`, `Caddyfile`, `compose` and `docker-compose.t.yml` into the project and prints the usage examples. Keep `dockerize` in your project and re-run it to update those files to the latest version. It overwrites them, including any manual edits to `Caddyfile`. Requires `curl`.

## Usage

Run `compose` from your project root, then start the container:

```bash
# HTTPS for local development, certificate issued by Caddy (binds ports 80 and 443)
bash compose -n myapp -s myapp.localhost -a

# HTTPS on a public domain, certificate issued and renewed by Caddy
bash compose -n myapp -s example.com -a

# HTTPS with your own certificate files in .caddy/certs/
bash compose -n myapp -s example.com

# Plain HTTP on a single port, e.g. behind a reverse proxy
bash compose -n myapp -p 8080

docker compose up -d
```

| Flag | Description |
| --- | --- |
| `-n`, `--name` | Container name (required) |
| `-s`, `--server` | Domain to serve over HTTPS |
| `-a`, `--auto` | With `-s`: let Caddy issue and renew the certificate instead of using files in `.caddy/certs/` |
| `-p`, `--port` | Port to serve over plain HTTP |
| `-u`, `--uid` | User ID to run the container as (default: your UID) |
| `-g`, `--gid` | Group ID to run the container as (default: your GID) |
| `-v`, `--verbose` | Print extra details |
| `-h`, `--help` | Show usage |

You must pass exactly one of `-s` or `-p`. `-a` only works together with `-s`.

### Choosing a mode

| Use case | Command | Certificate | Host ports |
| --- | --- | --- | --- |
| Local development | `-s myapp.localhost -a` | Issued by Caddy's local CA | 80, 443 |
| Public server | `-s example.com -a` | Issued and renewed by Caddy (Let's Encrypt/ZeroSSL) | 80, 443 |
| Your own certificate (e.g. purchased) | `-s example.com` | Files in `.caddy/certs/` | 80, 443 |
| Behind another reverse proxy | `-p 8080` | None, plain HTTP | the given port |

`compose` prints the TLS mode it picked (🔒 auto, 🔒 custom or 🔓 disabled) along with a config summary. The HTTPS modes bind host ports 80 and 443, so they fail to start if something else is already using those ports. In that case, stop the other service or use port mode behind it.

### PHP version

By default the latest PHP version is used. To pin a specific version, set `PHP_VERSION` in your project's `.env` file (see [`.env.example`](.env.example)):

```dotenv
PHP_VERSION=8.5
```

| `PHP_VERSION` | Image |
| --- | --- |
| not set or empty | `dunglas/frankenphp:1` (latest PHP) |
| `8.5` | `dunglas/frankenphp:1-php8.5` |

- The version must be one FrankenPHP publishes an image for.
- A `PHP_VERSION` already set in your shell takes precedence over `.env`.
- The version is written into `docker-compose.yml` when it's generated, so re-run `compose` after changing it.
- "Latest" means the newest image you have locally. To move to a newer PHP release, pull the image and recreate the container:

  ```bash
  docker compose pull
  docker compose up -d
  ```

### Project name

Docker Compose groups the container under a project name, shown in `docker compose ls` and Docker Desktop. By default `compose` uses the container name given with `-n`, lowercased, with anything other than letters, digits and `-` (including `_` and `.`) replaced by `-` (for example, `-n My.Shop_API` becomes `my-shop-api`). To use a different name, set `COMPOSE_PROJECT_NAME` in `.env` (see [`.env.example`](.env.example)) and re-run `compose`. A value set in your shell takes precedence over `.env`.

```dotenv
COMPOSE_PROJECT_NAME=myapp
```

If the container is already running, stop it with `docker compose down` **before** changing the name. Otherwise the next `docker compose up -d` treats it as a new project and fails, because a container with the `-n` name already exists.

### Local development with `.localhost` (`-s` with `-a`)

Use a `.localhost` domain such as `myapp.localhost`. It always points to your own machine, so no `/etc/hosts` entry is needed, and Caddy issues the certificate itself from its own local certificate authority (CA), with no internet access required.

```bash
bash compose -n myapp -s myapp.localhost -a
docker compose up -d
```

The CA is trusted **on the host**, not inside the container, which is why the `Caddyfile` sets `skip_install_trust`. After the first start, trust Caddy's root certificate once so browsers accept the site:

```bash
# macOS
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain .caddy/data/caddy/pki/authorities/local/root.crt

# Debian/Ubuntu
sudo cp .caddy/data/caddy/pki/authorities/local/root.crt /usr/local/share/ca-certificates/caddy-local.crt
sudo update-ca-certificates
```

Firefox uses its own certificate store, so import the file there separately if you use it. The root certificate lives in `.caddy/data`; if you delete that folder, Caddy creates a new CA and you have to trust it again.

### Public domain (`-s` with `-a`)

For a public domain, add `-a` and Caddy gets a free certificate from Let's Encrypt or ZeroSSL and renews it automatically. `compose` leaves the `tls` line commented and doesn't use `.caddy/certs/`. Requirements:

- The domain's DNS points to the server's public IP.
- Ports 80 and 443 are reachable from the internet, because the certificate authority connects to Caddy to verify the domain.

Certificates are stored in `.caddy/data`, so they survive container restarts.

### Your own certificate (`-s` without `-a`)

Use this when you have certificate files for the domain, for example a purchased certificate. `compose` uncomments the `tls` line in the `Caddyfile` so Caddy serves those files:

```
tls /etc/caddy/certs/{$NAME}.pem /etc/caddy/certs/{$NAME}-key.pem
```

1. Run `compose`, e.g. `bash compose -n myapp -s example.com`.
2. Copy your certificate to `.caddy/certs/myapp.pem` and its private key to `.caddy/certs/myapp-key.pem`. The file names must match the `-n` container name. Use the full chain (your certificate followed by the intermediate certificates) in the `.pem` file.
3. Start or restart the container with `docker compose up -d`.

`compose` never creates or overwrites these files; it warns if they're missing. The domain must resolve to the machine running the container.

### Port mode (`-p`)

TLS is disabled in the `Caddyfile`, and the container listens only on the given port. Use this when another proxy terminates TLS in front of it. Caddy trusts proxies on private network ranges (the Docker network or LAN), so its own logs and matchers use the client IP they forward. PHP still sees the proxy as `REMOTE_ADDR` and no `HTTPS` flag. The original values arrive as `X-Forwarded-For` and `X-Forwarded-Proto` headers, so configure your framework to trust them (for example, Laravel's `TrustProxies` middleware).

## Notes

- The `tls` line in `Caddyfile` ships commented out. `compose` edits the file in place to uncomment it for your own certificate (`-s`) and comment it again for a Caddy-managed certificate (`-s -a`) or port mode (`-p`), so expect that file to change when you switch modes.
- `docker-compose.yml` is generated output and is gitignored. Re-run `compose` instead of editing it by hand.
- Caddy's data, config and certificates are kept under `.caddy/` (gitignored), so they survive container restarts.
