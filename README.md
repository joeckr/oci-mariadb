# oci-mariadb

[![CI](https://github.com/joeckr/oci-mariadb/actions/workflows/build.yml/badge.svg)](https://github.com/joeckr/oci-mariadb/actions/workflows/build.yml)
[![Helm](https://github.com/joeckr/oci-mariadb/actions/workflows/helm.yml/badge.svg)](https://github.com/joeckr/oci-mariadb/actions/workflows/helm.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A hardened, rootless MariaDB container image and Helm chart designed for **Red Hat OpenShift (OCP)**, **OKD**, and rootless Kubernetes environments running under strict Security Context Constraints (SCC).

---

## Features

- **Rootless & Arbitrary UID Ready**: Built in compliance with OpenShift guidelines (`restricted` / `restricted-v2` SCCs). Directories are assigned group ownership `gid=0` (`root`) with `g+rwX` permissions, allowing any arbitrary runtime UID to read and write database files.
- **Red Hat UBI Base**: Built upon the official upstream `mariadb:<version>-ubi` Red Hat Universal Base Images for enterprise security, stability, and consistent patching.
- **Multi-Version Matrix Builds**: Automated GitHub Actions matrix builds producing multi-arch (`linux/amd64`, `linux/arm64`) images for multiple major MariaDB releases defined in [`versions.json`](versions.json).
- **Production Helm Chart**: Fully configurable Helm chart under [`chart/`](chart/) with persistent storage (PVC), custom environment variables, and schema initialization support.
- **Schema Initialization**: Automatically bootstrap database schemas on first startup by mounting SQL scripts into `/docker-entrypoint-initdb.d/`.
- **Developer Experience**: Integrated with [`mise`](https://mise.jdx.dev/) and [`hk`](https://github.com/jdx/hk) for one-command environment setup, linting, security audits, and container orchestration.

---

## Supported Versions

Images are published to GitHub Container Registry (GHCR) under `ghcr.io/joeckr/mariadb`. Supported versions are configured via [`versions.json`](versions.json):

| Major Version | Upstream Version | Image Tags | Status |
|---|---|---|---|
| **12** | `12.3` | `12`, `12.3`, `latest` | Active (Default) |
| **11** | `11.8` | `11`, `11.8` | Supported |
| **10** | `10.11` | `10`, `10.11` | Supported |

> Need an unlisted version? Add it to [`versions.json`](versions.json) and open a pull request!

---

## Local Environment & Podman Setup

To ensure containerized applications and Helm charts tested locally run cleanly when deployed to OpenShift or Kubernetes, this repository is designed to be used alongside the Podman configuration in [joeckr/dotfiles](https://github.com/joeckr/dotfiles).

The dotfiles repository provides a centralized [`containers.conf`](https://github.com/joeckr/dotfiles/blob/main/containers/containers.conf) (deployed to `~/.config/containers/containers.conf`) that configures Podman to simulate OpenShift's default **`restricted-v2` Security Context Constraints (SCC)**:

| OpenShift SCC Rule | Podman Configuration | Description |
|---|---|---|
| **Random UID (`MustRunAsRange`)** | `userns = "auto"` | Allocates dynamic subordinate UID/GID ranges from `/etc/subuid` and `/etc/subgid`. Containers run unprivileged without mapping host root. |
| **Drop Capabilities** | `default_capabilities = ["NET_BIND_SERVICE"]` | Drops standard root capabilities (`CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `SETUID`, `SETGID`, `SYS_CHROOT`, etc.) and permits only `NET_BIND_SERVICE`. |
| **Disallow Privileged** | `privileged = false` | Disallows privileged container execution by default. |
| **Seccomp Profile** | `seccomp_profile = "/usr/share/containers/seccomp.json"` | Enforces the runtime default seccomp profile (`RuntimeDefault`). |
| **Namespace Isolation** | `cgroupns`, `ipcns`, `pidns`, `utsns = "private"` | Enforces private container namespaces (host namespaces are forbidden in restricted SCC). |

### macOS Podman Machine Integration

On macOS, the dotfiles installer script (`brew/podman.sh`) automates the machine lifecycle:

1. Deploys `containers/containers.conf` to `~/.config/containers/containers.conf` on the host.
2. Initializing `podman machine init` automatically mounts `~/.config/containers` into `/etc/containers` inside the Fedora CoreOS VM.
3. Automatically symlinks `/etc/containers/containers.conf` to the VM user's config (`~core/.config/containers/containers.conf`) and restarts the Podman API service so all container executions immediately enforce these constraints.

## Testing with Podman Compose

Two Compose configurations are provided to facilitate testing, benchmarking, and debugging:

### 1. Upstream Baseline (`compose.upstream.yml`)

The [`compose.upstream.yml`](compose.upstream.yml) file runs the original, unmodified upstream container image (`mariadb:12-ubi`):

```sh
# Start upstream container
podman compose -f compose.upstream.yml up -d
```

**Why test upstream?**
Running the unmodified image against your SCC-compliant Podman setup simulates deploying standard public images directly into OpenShift. This will typically surface common failures:
- Processes attempting to run as `root` (UID 0) or user `mysql` (UID 999) without arbitrary UID flexibility.
- Inability to write to system, run, or database directories without proper group 0 (`root` group) permissions.
- Disallowed operations and capability rejections under `restricted-v2` SCC.

### 2. Modified Image (`compose.yml`)

The [`compose.yml`](compose.yml) file builds and runs the customized `Dockerfile` containing the adaptations required for OpenShift and rootless environments:

```sh
# Build and start the modified compliant container
podman compose up -d --build

# Or via mise
mise run compose
```

This verified configuration applies:
- Non-root user execution (`USER 1031`).
- Root group ownership (`chgrp -R 0`) and group read/write permissions (`chmod -R g+rwX`) on `/var/lib/mysql`, `/run/mariadb`, `/var/log/mysql`, and `/docker-entrypoint-initdb.d`.
- Persistent volume storage via `mdbdata`.
- Automatic schema initialization by mounting [`schema.sql`](schema.sql) into `/docker-entrypoint-initdb.d/schema.sql:z`.

**Default Credentials:**
- **Port:** `3306`
- **Database:** `mariadb`
- **User:** `mariadb`
- **Password:** `replaceme`
- **Root Password:** `really_replaceme`

### Stopping Containers

```sh
# Stop modified compose stack
podman compose down
# or: mise run down

# Stop upstream compose stack
podman compose -f compose.upstream.yml down

# View logs
podman compose logs -f
# or: mise run logs
```

### Local Helm Testing (Podman Play Kube)

Test rendered Helm chart manifests directly in Podman without requiring a remote cluster:

```sh
# Render Helm template and run pods locally via podman play kube
mise run play

# Tear down the local podman pods
mise run downplay
```

---

## OpenShift & Kubernetes Deployment (Helm)

The [`chart/`](chart/) directory contains a production-ready Helm chart tailored for OpenShift and Kubernetes.

### Installing the Chart

```bash
helm upgrade --install mariadb ./chart \
  --namespace mariadb \
  --create-namespace \
  --set mariadb.rootPassword="<your-strong-root-password>" \
  --set mariadb.password="<your-strong-user-password>"
```

### Chart Configuration

Key configuration parameters in [`chart/values.yaml`](chart/values.yaml):

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

- **Local Development:** [`compose.yml`](compose.yml) binds [`schema.sql`](schema.sql) directly into `/docker-entrypoint-initdb.d/schema.sql:z`.
- **Helm Chart:** When `initSchema.enabled: true`, the chart creates a ConfigMap from [`chart/config/schema.sql`](chart/config/schema.sql) and mounts it into the container.
- If you do not require initialization schemas, disable it via `--set initSchema.enabled=false` or remove the volume mount from `compose.yml`.

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

This repository utilizes [`mise`](https://mise.jdx.dev/) for developer toolchain management and [`hk`](https://github.com/jdx/hk) for git hooks and linting.

### Setup

```bash
# Install toolchain and set up git hooks
mise run install
```

### Available Tasks

Run tasks with `mise run <task>`:

| Task | Description | Command |
|---|---|---|
| `install` | Install tools and set up git hooks | `hk install --mise` |
| `hk` (or `check`) | Run all linters and hook checks | `hk check --all` |
| `compose` | Start local container stack with Podman Compose | `podman compose up -d --build` |
| `down` | Stop local Podman Compose stack | `podman compose down` |
| `logs` | View Podman Compose logs | `podman compose logs -f` |
| `play` | Test Helm chart manifests locally with Podman Play Kube | `podman play kube rendered.yaml` |
| `downplay` | Stop and remove Podman Play Kube pods | `podman play kube rendered.yaml --down` |
| `helm-lint` | Lint Helm chart | `helm lint chart/` |
| `helm-template` | Render Helm chart templates to `rendered.yaml` | `helm template test chart/ > rendered.yaml` |
| `helm-dep` | Build Helm chart dependencies | `helm dependency build chart/` |
| `build` | Build container image locally with Podman Buildx | `podman buildx build --platform linux/amd64 -t ghcr.io/joeckr/oci-modified:test . --load` |
| `trivy-fs` | Scan repository filesystem for security vulnerabilities | `trivy fs .` |
| `trivy-image` | Scan built container image with Trivy | `trivy image ghcr.io/joeckr/oci-modified:test` |

### Linters & Quality Checks

The CI pipeline and `hk` run automated checks on every push and PR:
- **Hadolint**: Dockerfile linting and best practices (`DL3066` numeric user ID compliance).
- **Actionlint & Zizmor**: GitHub Actions workflow syntax and security audits.
- **Shellcheck**: Shell script analysis.
- **Yamllint**: YAML syntax and formatting validation.
- **Tombi**: TOML linting and formatting.
- **Helm Lint**: Helm chart validation.
- **Betterleaks**: Secrets detection.

---

## Support

If you find this project useful, consider supporting my work on [Ko-fi](https://ko-fi.com/joeckr):

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/joeckr)

## License

Please refer to the `LICENSE` file for details.
