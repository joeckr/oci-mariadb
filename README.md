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

## Security & Compliance Architecture

Both OpenShift and Talos Linux prioritize workload security and least privilege, but they enforce and evaluate constraints through different mechanisms. This repository is architected to satisfy both environments without code changes.

### OpenShift Compliance (`restricted-v2` SCC)

OpenShift uses **Security Context Constraints (SCC)** to control pod permissions. Under the default `restricted-v2` SCC:
- **Arbitrary Dynamic UIDs**: OpenShift assigns a random UID from a dedicated per-namespace range (e.g., `1000670000`). Containers cannot assume a fixed UID like `1000`.
- **Root Group (GID 0)**: Files and directories required at runtime (`/var/lib/mysql`, `/run/mariadb`, `/var/log/mysql`, `/docker-entrypoint-initdb.d`) are owned by group 0 (`chgrp -R 0`) with group read/write permissions (`chmod -R g+rwX`) so the dynamically assigned UID can access them.
- **Dropped Capabilities**: Drops standard root capabilities (`CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `SETUID`, `SETGID`, `SYS_CHROOT`, etc.) and permits only unprivileged operations (and `NET_BIND_SERVICE` when needed).
- **Unprivileged Ports**: Listens on standard port `3306` without requiring elevated privileges.

### Talos Linux Compliance (Kubernetes PSS `restricted`)

Talos Linux is an immutable, minimal, secure-by-default Kubernetes operating system with no SSH, no interactive shell, and an immutable root filesystem. In Talos clusters:
- **Pod Security Standards (PSS)**: Workload namespaces enforce the Kubernetes **Pod Security Admission (PSA)** `restricted` profile.
- **Must Run As Non-Root**: The pod specification must set `securityContext.runAsNonRoot: true`. Containers cannot execute as UID 0.
- **Drop All Capabilities**: The container specification explicitly drops all Linux capabilities (`capabilities: drop: ["ALL"]`).
- **Disallow Privilege Escalation**: Sets `securityContext.allowPrivilegeEscalation: false` to prevent child processes from acquiring more privileges than the parent.
- **Seccomp Profile**: Pods enforce `seccompProfile: { type: RuntimeDefault }`.
- **Credential Protection**: Best practice sets `automountServiceAccountToken: false` to avoid leaking Kubernetes API tokens to database containers.
- **Persistent Storage**: Integrates with CSI storage providers (e.g., Local Path Provisioner, OpenEBS Mayastor, Rook-Ceph) via configurable PVC StorageClass.

### Rootless Build Environment Compliance

Building container images inside secure or unprivileged environments (such as rootless Podman/Buildah on developer workstations, or unprivileged Kubernetes CI runners like Tekton or Kaniko) requires that the build process itself does not rely on host `root` privileges or the legacy root-owned Docker daemon socket (`/var/run/docker.sock`).

This repository's `Dockerfile` is engineered for complete rootless build support:
- **No Host Root Required**: Builds execute and succeed cleanly under unprivileged user namespaces without needing `sudo` or privileged container builders.
- **User Namespace Friendly Permissions**: Layer modifications rely on `chgrp -R 0` and group-based permissions (`g+rwX`), which map cleanly into subordinate UID/GID allocations (`/etc/subuid` and `/etc/subgid`) without failing on host-restricted `chown` operations.
- **Unprivileged Local Build**: Run `mise run build` (`podman buildx build --platform linux/amd64 -t ghcr.io/joeckr/mariadb:test . --load`) or `mise run compose` to build locally without root escalation.

### Compliance Matrix

| Security Dimension | OpenShift (`restricted-v2` SCC) | Talos Linux (Kubernetes PSS `restricted`) | Implementation in This Repo |
|---|---|---|---|
| **Build Execution** | Rootless builder compatible | Rootless builder compatible | Builds unprivileged via rootless Podman/Buildah (`mise run build`) |
| **User ID** | Dynamic arbitrary UID (`MustRunAsRange`) | Non-root UID (`runAsNonRoot: true`) | `USER 1031` in Dockerfile + `runAsNonRoot: true` in Helm |
| **Group Permissions** | Requires GID 0 (`root`) with `g+rwX` | Compatible with GID 0 / unprivileged groups | `chgrp -R 0` & `chmod -R g+rwX` on runtime paths |
| **Capabilities** | Drops root caps; allows `NET_BIND_SERVICE` | Must drop `ALL` capabilities | `capabilities.drop: ["ALL"]` in Helm chart |
| **Privilege Escalation** | Prohibited | `allowPrivilegeEscalation: false` | Configured in Helm `securityContext` |
| **Seccomp Profile** | `RuntimeDefault` | `RuntimeDefault` or `Localhost` | `seccompProfile: { type: RuntimeDefault }` |
| **Service Account Token** | Optional | Recommended disabled | Hardened in pod configuration |
| **Port Binding** | Unprivileged (> 1024) | Unprivileged (> 1024) | Listens on port `3306` |
| **Storage Layer** | OpenShift StorageClass | Talos CSI StorageClass | Standard PVC template with configurable `storageClass` |

---

## Local Environment & Podman Setup

To ensure containerized applications and Helm charts tested locally run cleanly when deployed to OpenShift or Talos Linux, this repository is designed to be used alongside the Podman configuration in [joeckr/dotfiles](https://github.com/joeckr/dotfiles).

The dotfiles repository provides a centralized [`containers.conf`](https://github.com/joeckr/dotfiles/blob/main/containers/containers.conf) (deployed to `~/.config/containers/containers.conf`) that configures Podman to simulate OpenShift and Talos Linux runtime restrictions:

| Security Rule | Podman Configuration | Description |
|---|---|---|
| **Random UID (`MustRunAsRange`)** | `userns = "auto"` | Allocates dynamic subordinate UID/GID ranges from `/etc/subuid` and `/etc/subgid`. Containers run unprivileged without mapping host root. |
| **Drop Capabilities** | `default_capabilities = ["NET_BIND_SERVICE"]` | Drops standard root capabilities (`CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `SETUID`, `SETGID`, `SYS_CHROOT`, etc.) and permits only `NET_BIND_SERVICE`. |
| **Disallow Privileged** | `privileged = false` | Disallows privileged container execution by default. |
| **Seccomp Profile** | `seccomp_profile = "/usr/share/containers/seccomp.json"` | Enforces the runtime default seccomp profile (`RuntimeDefault`). |
| **Namespace Isolation** | `cgroupns`, `ipcns`, `pidns`, `utsns = "private"` | Enforces private container namespaces (host namespaces are forbidden in restricted profiles). |

### macOS Podman Machine Integration

On macOS, the dotfiles installer script (`brew/podman.sh`) automates the machine lifecycle:

1. Deploys `containers/containers.conf` to `~/.config/containers/containers.conf` on the host.
2. Initializing `podman machine init` automatically mounts `~/.config/containers` into `/etc/containers` inside the Fedora CoreOS VM.
3. Automatically symlinks `/etc/containers/containers.conf` to the VM user's config (`~core/.config/containers/containers.conf`) and restarts the Podman API service so all container executions immediately enforce these constraints.

---

## Testing & Validation Process

This repository defines a 4-tier testing process to validate container security, manifest generation, and runtime compatibility from local development through to production cluster deployment.

```
┌─────────────────────────┐     ┌─────────────────────────┐     ┌─────────────────────────┐     ┌─────────────────────────┐
│ Tier 1: Upstream Test   │ ──> │ Tier 2: Modified Test   │ ──> │ Tier 3: Podman Play     │ ──> │ Tier 4: Talos Cluster   │
│ Surface root & cap gaps │     │ Verify non-root & fixes │     │ Validate K8s manifests  │     │ Live Helm verification  │
│ (compose.upstream.yml)  │     │ (compose.yml)           │     │ (podman play kube)      │     │ (helm install)          │
└─────────────────────────┘     └─────────────────────────┘     └─────────────────────────┘     └─────────────────────────┘
```

### Tier 1: Upstream Baseline Comparison (`compose.upstream.yml`)

The [`compose.upstream.yml`](compose.upstream.yml) configuration runs the original, unmodified upstream container image (`mariadb:12-ubi`):

```sh
# Start upstream container
podman compose -f compose.upstream.yml up -d

# Stop upstream container
podman compose -f compose.upstream.yml down
```

**Why test upstream?**
Running the unmodified image against your restricted Podman environment simulates deploying standard public images directly into OpenShift or Talos Linux. This will typically surface common failures:
- Processes attempting to run as `root` (UID 0) or user `mysql` (UID 999) without arbitrary UID flexibility.
- Inability to write to system, run, or database directories without proper group 0 (`root` group) permissions.
- Disallowed operations and capability rejections under restricted SCC / PSA profiles.

---

### Tier 2: Modified Image Local Validation (`compose.yml`)

The [`compose.yml`](compose.yml) configuration builds and runs the customized `Dockerfile` containing the adaptations required for OpenShift and Talos Linux:

```sh
# Build and start the modified compliant container
mise run compose
# or: podman compose up -d --build

# View container logs
mise run logs
# or: podman compose logs -f

# Stop modified stack
mise run down
# or: podman compose down
```

**What this verifies:**
- Rootless image build and layer assembly without host root privileges.
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

---

### Tier 3: Local Kubernetes Manifest Testing (`mise run play`)

Before deploying to an actual Kubernetes cluster, you can test the rendered Kubernetes manifests locally using Podman's built-in `play kube` feature.

```sh
# Render templates and play Kubernetes manifests locally
mise run play

# Teardown the played pod and resources
mise run play-d
```

**How `mise run play` works:**
1. Triggers the dependent task `mise run helm-t`, which executes:
   ```sh
   helm dependency build chart/
   helm template test chart/ > rendered.yaml
   ```
2. Executes `podman play kube rendered.yaml`, which:
   - Reads the multi-document Kubernetes YAML (`ConfigMap`, `PersistentVolumeClaim`, `Service`, `Deployment`).
   - Creates a local Podman pod matching the Kubernetes `Deployment` specification.
   - Applies the pod's `securityContext` (`runAsNonRoot: true`, capabilities drop, seccomp profile).
   - Mounts the persistent volume and ConfigMap into the container at `/var/lib/mysql` and `/docker-entrypoint-initdb.d/`.
   - Exposes container port `3306`.

**Inspecting the local play deployment:**
```sh
# View running pods created by play kube
podman pod ps

# View container status within the pod
podman ps --filter "pod=mariadb"

# Check container logs within the pod
podman logs -f mariadb-pod-mariadb
```

**Teardown:**
```sh
mise run play-d
# or: podman play kube rendered.yaml --down
```

---

### Tier 4: Cluster Deployment & Testing on Talos Linux (`mise run helm-i`)

The final phase validates the workload on a live **Talos Linux** Kubernetes cluster. This tests real-world Pod Security Admission (PSA) enforcement, CSI storage provisioning, network policies, and database startup.

#### 1. Cluster Prerequisites & Configuration

Ensure your `kubectl` context points to your Talos cluster:
```sh
kubectl config current-context
# Example: admin@my-talos-cluster
```

Ensure the container image is accessible to your Talos nodes (e.g., built and pushed to GitHub Container Registry `ghcr.io` or your local registry):
```sh
# Build image locally with target tag
mise run build
```

Configure `chart/values.yaml` for Talos Linux:
- **StorageClass**: If your Talos cluster uses a specific CSI storage provisioner (e.g., `local-path`, `mayastor`, `ceph-block`), configure `mariadb.storageClass` in `values.yaml` or leave it empty `""` to use the cluster's default StorageClass.
- **Security Context & fsGroup**: Under `mariadb.podSecurityContext`, `fsGroup: 1031` ensures mounted storage has permissions accessible by the container user in vanilla Kubernetes / Talos Linux. If deploying to OpenShift, remove or comment out `fsGroup` as OpenShift's SCC allocates fsGroup dynamically.

#### 2. Linting & Template Validation

```sh
# Lint the chart for syntax and formatting errors
mise run helm-l

# Inspect the rendered manifests before installation
mise run helm-t
cat rendered.yaml
```

#### 3. Deploying to the Talos Cluster

Install the Helm chart release:
```sh
mise run helm-i
# or: helm install test chart/
```

#### 4. Verifying Talos PSS Compliance & Health

Check the pod status and verify that Talos Linux Pod Security Admission (PSA) allowed the pod to run:

```sh
# Check pod deployment status
kubectl get pods -l app=mariadb

# Inspect pod details and events for security policy rejections
kubectl describe pod -l app=mariadb
```

> [!TIP]
> If your namespace enforces the `restricted` Pod Security Standard and there are non-compliant settings (such as missing `runAsNonRoot` or un-dropped capabilities), `kubectl describe pod` will show warning events from the `pod-security` admission controller.

Check the application logs:
```sh
kubectl logs -l app=mariadb -f
```

Verify persistent storage and schema initialization inside the pod:
```sh
kubectl exec -it deployment/mariadb -- ls -la /var/lib/mysql
kubectl exec -it deployment/mariadb -- mariadb -u mariadb -p -e "SHOW DATABASES;"
```

Verify network access via port-forwarding:
```sh
kubectl port-forward svc/mariadb 3306:3306
```

#### 5. Uninstalling from the Talos Cluster

When testing is complete, clean up the release:
```sh
mise run helm-u
# or: helm uninstall test
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
| `mariadb.podSecurityContext` | Pod security context (runAsNonRoot, seccompProfile, fsGroup) | `runAsNonRoot: true`, `RuntimeDefault`, `fsGroup: 1031` |
| `mariadb.securityContext` | Container security context (capabilities, allowPrivilegeEscalation) | `allowPrivilegeEscalation: false`, `drop: [ALL]` |
| `initSchema.enabled` | Mount and execute `schema.sql` on first boot | `true` |

---

## Database Initialization (`schema.sql`)

MariaDB executes scripts found in `/docker-entrypoint-initdb.d/` on first startup when the database directory is empty.

- **Local Development:** [`compose.yml`](compose.yml) binds [`schema.sql`](schema.sql) directly into `/docker-entrypoint-initdb.d/schema.sql:z`.
- **Helm Chart:** When `initSchema.enabled: true`, the chart creates a ConfigMap from [`chart/config/schema.sql`](chart/config/schema.sql) and mounts it into the container.
- If you do not require initialization schemas, disable it via `--set initSchema.enabled=false` or remove the volume mount from `compose.yml`.

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
| `play-d` | Stop and remove Podman Play Kube pods | `podman play kube rendered.yaml --down` |
| `helm-d` | Build Helm chart dependencies | `helm dependency build chart/` |
| `helm-l` | Lint Helm chart | `helm lint chart/` |
| `helm-t` | Render Helm chart templates to `rendered.yaml` | `helm template test chart/ > rendered.yaml` |
| `helm-i` | Install Helm chart to current Kubernetes cluster | `helm install test chart/` |
| `helm-u` | Uninstall Helm chart release from cluster | `helm uninstall test` |
| `build` | Build container image locally with Podman Buildx | `podman buildx build --platform linux/amd64 -t ghcr.io/joeckr/mariadb:test . --load` |
| `trivy-fs` | Scan repository filesystem for security vulnerabilities | `trivy fs .` |
| `trivy-i` | Scan built container image with Trivy | `trivy image ghcr.io/joeckr/mariadb:test` |

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
