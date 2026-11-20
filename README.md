# frankenphp-composer

Generates a `docker-compose.yml` and a `Dockerfile` that serve a PHP app with [FrankenPHP](https://frankenphp.dev) (`dunglas/frankenphp`), using the latest PHP version unless you pick one. The app is served from the project's `public/` folder, and the container runs as your host user, so files it creates belong to you.

## Requirements

- Docker with the Compose plugin
- Bash

## Installation

From the root of your PHP project, download and run the `dockerize` script into a `bash/` folder:

```bash
mkdir -p bash
curl -fsSL https://raw.githubusercontent.com/xaamin/frankenphp-composer/refs/heads/master/bash/dockerize -o bash/dockerize
bash bash/dockerize
```

It downloads `.caddy/.gitignore`, `Caddyfile`, `Dockerfile.template`, `docker-compose.template.yml`, `bash/compose` and `bash/reload` into the project and prints the usage examples. If the project has no `deploy.yml`, it also creates one from [`deploy.template.yml`](deploy.template.yml), a sample Deployer config to fill in; an existing `deploy.yml` is never overwritten. If the project has a `.env`, it appends the settings from [`.env.example`](.env.example) (`PHP_VERSION`, `COMPOSE_PROJECT_NAME`) that it doesn't define yet, with their comments, under a `# frankenphp-composer` heading; existing values are kept, and no `.env` is created if there isn't one. Keep `bash/dockerize` in your project and re-run it to update those files to the latest version. It overwrites them, including any manual edits to `Caddyfile` and `Dockerfile.template`. Requires `curl`.

### Upgrading from older versions

Files were renamed: `docker-compose.t.yml` → `docker-compose.template.yml`, `Dockerfile.t` → `Dockerfile.template`, and the scripts moved to `bash/` (`compose`, `reload`, `dockerize`). Copies of `dockerize` from before that still try to download the old paths and fail with `Could not download`. Download `bash/dockerize` once (see [Installation](#installation)) and run it. It warns about leftover old files: move any custom changes into the new ones and delete the old files.

## Usage

Run `compose` from your project root, then start the container:

```bash
# HTTPS for local development, certificate issued by Caddy (binds ports 80 and 443)
bash bash/compose -n myapp -s myapp.localhost -a

# HTTPS on a public domain, certificate issued and renewed by Caddy
bash bash/compose -n myapp -s example.com -a

# HTTPS with your own certificate files in .caddy/certs/
bash bash/compose -n myapp -s example.com

# Plain HTTP on a single port, e.g. behind a reverse proxy
bash bash/compose -n myapp -p 8080

docker compose up -d
```

The image is tagged with a checksum of the generated `Dockerfile`, so `docker compose up -d` builds it only when `Dockerfile.template` or `PHP_VERSION` changed; otherwise it reuses the existing image and leaves the running container alone.

| Flag | Description |
| --- | --- |
| `-n`, `--name` | Container name (required) |
| `-s`, `--server` | Domain to serve over HTTPS |
| `-a`, `--auto` | With `-s`: let Caddy issue and renew the certificate instead of using files in `.caddy/certs/` |
| `-p`, `--port` | Port to serve over plain HTTP |
| `-u`, `--uid` | User ID to run the container as (default: your UID) |
| `-g`, `--gid` | Group ID to run the container as (default: your GID) |
| `-d`, `--deploy-path` | Deployer mode: serve the releases under this deploy path (see [Deploying with Deployer](#deploying-with-deployer)) |
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

| `PHP_VERSION` | Base image (`FROM` in `Dockerfile`) |
| --- | --- |
| not set or empty | `dunglas/frankenphp:1` (latest PHP) |
| `8.5` | `dunglas/frankenphp:1-php8.5` |

- The version must be one FrankenPHP publishes an image for.
- A `PHP_VERSION` already set in your shell takes precedence over `.env`.
- The version is written into `Dockerfile` when it's generated, so re-run `compose` and rebuild after changing it:

  ```bash
  bash bash/compose -n myapp -s myapp.localhost -a
  docker compose up -d
  ```

- "Latest" means the newest base image you have locally. To move to a newer PHP release, rebuild pulling the base image:

  ```bash
  docker compose build --pull
  docker compose up -d
  ```

### PHP extensions and system packages (`Dockerfile.template`)

The image is built from a `Dockerfile` that `compose` renders from `Dockerfile.template`, replacing `{{FRANKENPHP_TAG}}` with the tag for your `PHP_VERSION`. To add PHP extensions (via [`install-php-extensions`](https://github.com/mlocati/docker-php-extension-installer)) or system packages, edit `Dockerfile.template`, not `Dockerfile`, then re-run `compose` and `docker compose up -d`.

Not every extension is available for every PHP version. If the build fails after changing `PHP_VERSION`, check that each extension in `Dockerfile.template` supports that version.

### Project name

Docker Compose groups the container under a project name, shown in `docker compose ls` and Docker Desktop. By default `compose` uses the container name given with `-n`, lowercased, with anything other than letters, digits and `-` (including `_` and `.`) replaced by `-` (for example, `-n My.Shop_API` becomes `my-shop-api`). To use a different name, set `COMPOSE_PROJECT_NAME` in `.env` (see [`.env.example`](.env.example)) and re-run `compose`. A value set in your shell takes precedence over `.env`.

```dotenv
COMPOSE_PROJECT_NAME=myapp
```

If the container is already running, stop it with `docker compose down` **before** changing the name. Otherwise the next `docker compose up -d` treats it as a new project and fails, because a container with the `-n` name already exists.

### Local development with `.localhost` (`-s` with `-a`)

Use a `.localhost` domain such as `myapp.localhost`. It always points to your own machine, so no `/etc/hosts` entry is needed, and Caddy issues the certificate itself from its own local certificate authority (CA), with no internet access required.

```bash
bash bash/compose -n myapp -s myapp.localhost -a
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

1. Run `compose`, e.g. `bash bash/compose -n myapp -s example.com`.
2. Copy your certificate to `.caddy/certs/myapp.pem` and its private key to `.caddy/certs/myapp-key.pem`. The file names must match the `-n` container name. Use the full chain (your certificate followed by the intermediate certificates) in the `.pem` file.
3. Start or restart the container with `docker compose up -d`.

`compose` never creates or overwrites these files; it warns if they're missing. The domain must resolve to the machine running the container.

### Port mode (`-p`)

TLS is disabled in the `Caddyfile`, and the container listens only on the given port. Use this when another proxy terminates TLS in front of it. Caddy trusts proxies on private network ranges (the Docker network or LAN), so its own logs and matchers use the client IP they forward. PHP still sees the proxy as `REMOTE_ADDR` and no `HTTPS` flag. The original values arrive as `X-Forwarded-For` and `X-Forwarded-Proto` headers, so configure your framework to trust them (for example, Laravel's `TrustProxies` middleware).

## Connecting to services on the host

Inside the container, `127.0.0.1` is the container itself, not your machine. To reach a database or another service running on the host, use `host.docker.internal`:

```env
DB_HOST=host.docker.internal
```

Docker Desktop and Rancher Desktop (macOS, Windows) already define that name. On Linux, `compose` adds `extra_hosts: host.docker.internal:host-gateway` to the generated `docker-compose.yml`, which points it at the host's Docker bridge IP (usually `172.17.0.1`). Check it with:

```bash
docker exec -it myapp getent hosts host.docker.internal
```

On a Linux host, the service must also accept connections from the container. For MySQL/MariaDB:

- **Listen on the bridge**, not only on loopback: set `bind-address = 0.0.0.0` (or `127.0.0.1,172.17.0.1` on MySQL ≥ 8.0.13) and restart the service.
- **Grant access from the Docker networks.** The container connects from an address like `172.18.0.x`, not `localhost`:
  ```sql
  CREATE USER 'myapp'@'172.%' IDENTIFIED BY '...';
  GRANT ALL ON myapp.* TO 'myapp'@'172.%';
  ```
- **Open the firewall** to those networks, e.g. with `ufw`: `sudo ufw allow from 172.16.0.0/12 to any port 3306 proto tcp`.

## Deploying with Deployer

[Deployer](https://deployer.org) puts each deploy in a new `releases/N` folder and then points the `current` symlink at it. A container that mounts the project folder would keep serving the old release until it's recreated, which means a few seconds of downtime on every deploy. Deployer mode (`-d <deploy_path>`) avoids that:

- The whole deploy path is mounted in the container, and the app is served from `<deploy_path>/current/public`.
- `bash bash/reload` (run from the new release, **before** `deploy:symlink`) gracefully reloads FrankenPHP so new requests go to the new release's real path. Requests in flight finish on the old one, so there are no errors and no stale code (PHP caches resolved symlink paths for up to 2 minutes, which is why switching `current` alone is not enough).
- Every path in the generated `docker-compose.yml` is the same for all releases, so `docker compose up -d` leaves the running container alone. It only recreates it when the image changes (`Dockerfile.template` or `PHP_VERSION`), the one case with a brief restart.
- Caddy's state lives in `<deploy_path>/shared/.caddy` (certificates survive releases), and the image is built from `<deploy_path>/shared/.frankenphp`, so the build context only contains the `Dockerfile`.

Without `-d` (for example, locally) nothing changes: the project folder is served as is.

The tasks below go in your `deploy.yml`. For a complete Laravel example, see [`deploy.template.yml`](deploy.template.yml), which `bash/dockerize` saves as `deploy.yml` when your project doesn't have one (Laravel recipe, shared `.env`, and the FrankenPHP tasks placed around `deploy:symlink`). Its `artisan` tasks and `deploy:vendors` run with the server's own PHP and Composer, as in the Laravel recipe.

`deploy.yml`:

```yaml
config:
  shared_files:
    - .env    # PHP_VERSION, COMPOSE_PROJECT_NAME and your app settings

tasks:
  frankenphp:release:
    - desc: 'Points the running container to the new release'
    - cd: '{{release_path}}'
    - run: 'bash bash/compose -n myapp -p 8080 -d {{deploy_path}}'
    - run: 'bash bash/reload'

  frankenphp:up:
    - desc: 'Starts the container, or recreates it if the image changed'
    - cd: '{{current_path}}'
    - run: 'docker compose up -d'

before:
  deploy:symlink: frankenphp:release

after:
  deploy:symlink: frankenphp:up
```

Use your own container name and `-p`/`-s` flags. On the first deploy the container doesn't exist yet: `reload` says so and `frankenphp:up` creates it after the symlink. Run tasks that need PHP (e.g. Laravel's `artisan`) inside the container, which sees the deploy path at the same location: `docker exec -w {{release_or_current_path}} myapp php artisan ...`.

### Reloading without recreating the container

`bash bash/reload` gracefully reloads FrankenPHP inside the running container: requests in flight finish, new ones use the reloaded config, and the container is never stopped or recreated. Use it instead of `docker compose restart` or `up --force-recreate`, which drop requests for a few seconds.

| When | Run |
| --- | --- |
| Locally, after editing `Caddyfile` | `bash bash/reload` |
| On the server, from your machine | `dep frankenphp:reload` (task in [`deploy.template.yml`](deploy.template.yml)) |
| On the server, over SSH | `cd <deploy_path>/current && bash bash/reload` |
| After `dep rollback` | runs automatically (`after: rollback: frankenphp:reload`) |

The reload points the server at the real path of the release `current` links to. That's why it must run after a rollback: `dep rollback` only moves the `current` symlink, so without a reload the server keeps serving the newer release.

`docker compose up -d` doesn't recreate the container either unless the image changed (`Dockerfile.template` or `PHP_VERSION`), so it's safe to run on every deploy.

## Notes

- The `tls` line in `Caddyfile` ships commented out. `compose` edits the file in place to uncomment it for your own certificate (`-s`) and comment it again for a Caddy-managed certificate (`-s -a`) or port mode (`-p`), so expect that file to change when you switch modes.
- `docker-compose.yml` and `Dockerfile` are generated output. Re-run `compose` instead of editing them by hand, and add both to your project's `.gitignore`. Keep `Dockerfile.template` in version control, since it holds your extensions.
- Caddy's data, config and certificates are kept under `.caddy/` (gitignored), or `<deploy_path>/shared/.caddy` in Deployer mode, so they survive container restarts.
