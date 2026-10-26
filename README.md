# ci-workflows

Reusable GitHub Actions workflows.

## `docker.yml` — build and push a Docker image to GHCR

Builds the image with Buildx, pushes it to `ghcr.io/<owner>/<image>`, then
prunes old builds with
[`dataaxiom/ghcr-cleanup-action`](https://github.com/dataaxiom/ghcr-cleanup-action).
Provenance and SBOM attestations are off, so one build is exactly one GHCR
package version.

### Usage

```yaml
name: Docker
on:
  push: { branches: [main], tags: ['v*'] }
  workflow_dispatch:
jobs:
  image:
    if: "!contains(github.event.head_commit.message, '[skipci]')"
    uses: harold4/ci-workflows/.github/workflows/docker.yml@v1
    permissions: { contents: read, packages: write }
```

The caller job must grant `contents: read` and `packages: write`.
Commit-message filters (`[skipci]`), branch/path filters and test jobs stay in
the caller.

### Inputs

| Input | Default | Description |
|---|---|---|
| `image` | repository name | image name, pushed as `ghcr.io/<owner>/<image>` (lower-cased) |
| `context` | `.` | build context |
| `dockerfile` | `<context>/Dockerfile` | Dockerfile path |
| `version-from` | – | `package.json` (its `version`) or a plain `VERSION` file → `v{ver}` tag and `BUILD_VERSION` build-arg |
| `build-args` | – | extra build-args, one `KEY=value` per line |
| `cache-scope` | image name | GitHub Actions cache scope; set it when one repo builds several images |
| `keep-builds` | `5` | tagged builds kept by the cleanup; `0` disables the cleanup |

### Secrets

| Secret | Description |
|---|---|
| `secret-build-args` | secret build-args, one `KEY=value` per line (the `secrets` context is not available in `with:`, hence a separate secret) |
| `build-secrets` | BuildKit secrets, `id=value` per line, for `RUN --mount=type=secret,id=...` |

### Outputs

| Output | Description |
|---|---|
| `image` | `ghcr.io/<owner>/<image>` (without tag) |
| `hash` | 7-character commit hash |

### Tags

| Tag | When |
|---|---|
| `latest` | push to the default branch |
| `<sha7>` | always |
| `v{ver}` | `version-from` is set |
| `v{semver}` | a `v*` git tag is pushed |

### Build-args always passed

| Arg | Value |
|---|---|
| `BUILD_HASH` | 7-character commit hash |
| `BUILD_DATE` | UTC build time, `YYYY-MM-DDTHH:MM:SSZ` |
| `BUILD_VERSION` | only with `version-from` |

The `.git` directory is normally excluded from the build context, so the
Dockerfile should declare these as `ARG` and bake them into the app.

### Examples

Versioned frontend in a subdirectory, with a secret build-arg:

```yaml
jobs:
  image:
    uses: harold4/ci-workflows/.github/workflows/docker.yml@v1
    permissions: { contents: read, packages: write }
    with:
      image: my-app
      context: web
      version-from: web/package.json
    secrets:
      secret-build-args: |
        VITE_API_KEY=${{ secrets.MY_APP_API_KEY }}
```

Tests first, private Go module fetched through a BuildKit secret:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: go test ./...
  image:
    needs: test
    uses: harold4/ci-workflows/.github/workflows/docker.yml@v1
    permissions: { contents: read, packages: write }
    secrets:
      build-secrets: |
        netrc=machine github.com login x password ${{ secrets.MY_TOKEN }}
```

Two images from one repository:

```yaml
jobs:
  image:
    strategy:
      matrix:
        part: [backend, frontend]
    uses: harold4/ci-workflows/.github/workflows/docker.yml@v1
    permissions: { contents: read, packages: write }
    with:
      image: my-app-${{ matrix.part }}
      context: ${{ matrix.part }}
```

## Versioning

Callers pin a major tag (`@v1`). Compatible changes move the `v1` tag;
breaking changes get `v2`.
