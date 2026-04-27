# Stack Runtimes

Fork of [Bitpoke stack-runtimes](https://github.com/bitpoke/stack-runtimes) with custom patches for Kubernetes-native DNS resolution.

Published to: `ghcr.io/matthewkennedy/wordpress-runtime`

## Building and Publishing

### Prerequisites

- Docker (OrbStack or Docker Desktop)
- GitHub CLI (`gh`) authenticated with `write:packages` scope

```bash
gh auth refresh --scopes write:packages
```

### Login to GHCR

```bash
gh auth token | docker login ghcr.io -u MatthewKennedy --password-stdin
```

### Build

Build a WordPress runtime image (e.g. WordPress 6.9):

```bash
docker build \
  --build-arg BASE_IMAGE=docker.io/bitpoke/wordpress-runtime:bedrock-php-8.2 \
  -t ghcr.io/matthewkennedy/wordpress-runtime:6.9.1 \
  -f wordpress/Dockerfile-6.9 \
  wordpress
```

The Dockerfile pulls the upstream Bitpoke bedrock image as a base, downloads the specified WordPress version, and overlays our custom templates from `wordpress/docker/`.

### Push

```bash
docker push ghcr.io/matthewkennedy/wordpress-runtime:6.9.1
```

The GHCR package must be set to **public** visibility for the cluster to pull without image pull secrets:
https://github.com/users/MatthewKennedy/packages/container/package/wordpress-runtime

Go to **Package settings** > **Danger Zone** > **Change package visibility** > **Public**.

## Custom Patches

### Configurable Nginx Resolver

The upstream Bitpoke image hardcodes `resolver 8.8.8.8` which cannot resolve Kubernetes cluster-internal DNS names (e.g. `*.svc.cluster.local`).

Our patch makes the resolver configurable via environment variables:

| Variable | Default | Description |
|---|---|---|
| `STACK_RESOLVER` | `8.8.8.8` | DNS server address |
| `STACK_RESOLVER_VALID` | `300` | DNS cache TTL (seconds) |
| `STACK_RESOLVER_TIMEOUT` | `10` | Resolver timeout (seconds) |

For Kubernetes, set `STACK_RESOLVER` to the cluster DNS IP (typically `10.96.0.10`).

Template: `wordpress/docker/templates/nginx/conf.d/20-resolver.conf`

## Versioning

Tag images to match the WordPress version they contain (e.g. `6.9.1`). When updating only the custom patches without changing the WordPress version, append a patch suffix (e.g. `6.9.1-1`).
