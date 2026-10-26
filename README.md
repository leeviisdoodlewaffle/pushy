# Pushy — Technical Specification

## Overview

Pushy is a Rust CLI that automates a deployment workflow:
1. Upload a local directory to a remote host (SSH/SFTP, optional rsync).
2. Trigger `docker compose up` remotely.
3. Deploy a Docker image pre-configured with Cloudflare settings.

## Architecture

```
┌────────────┐      clap       ┌─────────────┐
│   main.rs  │ ─────────────▶ │   cli.rs    │
└─────┬──────┘                └──────┬──────┘
      │                             │
      ▼                             ▼
┌────────────┐   SSH/SFTP    ┌─────────────┐
│  ssh.rs    │◀─────────────▶│  transfer.rs │  (SFTP recursive upload / rsync)
└─────┬──────┘               └─────────────┘
      │ exec channel
      ▼
┌────────────┐   docker       ┌─────────────┐
│ deploy.rs  │──────────────▶│ remote host  │
│ (pipeline) │               │ compose up   │
└────────────┘               └─────────────┘
```

### Transport layer (`ssh.rs`)
- TCP connect with read/write timeouts.
- `ssh2::Session` handshake.
- Auth: public-key (`userauth_pubkey_file`) or password (`userauth_password`), key defaults to `~/.ssh/id_rsa`.
- Exposes `exec(cmd) -> (exit_code, stdout, stderr)` and `sftp()`.

### Transfer layer (`transfer.rs`)
- SFTP recursive directory upload using `walkdir`, with `mkdir -p` semantics via `ensure_remote_dir`.
- Optional `rsync -az --delete` subprocess for incremental transfers.

### Deployment pipeline (`deploy.rs`)
1. Upload local dir (unless `--skip-transfer`).
2. Inject Cloudflare env (`.env`) when token/zone/subdomain provided.
3. `docker pull <image>` (unless `--skip-pull`).
4. `docker compose -f <file> up -d --remove-orphans`.
5. Print `docker compose ps`.

### Cloudflare-ready image
The image is supplied via `--image` / `PUSHY_IMAGE`. Cloudflare credentials are written to the remote `.env` and consumed by the compose service (e.g. a Cloudflare tunnel sidecar). The image itself is expected to ship with the Cloudflare client pre-installed.

## Library stack

| Concern | Crate |
|---|---|
| CLI parsing | `clap` (derive, env) |
| SSH/SFTP | `ssh2` (libssh2 bindings) |
| Async (optional) | `tokio` (not required; ssh2 is sync) |
| Config files | `toml` + `serde` |
| Errors | `thiserror` + `anyhow` |
| Traversal | `walkdir` |
| Logging | `tracing` / `tracing-subscriber` |
| Paths | `dirs`, `shellexpand` |

## Command syntax

```
pushy deploy \
  --host 203.0.113.10 \
  --user deploy \
  --port 22 \
  --key ~/.ssh/id_rsa \
  --local ./app \
  --remote /opt/pushy/app \
  --image ghcr.io/example/app:latest \
  --compose-file docker-compose.yml \
  --cloudflare-token "$CLOUDFLARE_API_TOKEN" \
  --cloudflare-zone example.com \
  --cloudflare-subdomain app \
  --service app

pushy transfer --host ... --local ./app --remote /opt/pushy/app [--rsync]
pushy exec --host ... -- docker compose ps
pushy check --host ...
```

Environment variables: `PUSHY_HOST`, `PUSHY_USER`, `PUSHY_PORT`, `PUSHY_KEY`, `PUSHY_PASSWORD`, `PUSHY_IMAGE`, `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ZONE`, `CLOUDFLARE_SUBDOMAIN`.

## Implementation plan

1. Bootstrap crate (`cargo new pushy`), add dependencies.
2. Implement `error.rs` + `ssh.rs` (connect, auth, exec, sftp).
3. Implement `transfer.rs` (SFTP recursive upload + rsync fallback).
4. Implement `deploy.rs` pipeline + Cloudflare env injection.
5. Wire `cli.rs` subcommands in `main.rs`.
6. Add `config.rs` for optional `pushy.toml`.
7. Tests: unit tests for path joining / env rendering; integration smoke against a local SSH container.
8. Release build (`cargo build --release`), package.

## Build

```
cargo build --release
./target/release/pushy --help
```
