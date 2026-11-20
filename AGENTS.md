# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this is

A small, dependency-free Bash toolkit that generates a `docker-compose.yml` for serving a PHP app with FrankenPHP (`dunglas/frankenphp:1-php${PHP_VERSION}`, or `dunglas/frankenphp:1` = latest PHP when `PHP_VERSION` is empty). There is no build, lint, or test suite. The files are meant to be copied into a consuming PHP project: consumers download each file from `raw.githubusercontent.com/xaamin/frankenphp-composer/refs/heads/master/...` with a `dockerize` script. **Paths are fetched one by one, so renaming, moving, or adding a file breaks consumers until their download script is updated too.**

## Usage

`compose` is run from the root of the consuming project, and the app is served from that project's `public` folder. It takes a container name (`-n`, required) and exactly one of:
- **`-s <domain>` (TLS mode):** generates local certs for the domain and binds ports 80/443.
- **`-s <domain> -a` (auto mode):** Caddy issues the certificate itself: via ACME for public domains, or from its internal local CA for `localhost`/`*.localhost` (the recommended dev setup; its root is at `.caddy/data/caddy/pki/authorities/local/root.crt` and must be trusted on the host because of `skip_install_trust`). No cert is generated into `.caddy/certs/`. `-a` without `-s` is an error.
- **`-p <port>` (port mode, e.g. behind a reverse proxy):** no TLS; binds only the given port.

Optional flags: `-u`/`-g` override the host UID/GID, and `-v` enables verbose output. The PHP version comes from `PHP_VERSION` in `.env`, not a flag; when it is empty the template drops the `-php` suffix via `${PHP_VERSION:+-php}`. After generating the file, the container is started with Docker Compose.

## How the pieces fit

- **`compose`** is the entry point. It parses the flags and exports `NAME`, `DATA_VOLUME`, `HOST_UID`, `HOST_GID`, `SERVER_NAME`, `SERVER_PORT` and `PHP_VERSION` (shell env, else the last `PHP_VERSION=` line in the project `.env`, else empty; always exported, even when empty, to avoid a Compose "variable is not set" warning), then:
  1. **Edits `Caddyfile` in place** with `sed`. The `tls` line is committed commented out (the safe default for port mode). Port mode and auto mode comment out the `tls` line; local-cert TLS mode uncomments it and runs `.caddy/generate-ssl-cert`. In-place `sed` editing differs between macOS (BSD) and Linux (GNU), and the script handles both; keep both branches working.
  2. In port mode, sets `SERVER_NAME=":<port>"` so Caddy listens on that port. It also writes a temporary compose override that uses `ports: !override` to replace the default 80/443 bindings.
  3. Renders the final `docker-compose.yml` by merging the template with the optional override via Docker Compose's `config` command. The generated `docker-compose.yml` is gitignored.
- **`docker-compose.t.yml`** is the template. It uses `${VAR}` interpolation, which is resolved at `config` time, and runs the container as the host UID/GID. It bind-mounts the project at `${DATA_VOLUME}`, plus `.caddy/data`, `.caddy/config`, `Caddyfile` → `/etc/frankenphp/Caddyfile`, and `.caddy/certs` → `/etc/caddy/certs` (read-only).
- **`Caddyfile`** reads `{$SERVER_NAME}`, `{$DATA_VOLUME}` and `{$NAME}` from the container environment at runtime. The `tls` cert paths must match the filenames that `generate-ssl-cert` produces (`${NAME}.pem` and `${NAME}-key.pem`).
- **`.caddy/generate-ssl-cert`** generates locally trusted certs for `SERVER_NAME` (installing its tooling if needed, via Homebrew on macOS or `apt` on Linux) and writes them to `./.caddy/certs/`. It depends on the `NAME` and `SERVER_NAME` variables that `compose` exports.

## Gotchas

- The script's user-facing messages use emoji prefixes (👻 errors, 💡 verbose info, 🔒/🔓 TLS mode, 👀 config, 🌐 next steps). Some comments are in Spanish.
