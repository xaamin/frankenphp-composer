# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this is

A small, dependency-free Bash toolkit that generates a `docker-compose.yml` for serving a PHP app with FrankenPHP (`dunglas/frankenphp:1-php8.4`). There is no build, lint, or test suite. The files are meant to be copied into a consuming PHP project: consumers download each file from `raw.githubusercontent.com/xaamin/frankenphp-composer/refs/heads/master/...` with a `dockerize` script. **Paths are fetched one by one, so renaming, moving, or adding a file breaks consumers until their download script is updated too.**

## Usage

`compose` is run from the root of the consuming project, and the app is served from that project's `public` folder. It takes a container name (`-n`, required) and exactly one of:
- **`-s <domain>` (TLS mode):** generates local certs for the domain and binds ports 80/443.
- **`-p <port>` (port mode, e.g. behind a reverse proxy):** no TLS; binds only the given port.

Optional flags: `-u`/`-g` override the host UID/GID, and `-v` enables verbose output. After generating the file, the container is started with Docker Compose.

## How the pieces fit

- **`compose`** is the entry point. It parses the flags and exports `NAME`, `DATA_VOLUME`, `HOST_UID`, `HOST_GID`, `SERVER_NAME` and `SERVER_PORT`, then:
  1. **Edits `Caddyfile` in place** with `sed`. Port mode comments out the `tls` line; TLS mode uncomments it and runs `.caddy/generate-ssl-cert`. In-place `sed` editing differs between macOS (BSD) and Linux (GNU), and the script handles both; keep both branches working.
  2. In port mode, sets `SERVER_NAME=":<port>"` so Caddy listens on that port. It also writes a temporary compose override that uses `ports: !override` to replace the default 80/443 bindings.
  3. Renders the final `docker-compose.yml` by merging the template with the optional override via Docker Compose's `config` command. The generated `docker-compose.yml` is gitignored.
- **`docker-compose.t.yml`** is the template. It uses `${VAR}` interpolation, which is resolved at `config` time, and runs the container as the host UID/GID. It bind-mounts the project at `${DATA_VOLUME}`, plus `.caddy/data`, `.caddy/config`, `Caddyfile` → `/etc/frankenphp/Caddyfile`, and `.caddy/certs` → `/etc/caddy/certs` (read-only).
- **`Caddyfile`** reads `{$SERVER_NAME}`, `{$DATA_VOLUME}` and `{$NAME}` from the container environment at runtime. The `tls` cert paths must match the filenames that `generate-ssl-cert` produces (`${NAME}.pem` and `${NAME}-key.pem`).
- **`.caddy/generate-ssl-cert`** generates locally trusted certs for `SERVER_NAME` (installing its tooling if needed, via Homebrew on macOS or `apt` on Linux) and writes them to `./.caddy/certs/`. It depends on the `NAME` and `SERVER_NAME` variables that `compose` exports.

## Gotchas

- `compose` validates `-n/--name` only *after* it has edited the Caddyfile and generated certs. A missing name can therefore produce certs named `.pem`.
- The script's user-facing messages use emoji prefixes (👻 errors, 💡 verbose info, 👀 config, 🌐 next steps). Some comments are in Spanish.
