# rmes-githubactions-commons

Shared GitHub composite actions for the RMéS projects, so that the same release
and quality plumbing is not copied into every Java and JavaScript repository.

| Action | What it does |
| --- | --- |
| [`prepare-release`](.github/actions/prepare-release/action.yml) | Bumps the version, commits, tags and creates the GitHub release, in one click from the calling repository's Actions tab. |
| [`docker-publish`](.github/actions/docker-publish/action.yml) | Builds the image of the release that fired the event and pushes it under the release tag, on ghcr.io or Docker Hub. |
| [`quality-front`](.github/actions/quality-front/action.yml) | Installs a pnpm front-end, runs its linter, type checker, dead-code and copy-paste checks and unit tests, builds it, then scans it with Sonar when given a token. Starts with `quality`. |
| [`quality-back`](.github/actions/quality-back/action.yml) | Same for a Maven back-end: `quality`, the copy-paste check, then the JDK and the build with its tests. Called by Bauhaus-Back-Office. |
| [`quality`](.github/actions/quality/action.yml) | The checks shared by every project whatever its stack — today the Conventional Commits title of the pull request. Called by `quality-front` and `quality-back`, not directly. |
| [`playwright`](.github/actions/playwright/action.yml) | Installs a pnpm front-end and runs its Playwright suite. The services the tests need are started by the caller. |
| [`setup-front`](.github/actions/setup-front/action.yml) | Installs Node and pnpm — the single place where the versions every front-end runs on are written down. Called by `quality-front` and `playwright`, and directly by a job that needs pnpm without either. |

Everything here is an **action**, not a reusable workflow. An action is inserted
into a job the caller already owns, so that caller keeps its runner, its
permissions, its concurrency group and its job name — and can chain several
bricks in the same job, which is exactly what a publication gate does. The
counterpart is that an action receives no secret on its own: every token is an
input the caller passes explicitly.

## prepare-release

The caller keeps what is specific to it: the `workflow_dispatch` form (it is the
only place a run can be started from), the checkout, and — when there is one —
where its version lives.

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

concurrency:
  group: prepare-release-${{ github.repository }}-${{ github.ref }}
  cancel-in-progress: false

jobs:
  release:
    name: ${{ inputs.kind }} from ${{ github.ref_name }}
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v7
      - uses: InseeFr/rmes-githubactions-commons/.github/actions/prepare-release@main
        with:
          kind: ${{ inputs.kind }}
          version: ${{ inputs.version }}
          dry_run: ${{ inputs.dry_run }}
          module: module-bauhaus-bo   # omit when the version lives in the root pom
          java-version: '25'
          release-token: ${{ secrets.RELEASE_TOKEN }}
```

### Node

Same `on:` and `concurrency:` blocks, then the same job without the two Maven
inputs:

```yaml
      - uses: InseeFr/rmes-githubactions-commons/.github/actions/prepare-release@main
        with:
          kind: ${{ inputs.kind }}
          version: ${{ inputs.version }}
          dry_run: ${{ inputs.dry_run }}
          release-token: ${{ secrets.RELEASE_TOKEN }}
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
| `release-token` | no | *empty* | PAT creating the release. See below. |
| `github-token` | no | `${{ github.token }}` | Used when no `release-token` is given. |

Output: `version`, the version that was tagged.

### What the caller must provide

- **The checkout**, since an action starts inside a job that is already running.
- **`permissions: contents: write`** on the job.
- **`release-token`**, a PAT with `contents: write`, optional but strongly
  recommended: a release created with the default `GITHUB_TOKEN` never triggers
  the workflows that listen to the `release` event, so the Docker image is not
  built. Without it the run still succeeds and prints a warning.
- **A `concurrency` block**, if two runs must not overlap. An action cannot
  declare one — that is a property of the workflow.
- **The `mvnw` wrapper**, for a Maven project: the action calls `./mvnw`.

## docker-publish

Triggered by the calling repository's own `release` event: `prereleased` for the
ghcr.io image, `released` for the Docker Hub one. The image is built from the
tree the caller checked out, and no artifact is expected from a previous job,
since the Dockerfiles build the application themselves.

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
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write      # ghcr.io only
    steps:
      - uses: actions/checkout@v7
      - uses: InseeFr/rmes-githubactions-commons/.github/actions/docker-publish@main
        with:
          image: inseefr/bauhaus-back-office
          registry: ghcr.io
          dockerfile: Dockerfile.bauhaus
```

A repository that wants a quality gate before publishing puts it in the same job,
before the publication — one checkout, one runner:

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v7
        with:
          ref: ${{ github.event.release.tag_name }}
      - uses: InseeFr/rmes-githubactions-commons/.github/actions/quality-front@main
      - uses: InseeFr/rmes-githubactions-commons/.github/actions/docker-publish@main
        with:
          image: inseefr/bauhaus
          registry: ghcr.io
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
| `dockerhub-username` | no | *empty* | Docker Hub account. Required when the registry is Docker Hub. |
| `dockerhub-token` | no | *empty* | Docker Hub token. Required when the registry is Docker Hub. |
| `github-token` | no | `${{ github.token }}` | Used to log in to ghcr.io. |

Output: `image`, the fully qualified reference that was pushed.

### What the caller must provide

- **The checkout.** On a `release` event the default ref is already the tag; a
  job that does something else first should say `ref: ${{ github.event.release.tag_name }}`.
- **`permissions: packages: write`** when pushing to ghcr.io, on top of
  `contents: read`.
- **`dockerhub-username` and `dockerhub-token`** when the target is Docker Hub.

## quality-front

The body of a front-end's quality job: pnpm, Node, install, then the checks the
project declares, the production build, and the Sonar scan when the caller hands
over its token.

```yaml
name: Quality

on:
  pull_request:
    types: [opened, synchronize, reopened]
  push:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: InseeFr/rmes-githubactions-commons/.github/actions/quality-front@main
        with:
          sonar-token: ${{ secrets.SONAR_TOKEN }}
```

A project that names its scripts otherwise, or that has no `knip`, says so —
an empty command drops the step:

```yaml
      - uses: InseeFr/rmes-githubactions-commons/.github/actions/quality-front@main
        with:
          node-version: '22'
          pnpm-version: '10'
          knip-command: ''
          test-command: pnpm test --coverage --watch=false
          react-doctor-command: pnpm react-doctor
          sonar-token: ${{ secrets.SONAR_TOKEN }}
```

The copy-paste check is run with `pnpm dlx`, not a project script: the threshold
is the same for every front-end — **2% of duplicated lines** — and a project
without the dependency is covered too. A project carrying a `.jscpd.json` keeps
it, since jscpd reads it from the working directory; only the threshold is
imposed here, the ignore list and the reporters stay the project's. A codebase
that is above the threshold today raises it for itself — `jscpd-threshold: '5'`
— rather than dropping the check.

The scan runs last, on the coverage the test step has just written, and is
skipped on Dependabot's pull requests, which reach no repository secret. A
publication gate passes no token: it chains the checks before pushing an image
and wants no scan.

### Inputs

| Input | Required | Default | |
| --- | --- | --- | --- |
| `node-version` | no | *empty* | Empty takes the shared version of [`setup-front`](#setup-front). |
| `pnpm-version` | no | *empty* | Empty takes the shared version of [`setup-front`](#setup-front). |
| `working-directory` | no | `.` | Directory holding the application, for a repository whose front is not at the root. |
| `install-command` | no | `pnpm install` | |
| `lint-command` | no | `pnpm lint` | Empty skips the step. |
| `typecheck-command` | no | `pnpm typecheck` | Empty skips the step. |
| `knip-command` | no | `pnpm knip` | Empty skips the step. |
| `jscpd-command` | no | `pnpm dlx jscpd@5 .` | Copy-paste detection, the threshold below appended. Empty skips the step. |
| `jscpd-threshold` | no | `2` | Percentage of duplicated lines tolerated. Above it, the step fails. |
| `test-command` | no | `pnpm test:coverage` | Empty skips the step. |
| `build-command` | no | `pnpm build` | Empty skips the step. |
| `react-doctor-command` | no | *empty* | React diagnostics and dependency audit. Empty skips the step. |
| `sonar-token` | no | *empty* | SonarQube Cloud token, from the caller's secrets. Empty skips the scan. |
| `pr-title-check` | no | `true` | Passed to `quality`. |
| `pr-title-require-scope` | no | `false` | Passed to `quality`. |
| `github-token` | no | `${{ github.token }}` | Passed to `quality`. |

### What the caller must provide

- **The checkout.** A publication gate checks out the release tag; a quality job
  takes the default.
- **`sonar-token`**, when it wants the scan: a composite action receives no
  secret on its own. The scan itself lives here, and reads the repository's
  `sonar-project.properties`.
- **A pnpm lockfile** next to the manifest: it is what the Node cache keys on.

## quality

The checks every project runs whatever its stack, called first by
`quality-front` and `quality-back`: a check added here reaches the front-ends and
the back-ends alike, without touching either.

| Check | Runs when | Replaces |
| --- | --- | --- |
| Pull request title follows [Conventional Commits](https://www.conventionalcommits.org/) | the event is `pull_request` or `pull_request_target` | the `pr-name.yml` workflow each repository carried |

Outside a pull request every check is skipped, so the same job still serves as
the gate of a release or a scheduled run.

The title is checked when the quality job runs, that is on `opened`,
`synchronize` and `reopened`. The old `pr-name.yml` also listened to `edited`:
fixing a bad title now takes a new push or a re-run of the job, unless the
caller adds `edited` to its `pull_request` types — at the cost of a full quality
run on every title or description edit.

### Inputs

| Input | Required | Default | |
| --- | --- | --- | --- |
| `pr-title-check` | no | `true` | `false` skips the title check. |
| `pr-title-require-scope` | no | `false` | Require a scope: `feat(concepts): …`. |
| `github-token` | no | `${{ github.token }}` | Reads the pull request. A job declaring `permissions:` must grant `pull-requests: read`. |

### Composing actions from this repository

An action of this repository calls another one **by its full name**,
`InseeFr/rmes-githubactions-commons/.github/actions/quality@main`. A relative
`uses: ./.github/actions/quality` inside a composite action resolves against the
caller's workspace, where it does not exist. The counterpart: a change to
`quality` is only seen by `quality-front` and `quality-back` once on `main`.

## quality-back

The back-end counterpart of `quality-front`: `quality`, then the JDK with a
Maven cache, then the build and its tests. Bauhaus-Back-Office calls it, and
runs its Sonar analysis through the Maven plugin, inside the build itself. It is
also the first caller to sit above the shared duplication threshold, and raises
it for itself — `jscpd-threshold: '14'` — rather than dropping the check.

```yaml
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - uses: InseeFr/rmes-githubactions-commons/.github/actions/quality-back@main
        with:
          verify-command: ./mvnw -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Pcoverage
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

### Inputs

| Input | Required | Default | |
| --- | --- | --- | --- |
| `java-version` | no | `25` | |
| `java-distribution` | no | `corretto` | |
| `working-directory` | no | `.` | Directory holding the root pom. |
| `jscpd-command` | no | `npx --yes jscpd@5 .` | Copy-paste detection, the threshold below appended. Empty skips the step. |
| `jscpd-threshold` | no | `2` | Percentage of duplicated lines tolerated. Above it, the step fails. |
| `verify-command` | no | `./mvnw -B verify` | Empty skips the step. |
| `pr-title-check` | no | `true` | Passed to `quality`. |
| `pr-title-require-scope` | no | `false` | Passed to `quality`. |
| `github-token` | no | `${{ github.token }}` | Passed to `quality`. |

### What the caller must provide

- **The checkout**, with `fetch-depth: 0` when Sonar runs.
- **The `mvnw` wrapper**, unless `verify-command` says otherwise.
- **The Sonar analysis and its token**, when it wants one.
- **A Node on the runner**, for the copy-paste check: jscpd is a Node tool, run
  with the one the GitHub image already ships rather than a `setup-node` step
  added to every Maven build. A runner without Node passes `jscpd-command: ''`.

The check runs before the JDK and the build: a minute of static analysis rather
than the whole `verify` when duplication is what fails. It respects `.gitignore`,
so `target/` is out of it, and reads a `.jscpd.json` at the root when there is
one. **2% of duplicated lines** is the shared threshold; a codebase above it
raises it for itself rather than dropping the check.

## playwright

The Playwright suite of a front-end: pnpm, Node, install, browsers, the tests,
and the HTML report as an artifact. Playwright starts the dev server itself
(`webServer`).

The action knows nothing of the services a suite talks to. A front-end whose
tests need a back-end — Bauhaus, against a Back-Office and a GraphDB — starts
them in its own workflow, in the steps before this action: the stack, its
fixtures and its startup order belong to that project.

```yaml
name: Playwright Tests

on:
  pull_request:
    branches: [main]

jobs:
  playwright:
    timeout-minutes: 30
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: InseeFr/rmes-githubactions-commons/.github/actions/playwright@main
```

A Playwright project living in its own directory, with its own manifest, and
testing a single browser:

```yaml
      - uses: InseeFr/rmes-githubactions-commons/.github/actions/playwright@main
        with:
          working-directory: e2e
          e2e-install-command: npm ci
          browsers: chromium
```

The tests see the environment of the step: an `env:` block on the `uses:` step
reaches the dev server Playwright starts.

### Inputs

| Input | Required | Default | |
| --- | --- | --- | --- |
| `node-version` | no | *empty* | Empty takes the shared version of [`setup-front`](#setup-front). |
| `pnpm-version` | no | *empty* | Empty takes the shared version of [`setup-front`](#setup-front). |
| `install-command` | no | `pnpm install --frozen-lockfile` | From the workspace root: the application the dev server runs. Empty skips pnpm entirely. |
| `working-directory` | no | `.` | Playwright project, relative to the workspace. |
| `e2e-install-command` | no | *empty* | From `working-directory`, when the Playwright project has its own manifest. Empty skips the step. |
| `browsers` | no | *empty* | Space separated. Empty = every browser Playwright installs by default. |
| `test-command` | no | `npx playwright test` | From `working-directory`, with `CI=true`. |
| `report-name` | no | `playwright-report` | Artifact name of the HTML report, read from `working-directory/playwright-report`. Empty = not uploaded. |
| `report-retention-days` | no | `30` | |

### What the caller must provide

- **The checkout** of the front-end.
- **The job timeout**: an action cannot declare one.
- **The services the tests need**, started in the steps before the action.

## setup-front

Node and pnpm, at the versions every RMéS front-end runs on. Those two versions
are written down **only here**, in the `Resolve the shared versions` step:
`quality-front` and `playwright` start with this action, and a job needing pnpm
without either of them calls it directly. No workflow passes a version, and no
manifest carries a `packageManager` field.

```yaml
      - uses: actions/checkout@v7
      - uses: InseeFr/rmes-githubactions-commons/.github/actions/setup-front@main
      - run: pnpm install
      - run: pnpm test:docker
```

**Bumping Node or pnpm** is a one-line change to `NODE_SHARED` / `PNPM_SHARED`,
picked up by every project on its next run. Both are majors on purpose: the
runner resolves the latest patch by itself. Node follows the active LTS line,
pnpm the current major.

**`packageManager` is deliberately not relied on.** `pnpm/action-setup` refuses
to start when the version is given both by the action and by a manifest, and a
version in a manifest is a version back in every repository — the opposite of
what this action is for. A repository carrying the field drops it.

Related, and worth knowing when bumping pnpm: since pnpm 11 the `overrides` and
`onlyBuiltDependencies` settings are read from `pnpm-workspace.yaml`, no longer
from `package.json`. A repository that left them in its manifest fails
`pnpm install --frozen-lockfile` on a recent pnpm, with *the current "overrides"
configuration doesn't match the value found in the lockfile*.

### Inputs

| Input | Required | Default | |
| --- | --- | --- | --- |
| `node-version` | no | *empty* | Empty takes the shared version. |
| `pnpm-version` | no | *empty* | Empty takes the shared version. |
| `working-directory` | no | `.` | Directory holding the lockfile the Node cache keys on. |
| `pnpm` | no | `true` | `false` sets Node up alone, without pnpm nor its store cache. |

Outputs: `node-version` and `pnpm-version`, the versions that were installed.

## Conventions

Callers pin `@main`: every repository picks up a change here on its next run.
The counterpart is that **a change to this repository must stay backwards
compatible** — add optional inputs, never rename or remove one, and never change
a default in a way that alters what an existing caller does. A genuinely
breaking change means a `v2` branch and a migration of the callers, one by one.

Inputs are always strings, including the ones a caller writes as booleans: test
them with `!= 'true'`, never with `!`, which reads the non-empty string `false`
as true.

Adding an ecosystem (gradle, poetry, cargo…) means adding one branch to the two
`case` blocks of `prepare-release` — how to read the current version, how to
write the next one — and one line to the table above. The rest of the action
(branch rules, version arithmetic, tag, release, warnings) is shared as is.
