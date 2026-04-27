# Stack Runtimes

Fork of Bitpoke WordPress runtime images. Contains Dockerfiles, nginx templates, and PHP configuration for WordPress hosting on Kubernetes.

Published to `ghcr.io/matthewkennedy/wordpress-runtime`. The GHCR package must be public for the cluster to pull without image pull secrets.

## Architecture

- Nginx config is generated at container startup via Dockerize templates in `php/docker/templates/`
- Environment variables set in the WordPress CR `spec.env` are available to both PHP and nginx templates (single container runs both)
- The WordPress operator (separate repo: wordpress-operator) passes `spec.env` to the main container

## Nginx Resolver

The nginx resolver (`php/docker/templates/nginx/conf.d/20-resolver.conf`) controls DNS resolution for upstream connections (memcached, redis, etc).

Configurable via environment variables:
- `STACK_RESOLVER` - DNS server address (default: `8.8.8.8`)
- `STACK_RESOLVER_VALID` - DNS cache TTL in seconds (default: `300`)
- `STACK_RESOLVER_TIMEOUT` - resolver timeout in seconds (default: `10`)

When running in Kubernetes and using cluster-internal DNS names (e.g. `svc.cluster.local`), `STACK_RESOLVER` must be set to the cluster DNS IP (typically `10.96.0.10` for kube-dns). Without this, internal DNS names will fail to resolve because the default `8.8.8.8` cannot resolve cluster-local names.

## Page Cache

Page caching is handled at the nginx level (not WordPress), configured via:
- `STACK_PAGE_CACHE_ENABLED` - enable/disable
- `STACK_PAGE_CACHE_BACKEND` - `memcached` or `redis`
- `STACK_PAGE_CACHE_MEMCACHED_HOST` / `STACK_PAGE_CACHE_MEMCACHED_PORT`
- `STACK_PAGE_CACHE_REDIS_HOST` / `STACK_PAGE_CACHE_REDIS_PORT`

Key template files:
- `php/docker/templates/nginx/vhost-conf.d/page-cache.d/10-index.conf` - Lua-based cache key generation and memcached version lookups
- `php/docker/templates/nginx/vhost-conf.d/75-page-cache-locations.conf` - srcache fetch/store locations using memc_pass or redis_pass

## Building and Publishing

Images are built from `wordpress/Dockerfile-*` files which extend the upstream Bitpoke bedrock base image and overlay custom templates from `wordpress/docker/`.

```bash
# Login
gh auth token | docker login ghcr.io -u MatthewKennedy --password-stdin

# Build (example for WP 6.9)
docker build \
  --build-arg BASE_IMAGE=docker.io/bitpoke/wordpress-runtime:bedrock-php-8.2 \
  -t ghcr.io/matthewkennedy/wordpress-runtime:6.9.1 \
  -f wordpress/Dockerfile-6.9 \
  wordpress

# Push
docker push ghcr.io/matthewkennedy/wordpress-runtime:6.9.1
```

Requires `gh auth refresh --scopes write:packages` if the token lacks `write:packages`.

## Custom Template Overrides

Custom nginx template overrides live in `wordpress/docker/templates/` and are copied over the base image's templates at build time (Dockerfile line: `COPY ./docker /usr/local/docker`). The base templates in `php/docker/templates/` are the upstream originals for reference.
