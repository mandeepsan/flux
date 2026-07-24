# Kairos Build and Forgejo Actions Context

## Objective

Build a reproducible Forgejo Actions pipeline for Kairos-based OCI images.

The pipeline must:

- Build and publish `linux/amd64` and `linux/arm64` OCI images.
- Publish OCI images to the existing Harbor registry.
- Optionally upload ISO, raw disk, QCOW2, checksums, and release bundles to SeaweedFS S3.
- Use existing organization-scoped Forgejo Actions secrets and variables.
- Run on the Kubernetes-hosted Forgejo runner.
- Keep credentials out of source code, generated artifacts, and logs.

Do not modify the Flux infrastructure repository unless explicitly requested. Work only in the application or Kairos repository. Inspect the target repository before choosing a Kairos build mechanism.

## Platform

### Forgejo

Forgejo is available at:

```text
https://git.otroid.net
```

Workflows belong in:

```text
.forgejo/workflows/
```

Prefer fully qualified Forgejo action references:

```yaml
uses: https://data.forgejo.org/actions/checkout@v4
```

Do not assume complete GitHub Actions compatibility. Confirm that each action and version is available from the selected action source.

### Forgejo Automation User

Automation user:

```text
otroid-bot
```

The user belongs to the organization team `ci-bots`, which has access to all organization repositories with these permissions:

| Unit | Permission |
| --- | --- |
| Code | Read |
| Pull requests | Read |
| Packages | Write |
| Actions | Write |

A Forgejo application token is not required for checkout, building, Harbor login, or Harbor publication. Do not request one unless the workflow must call the Forgejo API, create releases, push commits or tags, open pull requests, or publish to Forgejo Packages.

### Actions Runner

Global runner identity:

```text
netcup-k8s-dind-01
```

Select the runner with:

```yaml
runs-on: kairos
```

Available labels:

```text
self-hosted
linux-amd64
docker
buildx
qemu
kairos
```

Build capacity:

| Resource | Capacity |
| --- | --- |
| CPU limit | 6 cores |
| Memory limit | 12 GiB |
| Docker cache PVC | 100 GiB |
| Native architecture | AMD64 |
| ARM64 execution | QEMU emulation |

The runner uses Docker-in-Docker and supports privileged container builds. It has one replica, so expensive multi-platform builds should be serialized. ARM64 builds are emulated and will generally be slower than AMD64 builds.

Do not mount a host Docker socket or provide Kubernetes API credentials to application workflows.

## Harbor Registry

Registry authority:

```text
harbor.otroid.net
```

Harbor project:

```text
kairos
```

Use OCI references without an URL scheme:

```text
harbor.otroid.net/kairos/<image>:<tag>
```

Do not use `https://` in an OCI image reference. Docker and OCI clients use HTTPS automatically.

The `kairos` project is private. Harbor performs Trivy vulnerability scanning after images are pushed. CI authenticates using a Harbor robot account; Harbor records that robot identity as the image publisher, independently of the Forgejo workflow actor.

### Harbor Secrets

These organization-scoped Forgejo Actions secrets already exist:

```text
HARBOR_USERNAME
HARBOR_PASSWORD
```

Reference them only through the secrets context:

```yaml
${{ secrets.HARBOR_USERNAME }}
${{ secrets.HARBOR_PASSWORD }}
```

### Harbor Variables

These organization-scoped variables already exist:

```text
HARBOR_REGISTRY=harbor.otroid.net
HARBOR_PROJECT=kairos
```

Reference them through the variables context:

```yaml
${{ vars.HARBOR_REGISTRY }}
${{ vars.HARBOR_PROJECT }}
```

### Harbor Authentication

Authenticate before invoking Buildx:

```yaml
- name: Log in to Harbor
  uses: https://data.forgejo.org/docker/login-action@v3
  with:
    registry: ${{ vars.HARBOR_REGISTRY }}
    username: ${{ secrets.HARBOR_USERNAME }}
    password: ${{ secrets.HARBOR_PASSWORD }}
```

If an action is unavailable at `data.forgejo.org`, verify a compatible and trusted source before changing it. Do not silently introduce an unpinned third-party action.

## S3 Artifact Storage

SeaweedFS provides an S3-compatible endpoint:

```text
https://s3.otroid.net
```

Dedicated bucket:

```text
kairos-artifacts
```

S3 is not used for OCI images. Harbor owns OCI manifests and layers. Use S3 for non-OCI release outputs such as:

```text
*.iso
*.raw
*.img
*.qcow2
*.sha256
*.sig
build-manifest.json
release metadata
```

Workflow logs are collected by Forgejo and must not be uploaded to S3.

### S3 Secrets

These organization-scoped secrets already exist:

```text
S3_ACCESS_KEY_ID
S3_SECRET_ACCESS_KEY
```

### S3 Variables

These organization-scoped variables already exist:

```text
S3_ENDPOINT=https://s3.otroid.net
S3_BUCKET=kairos-artifacts
S3_REGION=auto
```

Use a deterministic object layout:

```text
s3://kairos-artifacts/<repository>/<version>/<artifact>
```

For example:

```text
s3://kairos-artifacts/kairos-custom/v1.2.0/kairos-amd64.iso
s3://kairos-artifacts/kairos-custom/v1.2.0/kairos-arm64.iso
s3://kairos-artifacts/kairos-custom/v1.2.0/SHA256SUMS
```

Upload with the endpoint explicitly supplied:

```bash
aws --endpoint-url "$S3_ENDPOINT" \
  s3 cp ./dist/ "s3://$S3_BUCKET/$REPOSITORY/$VERSION/" \
  --recursive
```

Pass credentials through step environment variables. Do not persist them with `aws configure`.

## Suggested Repository Structure

```text
.
|-- .forgejo/
|   `-- workflows/
|       |-- validate.yaml
|       `-- release.yaml
|-- config/
|   `-- kairos/
|-- scripts/
|   |-- build.sh
|   |-- verify.sh
|   `-- publish-artifacts.sh
|-- Containerfile
|-- Makefile
|-- README.md
`-- .gitignore
```

Keep substantial build logic in version-controlled scripts or a Makefile. Workflows should orchestrate those scripts instead of embedding large shell programs. Shell scripts should begin with:

```bash
set -euo pipefail
```

## Workflow Design

### Validation Workflow

Run validation for pull requests and non-release branch pushes. It should:

- Check out the exact revision.
- Validate Kairos configuration, scripts, and the Containerfile.
- Build enough of the image to detect failures.
- Avoid publishing images or persistent S3 artifacts.
- Avoid exposing production credentials to untrusted pull-request code.
- Cancel superseded runs where Forgejo concurrency support permits it.

### Release Workflow

Run releases for:

```text
pushes to main
version tags matching v*
manual workflow dispatch
```

The release workflow should:

1. Check out the exact commit.
2. Derive immutable image and artifact versions.
3. Configure QEMU for ARM64 emulation.
4. Configure Docker Buildx.
5. Authenticate to Harbor.
6. Build `linux/amd64` and `linux/arm64` images.
7. Push both images and a combined multi-platform manifest.
8. Generate checksums and build metadata.
9. Upload non-OCI release files to S3 when they exist.
10. Report image references and artifact locations without exposing credentials.

Recommended image tags:

```text
harbor.otroid.net/kairos/<repository>:<git-sha>
harbor.otroid.net/kairos/<repository>:v1.2.0
harbor.otroid.net/kairos/<repository>:latest
```

The commit SHA or release version is authoritative. `latest` is only a convenience tag and must not be the sole published reference.

## Multi-Architecture Build

Configure QEMU and Buildx:

```yaml
- name: Configure QEMU
  uses: https://data.forgejo.org/docker/setup-qemu-action@v3
  with:
    platforms: amd64,arm64

- name: Configure Buildx
  uses: https://data.forgejo.org/docker/setup-buildx-action@v3
```

Build these platforms:

```yaml
platforms: linux/amd64,linux/arm64
```

Prefer a Harbor-backed BuildKit registry cache where practical:

```text
harbor.otroid.net/kairos/<repository>-buildcache
```

Do not publish cache images as releases or apply release tags to them.

Kairos tooling may require privileged Docker operations, loop devices, filesystem creation, or nested container execution. Inspect the selected Kairos build method before assuming that a conventional unprivileged Containerfile build is sufficient.

## OCI Metadata

Add standard OCI labels when values are available:

```text
org.opencontainers.image.title
org.opencontainers.image.description
org.opencontainers.image.version
org.opencontainers.image.revision
org.opencontainers.image.source
org.opencontainers.image.created
```

The revision must match the Forgejo commit SHA.

Generate an SBOM where supported. Prefer attaching SBOMs, signatures, and provenance to the Harbor image as OCI artifacts instead of storing them in S3. Do not introduce signing keys until a key-management design has been approved.

## Security Requirements

- Never commit secrets or generated credential files.
- Never print credential-bearing environment variables.
- Never enable `set -x` in steps that handle credentials.
- Prefer environment variables or stdin over command-line secret arguments.
- Pin actions to reviewed versions; commit SHA pinning is preferable for publishing workflows.
- Never publish from untrusted pull-request code.
- Protect `.forgejo/workflows/`, the Containerfile, Kairos configuration, and build scripts.
- Do not provide Kubernetes credentials to build workflows.
- Do not add a Forgejo bot token without a concrete Forgejo API requirement.
- Treat every organization repository with access to organization secrets and the privileged global runner as trusted.

## Expected Deliverables

Before implementation, inspect the target repository and determine the correct Kairos build mechanism and output formats. Then provide:

1. A maintainable repository structure.
2. Kairos configuration.
3. Reproducible local build commands.
4. A pull-request validation workflow.
5. A multi-platform release workflow.
6. Harbor publication.
7. Optional S3 artifact publication.
8. `.gitignore` entries for generated images, credentials, and build state.
9. README instructions for local builds and CI releases.
10. Verification commands.

Do not commit or push until explicitly requested.

## Acceptance Criteria

The implementation is complete when:

- Pull requests validate without publishing.
- A release build runs on the `kairos` runner.
- Both AMD64 and ARM64 images are produced.
- Harbor contains a combined multi-platform manifest.
- Anonymous access cannot pull the private Kairos image.
- Authenticated pulls succeed.
- Harbor starts a Trivy scan after publication.
- Optional ISO or disk artifacts appear under the expected S3 prefix.
- No credentials appear in Git history, workflow logs, or artifacts.
- Re-running a build uses deterministic names and does not corrupt prior releases.

## First Decision for the Implementation Chat

Establish which Kairos build path the repository will use before writing workflows. Examples include Kairos Factory, AuroraBoot, Earthly, or a custom Kairos Containerfile. This choice determines the privileged build requirements, artifact formats, and ARM64 strategy.
