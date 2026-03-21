# Xahau Devnets

Run multiple Xahau devnet clusters on a single machine with subdomain-based routing via Traefik. Each cluster is isolated by Docker Compose project name.

## Architecture

```
                 ┌──────────────────┐
                 │     Traefik      │
                 │   (80 / 443)     │
                 └────────┬─────────┘
                          │
         ┌────────────────┼────────────────┐
         │                │                │
  *.devnet-a.domain  *.devnet-b.domain    ...
         │                │
    stack devnet-a   stack devnet-b
  (compose -p a)    (compose -p b)
```

Instead of exposing container ports directly on the host, Traefik acts as a reverse proxy and routes traffic based on subdomains. Each stack is started with a unique `COMPOSE_PROJECT_NAME`, producing fully isolated containers and networks.

## Services

Each devnet stack exposes the following services through Traefik:

| Service        | Subdomain                          | Internal Port |
| -------------- | ---------------------------------- | ------------- |
| Faucet         | `faucet.{project}.{BASE_DOMAIN}`   | 8080          |
| Explorer       | `explorer.{project}.{BASE_DOMAIN}` | 4000          |
| Validator List | `vl.{project}.{BASE_DOMAIN}`       | 80            |
| WebSocket      | `{project}.{BASE_DOMAIN}`          | 6018          |
| JSON-RPC       | `rpc.{project}.{BASE_DOMAIN}`      | 5017          |

## Quick Start

### 1. Install tools

```sh
mise install
```

This installs Python 3.10 and [xrpld-netgen](https://github.com/tequdev/xrpld-network-gen.git).

### 2. Configure environment

Copy the sample config and edit it:

```sh
cp mise.local.sample.toml mise.local.toml
```

Set `BASE_DOMAIN` and `ACME_EMAIL` in `mise.local.toml`.

### 3. Generate a network

```sh
xrpld-netgen create:network --protocol xahau --name <name> --build_version <version>
```

This creates a cluster configuration under `./workspace/<name>-cluster/`.

### 4. Create the shared proxy network

```sh
docker network create proxy
```

### 5. Start Traefik

```sh
mise run traefik
```

### 6. Start a devnet stack

```sh
docker compose \
  -f ./workspace/<name>-cluster/docker-compose.yml \
  -f ./compose.override.yml \
  up -d
```

Replace `<name>` with a unique project name (e.g. `devnet-a`) and `<version>` with the build version used in step 3.

To run a second instance, repeat step 3 with a different name, version:

```sh
docker compose \
  -f ./workspace/<name-b>-cluster/docker-compose.yml \
  -f ./compose.override.yml \
  up -d
```

## Environment Variables

| Variable               | Description                                | Default         |
| ---------------------- | ------------------------------------------ | --------------- |
| `BASE_DOMAIN`          | Base domain for subdomain routing          | `example.com`   |
| `ACME_EMAIL`           | Email for Let's Encrypt registration       | `admin@example.com` |

## Production (HTTPS with Let's Encrypt)

- Point a wildcard DNS record (`*.{BASE_DOMAIN}`) to your server.
- Set `BASE_DOMAIN` to the real domain.
- Set `ACME_EMAIL` to a valid email address.
- Certificates are automatically obtained via TLS challenge and stored in a Docker volume.

## Local Development

Set `BASE_DOMAIN=127.0.0.1.nip.io` in `mise.local.toml`. All subdomains resolve to `127.0.0.1` via [nip.io](https://nip.io), so services are accessible via HTTP on port 80 without certificates.

Example URLs for a stack named `devnet`:

- `http://explorer.devnet.127.0.0.1.nip.io`
- `http://faucet.devnet.127.0.0.1.nip.io`
- `ws://devnet.127.0.0.1.nip.io`

## Repository Structure

```
.
├── compose.override.yml        # Removes direct ports, adds Traefik labels and proxy network
├── compose.traefik.yml         # Traefik reverse proxy with Let's Encrypt ACME
├── mise.toml                   # Tool definitions (python, xrpld-netgen) and tasks
├── mise.local.sample.toml      # Sample local env config
└── workspace/                  # Generated network configs (gitignored)
```
