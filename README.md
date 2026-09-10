# rmes-githubactions-commons

Shared GitHub Actions workflows for the RMéS projects, so that the same release
plumbing is not copied into every Java and JavaScript repository.

| Workflow | What it does |
| --- | --- |
| [`prepare-release.yml`](.github/workflows/prepare-release.yml) | Bumps the version, commits, tags and creates the GitHub release, in one click from the calling repository's Actions tab. |
| [`docker-publish.yml`](.github/workflows/docker-publish.yml) | Builds the image of the release that fired the event and pushes it under the release tag, on ghcr.io or Docker Hub. |

## prepare-release

The caller keeps only what is specific to it: the `workflow_dispatch` form (it
is the only place a run can be started from) and, when there is one, where its
version lives.

Maven or Node is **deduced from the manifest** found at the root — `pom.xml` or
`package.json`. A repository holding both must say which one with the
`ecosystem` input; the run is refused rather than guessed.

### Maven

```yaml
name: Prepare release

on:
  workflow_dispatch:
    inputs:
      kind:
        description: "What to publish"
        type: choice
        options: [prerelease, release]
        default: prerelease
      version:
        description: "Explicit version. Empty = next rc (prerelease), or -rc suffix dropped (release)"
        required: false
      dry_run:
        description: "Compute the version and stop there, pushing nothing"
        type: boolean
        default: false

jobs:
  release:
    uses: InseeFr/rmes-githubactions-commons/.github/workflows/prepare-release.yml@main
    permissions:
      contents: write
    secrets: inherit
    with:
      kind: ${{ inputs.kind }}
      version: ${{ inputs.version }}
      dry_run: ${{ inputs.dry_run }}
      module: module-bauhaus-bo   # omit when the version lives in the root pom
      java-version: '25'
```

### Node

Same `on:` block, then — nothing but the three inputs of the form:

```yaml
jobs:
  release:
    uses: InseeFr/rmes-githubactions-commons/.github/workflows/prepare-release.yml@main
    permissions:
      contents: write
    secrets: inherit
    with:
      kind: ${{ inputs.kind }}
      version: ${{ inputs.version }}
      dry_run: ${{ inputs.dry_run }}
```

### Inputs

| Input | Required | Default | |
| --- | --- | --- | --- |
| `kind` | yes | | `prerelease` or `release`. |
| `version` | no | *empty* | Explicit version. Empty computes the next one: `x.y.z-rcN` → `-rcN+1` for a prerelease, `x.y.z-rcN` → `x.y.z` for a release. |
| `dry_run` | no | `false` | Compute and report the version in the job summary, push nothing. |
| `ecosystem` | no | *detected* | `maven` (version read with `help:evaluate`, written with `versions:set`) or `node` (`jq` / `npm pkg set`). Only needed when both manifests are present. |
| `module` | no | *empty* | Maven module carrying the application version (`-pl`). Empty targets the root pom. |
| `working-directory` | no | `.` | Directory holding the manifest, for a repository whose app is not at the root. |
| `release-branch` | no | `main` | Branch carrying final releases. A `release` run is refused anywhere else, a `prerelease` run is refused on it. |
| `java-version` | no | `25` | Maven only. |
| `java-distribution` | no | `corretto` | Maven only. |
| `versions-plugin` | no | `org.codehaus.mojo:versions-maven-plugin:2.19.1` | Maven only. |

Output: `version`, the version that was tagged.

### What the caller must provide

- **`permissions: contents: write`** on the calling job. A reusable workflow can
  only lower the permissions it is given, never raise them, so declaring it in
  the called workflow alone is not enough.
- **`secrets: inherit`**, so that `RELEASE_TOKEN` reaches the release step.
- **A `RELEASE_TOKEN` secret** (a PAT with `contents: write`), optional but
  strongly recommended: a release created with the default `GITHUB_TOKEN` never
  triggers the workflows that listen to the `release` event, so the Docker image
  is not built. Without it the run still succeeds and prints a warning.
- **The `mvnw` wrapper**, for a Maven project: the workflow calls `./mvnw`.

If this repository is ever made private, its workflows must additionally be
allowed for the organisation, in *Settings → Actions → General → Access*.

## docker-publish

Triggered by the calling repository's own `release` event: `prereleased` for the
ghcr.io image, `released` for the Docker Hub one. The image is built from the
source of the tag — a `release` event checks out `refs/tags/<tag>` — and no
artifact is expected from a previous job, since the Dockerfiles build the
application themselves.

**A final release must be tagged `x.y.z`** (a leading `v` is tolerated), else the
run is refused. That rule is built in rather than configured: an `-rcN` image
reaching the final registry is not a project preference, so no caller can opt out
of it. Prereleases are left alone, which is where the `-rcN` tags belong.

```yaml
name: Build Beta

on:
  release:
    types: [prereleased]

jobs:
  docker:
    uses: InseeFr/rmes-githubactions-commons/.github/workflows/docker-publish.yml@main
    permissions:
      contents: read
      packages: write      # ghcr.io only
    secrets: inherit
    with:
      image: inseefr/bauhaus-back-office
      registry: ghcr.io
      dockerfile: Dockerfile.bauhaus
```

A repository that wants a quality gate before publishing keeps that job for
itself and makes the shared one `needs:` it:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.release.tag_name }}
      - uses: ./.github/actions/generic-ci-app
  docker:
    needs: build
    uses: InseeFr/rmes-githubactions-commons/.github/workflows/docker-publish.yml@main
    ...
```

### Inputs

| Input | Required | Default | |
| --- | --- | --- | --- |
| `image` | yes | | Image name, without registry or tag: `inseefr/bauhaus`. |
| `registry` | no | *empty* | `ghcr.io`, or empty for Docker Hub. Drives which credentials are used. |
| `dockerfile` | no | `Dockerfile` | |
| `context` | no | `.` | |
| `tag` | no | *release tag* | Overrides the tag the image is pushed under. |
| `latest` | no | `false` | Also push `:latest`. |
| `platforms` | no | *empty* | `linux/amd64,linux/arm64`… QEMU is only set up when this is set. |

Output: `image`, the fully qualified reference that was pushed.

### What the caller must provide

- **`permissions: packages: write`** when pushing to ghcr.io, on top of
  `contents: read`. A reusable workflow can only lower what it is given.
- **`secrets: inherit`**, so that `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`
  reach the login step when the target is Docker Hub.

## Conventions

Callers pin `@main`: every repository picks up a change here on its next run.
The counterpart is that **a change to this repository must stay backwards
compatible** — add optional inputs, never rename or remove one, and never change
a default in a way that alters what an existing caller does. A genuinely
breaking change means a `v2` branch and a migration of the callers, one by one.

Adding an ecosystem (gradle, poetry, cargo…) means adding one branch to the two
`case` blocks of `prepare-release.yml` — how to read the current version, how to
write the next one — and one line to the table above. The rest of the workflow
(branch rules, version arithmetic, tag, release, warnings) is shared as is.
