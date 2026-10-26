# OpenQDA Docker Deployment

A Docker Compose setup for running [OpenQDA](https://github.com/openqda/openqda), an open-source tool for qualitative data analysis, together with MySQL and Redis.

The image is built directly from the upstream OpenQDA repository at a configurable Git ref, so this repository only contains the deployment tooling, not the application itself.

## Architecture

| Service   | Image / Build          | Purpose                                      |
|-----------|------------------------|----------------------------------------------|
| `openqda` | built from `Dockerfile` (PHP 8.4 CLI) | Laravel app, queue worker and Reverb WebSocket server |
| `mysql`   | `mysql:8.0`            | Database (persisted in `./openqda-mysql`)    |
| `redis`   | `redis:7-alpine`       | Cache / queues (persisted in `./openqda-redis`) |

Exposed ports (defaults): `8923` for the web app and `8443` for WebSockets (Reverb).

TLS is expected to be terminated by a reverse proxy (e.g. Apache) in front of the containers. An example Apache configuration is provided in [reverse-proxy-examples/apache.openqda.example.conf](reverse-proxy-examples/apache.openqda.example.conf). For local testing, [reverse-proxy-examples/certs/](reverse-proxy-examples/certs/) contains self-signed example certificates (`*.example.pem`, not for production use).

## Requirements

- Docker with the Compose plugin (v2)
- A reverse proxy providing HTTPS for production use

## Quick start

```bash
git clone --recurse-submodules <this-repository-url>
cd openqda

cp .env.default .env
# edit .env (see "Configuration" below)

docker compose build
docker compose up -d
```

The application is then available at `http://localhost:8923` (or the `APP_PORT` you configured).

On startup, `entrypoint.sh` generates an `APP_KEY` if none is set, runs database migrations and seeds the database (set `RUN_SEED=false` to skip seeding), and creates the storage link.

## Configuration

All configuration is done via `.env` (copied from `.env.default`, never committed). The most important settings:

| Variable | Description |
|----------|-------------|
| `OPENQDA_GIT_URL`, `OPENQDA_GIT_REF`, `OPENQDA_APP_SUBDIR` | Upstream repository, tag/branch to build, and the app subdirectory (default `web`) |
| `APP_URL`, `ASSET_URL`, `APP_PORT` | Public URL and host port of the app |
| `APP_KEY` | Laravel key. If empty, it is generated at start and **changes on every launch**, invalidating verification links. Set a fixed value in production. |
| `TRUSTED_PROXIES` | Patched into the Laravel `TrustProxies` middleware by `entrypoint.sh` |
| `FORCE_HTTPS` | If `true`, forces HTTPS URL generation behind a TLS-terminating proxy |
| `MAIL_*` | SMTP settings for verification and password-reset mails |
| `ALTCHA_*` | Captcha configuration |
| `REVERB_*`, `VITE_REVERB_*` | WebSocket server settings (server side) and browser-facing settings (build-time) |
| `DB_*`, `MYSQL_ROOT_PASSWORD` | Database credentials |
| `APP_DEBUG`, `LOG_LEVEL` | Debugging; `APP_DEBUG=true` makes `start.sh` run services with verbose output |

Note that the `VITE_REVERB_*` values are baked into the frontend at build time, so rebuild the image after changing them:

```bash
docker compose build --no-cache openqda
docker compose up -d
```

**Change all default passwords and secrets before exposing the instance.**

## Updating OpenQDA

Set `OPENQDA_GIT_REF` in `.env` to the desired upstream tag or branch, then rebuild:

```bash
docker compose build --no-cache openqda
docker compose up -d
```

## Useful commands

```bash
docker compose logs -f openqda     # follow application logs
docker compose ps                  # service and health status
docker compose down                # stop (data in ./openqda-mysql is kept)
```

## CI

`.gitlab-ci.yml` runs an integration test on GitLab: it builds the images, starts the stack with `.env.default`, and waits for MySQL and the app to become healthy.

## Repository layout

```
reverse-proxy-examples/  Example Apache reverse proxy config
  certs/                 Self-signed example certificates (testing only)
Dockerfile          Image build (clones and builds upstream OpenQDA)
docker-compose.yml  App, MySQL and Redis services
entrypoint.sh       Container setup: key, migrations, seeding, proxy patches
start.sh            Starts queue worker, Reverb and the web server
.env.default        Template for configuration
.gitlab-ci.yml      Deployment smoke test
openqda_src/        Upstream OpenQDA (Git submodule)
```

## License

The files in this repository are released under the [MIT License](LICENSE.md).

OpenQDA itself is a separate project with its own license; see the [upstream repository](https://github.com/openqda/openqda) for details.
