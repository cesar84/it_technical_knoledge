# Local CI Testing and GitHub Actions Updates (2026)

## Executive Summary
This document provides technical procedures for testing GitHub Actions workflows locally (focusing on Fedora Workstation with Docker/Podman), managing multi-environment configurations, executing unpushed/modified workflows, and an overview of recent GitHub Actions and `nektos/act` releases and security updates as of September 2026.

---

## 1. Local GitHub Actions Execution: `nektos/act`

`nektos/act` uses container runtimes (Docker or Podman) to parse `.github/workflows` and run jobs locally in an environment replicating GitHub-hosted runners.

### Installation on Fedora
On Fedora Workstation, install the binary directly or via system packages:

```bash
# Direct binary installation
curl --proto '=https' --tlsv1.2 -sSf https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash -s -- -b /usr/local/bin

# Verify installation
act --version
```

### Container Runtime Setup
`act` communicates with the container daemon via the Docker socket (`/var/run/docker.sock`).

* **Docker Engine**:
  ```bash
  sudo dnf install -y docker-ce docker-ce-cli containerd.io
  sudo systemctl enable --now docker
  sudo usermod -aG docker $USER
  newgrp docker
  ```

* **Podman (Alternative on Fedora)**:
  Enable the rootless Podman systemd socket to expose the Docker API:
  ```bash
  systemctl --user enable --now podman.socket
  export DOCKER_HOST="unix://$XDG_RUNTIME_DIR/podman/podman.sock"
  ```

---

## 2. Testing Unpushed & Modified Workflows

By default, `act` reads workflows and source files directly from the working tree in your current filesystem directory, rather than checking out a git ref.

### Direct Workflow Invocation
* **Execute a specific modified workflow file:**
  ```bash
  act -W .github/workflows/deploy.yml
  ```
* **Target a specific job inside the unpushed file:**
  ```bash
  act -W .github/workflows/deploy.yml -j build-and-test
  ```
* **Simulate `workflow_dispatch` without pushing:**
  ```bash
  act workflow_dispatch -W .github/workflows/deploy.yml
  ```

> **Warning on Checkout:** Avoid using `--no-skip-checkout`. That flag forces git to clone from the remote origin URL, discarding local uncommitted or unpushed modifications.

---

## 3. Multi-Environment Workflow Management

GitHub Actions environments (`jobs.<id>.environment`) and protected deployment secrets cannot be queried natively by `act` without a remote GitHub connection. Segregate environments locally using distinct property files and payloads.

### File Structure
```text
.
├── .github/workflows/deploy.yml
├── .env.staging
├── .secrets.staging
├── staging-event.json
├── .env.production
├── .secrets.production
└── production-event.json
```
*Always ensure `*.secrets*` and `.env*` are listed in `.gitignore`.*

### Staging Environment Execution
```bash
act workflow_dispatch \
  -W .github/workflows/deploy.yml \
  --env-file .env.staging \
  --secret-file .secrets.staging \
  --var ENVIRONMENT=staging \
  -e staging-event.json
```

### Production Environment Execution
```bash
act workflow_dispatch \
  -W .github/workflows/deploy.yml \
  --env-file .env.production \
  --secret-file .secrets.production \
  --var ENVIRONMENT=production \
  -e production-event.json
```

### Mocking GitHub Context & Repository State
To simulate repository secrets and GitHub event contexts:
```bash
act push \
  -W .github/workflows/deploy.yml \
  --secret-file .secrets.staging \
  --env-file .env.staging \
  -e <(echo '{"repository": {"full_name": "my-org/my-app"}, "ref": "refs/heads/main"}')
```

---

## 4. Latest Developments & Ecosystem Releases (2026)

### `nektos/act` Release Timeline (2025–2026)
* **v0.2.89 (May 31, 2026 / June 2026)**: Dependency upgrades (`go-git/go-billy`), multi-platform compilation updates, performance and build fixes.
* **v0.2.88 (April 30, 2026)**: OpenTelemetry SDK bump (`go.opentelemetry.io/otel/sdk` 1.43.0).
* **v0.2.86 (March 25, 2026)**: Addressed Node.js 20 deprecation warnings and runner step adjustments.
* **v0.2.84 – v0.2.85 (December 2025 – March 2026)**: Fixed YAML anchor handling issues (`explode yaml anchors`) and applied security dependency updates.

### GitHub Actions Upstream Platform Updates (Q3 2026)
* **September 3, 2026**:
  * **Runner Version Deprecations API**: Introduced `GET /actions/runners/deprecations/{version}` endpoint to allow platform engineers to inspect runtime and registration deprecation dates ahead of time.
  * **`vulnerability-alerts` Token Permission**: Added granular `read`/`none` permission scoping for `GITHUB_TOKEN` to access Dependabot alerts without granting full administrative privileges.
  * **Job Context for Reusable Workflows**: Added `job.workflow_ref`, `job.workflow_sha`, `job.workflow_repository`, and `job.workflow_file_path` runtime properties to let reusable workflows introspect their own identity.
* **Self-Hosted Runner Version Enforcement (Mid 2026)**:
  * GitHub resumed mandatory runner version enforcement (minimum version 2.329.0+) with phased brownouts culminating across GitHub Enterprise Cloud in September 2026.
* **Runner Scale Set Client (2026)**:
  * Introduction of the Go-based standalone Runner Scale Set Client enabling autoscaling of custom self-hosted runner infrastructure without mandatory Kubernetes dependencies.

---

## 5. Official References & Documentation
* [GitHub Changelog: Early September 2026 updates](https://github.blog/changelog/2026-09-03-github-actions-early-september-2026-updates/)
* [GitHub Changelog: Minimum version enforcement timeline for self-hosted runners](https://github.blog/changelog/2026-06-12-github-actions-minimum-version-enforcement-timeline-for-self-hosted-runners/)
* [nektos/act Releases on GitHub](https://github.com/nektos/act/releases)
* [nektos/act Documentation](https://nektosact.com/usage/)
