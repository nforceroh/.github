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
| `setup-test-command` | *(empty)* | command run on the runner before the smoke test, for suites needing host-side tooling (e.g. `pip install -r requirements.txt`); empty string skips |
| `test-command` | `./tests/run_tests.sh "$IMAGE"` | post-build smoke test; `$IMAGE` holds the locally built reference; empty string skips |
| `extra-build-args` | *(empty)* | newline separated extra build args, one `KEY=value` per line |
| `extra-tags` | *(empty)* | comma separated literal tags to publish in addition to the defaults |
| `publish-base-digest` | `false` | also tag with `<base-tag>-<base-digest>`, resolved from the Dockerfile's first `FROM` |
| `platforms` | `linux/amd64` | target platforms |
| `registry` | `ghcr.io` | registry host |
| `registry-owner` | calling repo owner | registry namespace |
| `provenance` | `true` | emit a build provenance attestation |

Published tags: `sha-<short>`, `<build date>`, `latest` on `main` and `v*`
tags, plus `branch` and any semver tags when the ref is a version tag.

`BUILD_DATE`, `VERSION` (both the build date), `VCS_REF` (the commit SHA) and
`BASE_DIGEST` (resolved from the Dockerfile's first `FROM`) are always passed
as build args. A Dockerfile that declares these `ARG`s with an `unknown`
default would otherwise publish an image whose labels all read `unknown`.

The registry login runs before the digest lookup and before the test build,
because both may need to pull from the registry. Digest lookup stays anonymous
on a `pull_request` event, where logging in is not appropriate — so a
Dockerfile whose base image is private will get `BASE_DIGEST=unknown` on PR
builds (a warning, not a failure) and a real digest on every other event.

Unpinned `latest` and `branch` tags mean two builds can share a tag; pin by
digest downstream if that matters.

The `setup-test-command` and `test-command` inputs are the only places caller
input is interpolated into a shell, which is intentional — they are code the
caller is declaring, same as `runs-on` or `script`. Every other input reaches
the shell through `env`.

### `image-cleanup.yaml`

Prunes old GHCR versions.

| input | default | purpose |
| --- | --- | --- |
| `image-name` | calling repo name | GHCR package to prune |
| `min-versions-to-keep` | `5` | most recent untagged versions retained |
| `delete-only-untagged` | `true` | never delete a tagged version |
| `dry-run` | `false` | list what would be deleted without deleting |

Listing a package's versions needs a different endpoint for org-owned and
user-owned packages, so the workflow probes both. Deletions are paginated via
`gh api`, and a failed delete is reported as a warning and counted rather than
silently ignored.

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

`default.json` in this repository is a shared preset — Renovate only looks for
`default.json` when a preset name is omitted, and `renovate.json` is deprecated
as a preset filename. Image repos consume it with a three line `renovate.json`
of their own:

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
