# nforceroh/.github

Shared, reusable CI for the `nforceroh` homelab container images.

Every image repo keeps its own repository, history and release cadence. This
repository exists only so the build and cleanup pipelines are defined once
instead of being copy-pasted into each one.

## Workflows

### `image-build.yaml`

Builds, smoke tests and publishes an image to GHCR.

| input | default | purpose |
| --- | --- | --- |
| `image-name` | calling repo name | published image name |
| `context` | `.` | build context |
| `dockerfile` | `Dockerfile` | Dockerfile path |
| `test-command` | `./tests/run_tests.sh "$IMAGE"` | post-build smoke test; `$IMAGE` holds the locally built reference; empty string skips |
| `platforms` | `linux/amd64` | target platforms |
| `registry` | `ghcr.io` | registry host |
| `registry-owner` | calling repo owner | registry namespace |
| `extra-tags` | *(empty)* | newline separated additional tags |
| `provenance` | `true` | emit a build provenance attestation |

Published tags: `sha-<short>`, `<build date>`, `latest` on `main` and `v*`
tags, plus `branch` and any semver tags when the ref is a version tag.

Because Git tags and `latest` would otherwise let two builds share a tag, the
caller is responsible for pinning `imagePullPolicy` downstream if it cares
about immutability.

### `image-cleanup.yaml`

Prunes old GHCR versions.

| input | default | purpose |
| --- | --- | --- |
| `image-name` | calling repo name | GHCR package to prune |
| `min-versions-to-keep` | `10` | versions retained |
| `delete-only-untagged` | `true` | skip tagged versions |
| `dry-run` | `false` | report without deleting |

## Usage

```yaml
# .github/workflows/build.yaml
name: build

on:
  push:
    branches: [main]
    tags: ["v*"]
  pull_request:
  schedule:
    - cron: "0 4 * * 0"
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  image:
    uses: nforceroh/.github/.github/workflows/image-build.yaml@v1
    permissions:
      contents: read
      packages: write
      attestations: write
      id-token: write
```

The caller **must** grant `packages: write`. A called workflow cannot elevate
the permissions of the token it receives, so omitting it makes the push fail.

```yaml
# .github/workflows/cleanup.yaml
name: cleanup

on:
  schedule:
    - cron: "17 3 * * 0"
  workflow_dispatch:
    inputs:
      dry_run:
        description: Preview deletions only
        type: boolean
        default: true

jobs:
  prune:
    uses: nforceroh/.github/.github/workflows/image-cleanup.yaml@v1
    permissions:
      contents: read
      packages: write
    with:
      dry-run: ${{ github.event_name == 'workflow_dispatch' && inputs.dry_run || false }}
```

## Renovate preset

`renovate.json` in this repository is a shared preset. Image repos consume it
with a three line `renovate.json` of their own:

```json
{
  "extends": ["github>nforceroh/.github"]
}
```

It tracks the shared `alpine-base` and `nginx-php` images in `FROM` lines,
pins GitHub Actions to digests, and groups those updates. Renovate cannot bump
`apk add` packages inside a Dockerfile, which is why the weekly scheduled
rebuild exists — it picks up patched Alpine packages without a config change.

## Versioning

Reference these workflows as `@v1`. Git tags take precedence over branches of
the same name, so `v1` stays put until it is deliberately re-tagged. Breaking
changes to an input's meaning should be released as `v2`.

## Licence

GPL-3.0-or-later. See [LICENSE](LICENSE).
