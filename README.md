# oci-mariadb

[![CI](https://github.com/joeckr/oci-mariadb/actions/workflows/build.yml/badge.svg)](https://github.com/joeckr/oci-mariadb/actions/workflows/build.yml)
[![Helm](https://github.com/joeckr/oci-mariadb/actions/workflows/helm.yml/badge.svg)](https://github.com/joeckr/oci-mariadb/actions/workflows/helm.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A hardened, rootless MariaDB container image and Helm chart designed for **Red Hat OpenShift (OCP)**, **OKD**, and rootless Kubernetes environments running under strict Security Context Constraints (SCC).

---

## Features

- **Rootless & Arbitrary UID Ready**: Built in compliance with OpenShift guidelines (`restricted` / `restricted-v2` SCCs). Directories are assigned group ownership `gid=0` (`root`) with `g+rwX` permissions, allowing any arbitrary runtime UID to read and write database files.
- **Red Hat UBI Base**: Built upon the official upstream `mariadb:<version>-ubi` Red Hat Universal Base Images for enterprise security, stability, and consistent patching.
- **Multi-Version Matrix Builds**: Automated GitHub Actions matrix builds producing multi-arch (`linux/amd64`, `linux/arm64`) images for multiple major MariaDB releases defined in [`versions.json`](file:///Users/josephking/Code/oci/oci-mariadb/versions.json).
- **Production Helm Chart**: Fully configurable Helm chart under [`chart/`](file:///Users/josephking/Code/oci/oci-mariadb/chart) with persistent storage (PVC), custom environment variables, and schema initialization support.
- **Schema Initialization**: Automatically bootstrap database schemas on first startup by mounting SQL scripts into `/docker-entrypoint-initdb.d/`.
- **Developer Experience**: Integrated with [`mise`](https://mise.jdx.dev/) and [`prek`](https://github.com/jdx/prek) for one-command environment setup, linting, security audits, and container orchestration.

---

## Supported Versions

Images are published to GitHub Container Registry (GHCR) under `ghcr.io/joeckr/mariadb`. Supported versions are configured via [`versions.json`](file:///Users/josephking/Code/oci/oci-mariadb/versions.json):

| Major Version | Upstream Version | Image Tags | Status |
|---|---|---|---|
| **12** | `12.3` | `12`, `12.3`, `latest` | Active (Default) |
| **11** | `11.8` | `11`, `11.8` | Supported |
| **10** | `10.11` | `10`, `10.11` | Supported |

> Need an unlisted version? Add it to [`versions.json`](file:///Users/josephking/Code/oci/oci-mariadb/versions.json) and open a pull request!

---

## Quickstart

### Local Development (Docker Compose)

The repository includes a [`docker-compose.yml`](file:///Users/josephking/Code/oci/oci-mariadb/docker-compose.yml) configured with persistent volume storage and automatic schema seeding via [`schema.sql`](file:///Users/josephking/Code/oci/oci-mariadb/schema.sql).

Start the stack:

```bash
# Using mise
mise run compose

# Or using docker compose directly
docker compose up -d --build
```

Stop the stack:

```bash
# Using mise
mise run down

# Or using docker compose directly
docker compose down
```

**Default Credentials:**
- **Port:** `3306`
- **Database:** `mariadb`
- **User:** `mariadb`
- **Password:** `replaceme`
- **Root Password:** `really_replaceme`

### Running via Docker CLI

```bash
docker run -d \
  --name mariadb \
  -p 3306:3306 \
  -e MARIADB_DATABASE=app \
  -e MARIADB_USER=appuser \
  -e MARIADB_PASSWORD=secret \
  -e MARIADB_ROOT_PASSWORD=supersecret \
  -v mariadb_data:/var/lib/mysql \
  ghcr.io/joeckr/mariadb:latest
```

---

## OpenShift & Kubernetes Deployment (Helm)

The [`chart/`](file:///Users/josephking/Code/oci/oci-mariadb/chart) directory contains a production-ready Helm chart tailored for OpenShift and Kubernetes.

### Installing the Chart

```bash
helm upgrade --install mariadb ./chart \
  --namespace mariadb \
  --create-namespace \
  --set mariadb.rootPassword="your-strong-root-password" \
  --set mariadb.password="your-strong-user-password"
```

### Chart Configuration

Key configuration parameters in [`chart/values.yaml`](file:///Users/josephking/Code/oci/oci-mariadb/chart/values.yaml):

| Parameter | Description | Default |
|---|---|---|
| `mariadb.name` | Deployment and service name | `mariadb` |
| `mariadb.image` | Container image repository | `ghcr.io/joeckr/mariadb` |
| `mariadb.tag` | Container image tag | `latest` |
| `mariadb.replicaCount` | Number of replicas | `1` |
| `mariadb.port` | MariaDB port | `3306` |
| `mariadb.db` | Default database to create | `mariadb` |
| `mariadb.user` | Database user account | `mariadb` |
| `mariadb.password` | Password for user account | `changeme` |
| `mariadb.rootPassword` | Root administrative password | `really_replaceme` |
| `mariadb.storageSize` | PVC storage request | `1Gi` |
| `mariadb.storageAccessMode` | PVC access mode | `ReadWriteOnce` |
| `mariadb.storageClass` | PVC StorageClass (`""` uses cluster default) | `""` |
| `initSchema.enabled` | Mount and execute `schema.sql` on first boot | `true` |

---

## Database Initialization (`schema.sql`)

MariaDB executes scripts found in `/docker-entrypoint-initdb.d/` on first startup when the database directory is empty.

- **Local Development:** [`docker-compose.yml`](file:///Users/josephking/Code/oci/oci-mariadb/docker-compose.yml) binds [`schema.sql`](file:///Users/josephking/Code/oci/oci-mariadb/schema.sql) directly into `/docker-entrypoint-initdb.d/schema.sql:z`.
- **Helm Chart:** When `initSchema.enabled: true`, the chart creates a ConfigMap from [`chart/config/schema.sql`](file:///Users/josephking/Code/oci/oci-mariadb/chart/config/schema.sql) and mounts it into the container.
- If you do not require initialization schemas, disable it via `--set initSchema.enabled=false` or remove the volume mount from `docker-compose.yml`.

---

## OpenShift Security Model

Standard MariaDB images often fail on OpenShift clusters due to the default `restricted` SCC, which executes containers under arbitrary, dynamically assigned UIDs.

This image resolves this by:
1. Creating required directories upfront (`/var/lib/mysql`, `/run/mariadb`, `/var/log/mysql`, `/docker-entrypoint-initdb.d`).
2. Setting group ownership to GID `0` (`root`) via `chgrp -R 0`.
3. Granting full read/write/execute permissions to group members via `chmod -R g+rwX`.
4. Specifying a numeric UID in the Dockerfile (`USER 0` during build setup, `USER 1031` default runtime), ensuring compatibility with both standard Docker hosts and arbitrary UID assignment on OpenShift.

---

## Development & Maintenance

This repository utilizes [`mise`](https://mise.jdx.dev/) for developer toolchain management and [`prek`](https://github.com/jdx/prek) for pre-commit validation.

### Setup

```bash
# Install toolchain and pre-commit hooks
mise run install
```

### Available Tasks

Run tasks with `mise run <task>`:

| Task | Description | Command |
|---|---|---|
| `install` | Install tools and set up pre-commit hooks | `prek install` |
| `prek` | Run all linters and pre-commit hooks | `prek run --all-files` |
| `compose` | Start the local Docker Compose stack | `docker compose up -d --build` |
| `down` | Stop the local Docker Compose stack | `docker compose down` |
| `build` | Build the container image locally with buildx | `docker buildx build ...` |
| `trivy-fs` | Scan the repository filesystem for security vulnerabilities | `trivy fs .` |
| `trivy-image` | Scan the built container image with Trivy | `trivy image ...` |

### Linters & Quality Checks

The CI pipeline runs automated checks on every push and PR:
- **Hadolint**: Dockerfile linting and best practices (`DL3066` numeric user ID compliance).
- **Actionlint & Zizmor**: GitHub Actions workflow syntax and security audits.
- **Shellcheck**: Shell script analysis.
- **Helm Lint**: Helm chart validation.
- **Gitleaks**: Secrets detection.

---

## License

This project is licensed under the [MIT License](file:///Users/josephking/Code/oci/oci-mariadb/LICENSE).
