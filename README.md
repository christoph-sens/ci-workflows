# ci-workflows

Reusable GitHub Actions workflows shared by the repositories of
[christoph-sens](https://github.com/christoph-sens) and [treetrail](https://github.com/treetrail)
(Kotlin/Gradle, Node and Flutter): one place to maintain CI instead of copies that drift apart.

| Workflow | Purpose |
| --- | --- |
| [`gradle-ci.yml`](.github/workflows/gradle-ci.yml) | Build + tests, optional integration tests, optional Docker image with Trivy scan, CodeQL (Kotlin/Java and workflows), Gradle dependency submission, dependency review on pull requests |
| [`node-ci.yml`](.github/workflows/node-ci.yml) | `npm ci` + npm scripts (default `check build`), CodeQL (JavaScript/TypeScript and workflows), dependency review on pull requests |
| [`flutter-ci.yml`](.github/workflows/flutter-ci.yml) | Formatting, localizations, `flutter analyze` + tests, Android app bundle, unsigned iOS build, CodeQL (workflows), dependency review on pull requests |
| [`dependabot-auto-merge.yml`](.github/workflows/dependabot-auto-merge.yml) | Auto-merge (squash) for Dependabot minor/patch PRs once CI is green |
| [`security-alert-issues.yml`](.github/workflows/security-alert-issues.yml) | One issue per vulnerable package, kept in sync with the open Dependabot alerts |
| [`container-release.yml`](.github/workflows/container-release.yml) | Release of a Spring Boot app as container image: on tag `vX.Y.Z` build + test, Trivy gate, push to GHCR, provenance + SBOM attestations, GitHub release |

## Usage

`.github/workflows/ci.yml` in a calling repository:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: "0 5 * * 1"

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  ci:
    uses: christoph-sens/ci-workflows/.github/workflows/gradle-ci.yml@<commit-sha> # v1.0.0
    permissions:
      contents: write        # dependency submission on push to main
      security-events: write # CodeQL and Trivy results
    with:
      integration-tests: true
```

For a Node repository use `node-ci.yml` instead; it needs only `contents: read` and
`security-events: write`:

```yaml
jobs:
  ci:
    uses: christoph-sens/ci-workflows/.github/workflows/node-ci.yml@<commit-sha> # v1.3.0
    permissions:
      contents: read
      security-events: write # CodeQL results
```

For a Flutter app use `flutter-ci.yml`; it needs the same permissions as `node-ci.yml` and the
exact Flutter version:

```yaml
jobs:
  ci:
    uses: christoph-sens/ci-workflows/.github/workflows/flutter-ci.yml@<commit-sha> # v1.5.0
    permissions:
      contents: read
      security-events: write # CodeQL results
    with:
      flutter-version: 3.47.8
```

Keep the workflow name `CI`: together with the job id `dependency-submission` it identifies the
dependency graph snapshot.

`.github/workflows/dependabot-auto-merge.yml`:

```yaml
name: Dependabot auto-merge

on: pull_request

permissions:
  contents: write
  pull-requests: write

jobs:
  auto-merge:
    uses: christoph-sens/ci-workflows/.github/workflows/dependabot-auto-merge.yml@<commit-sha> # v1.4.0
    secrets:
      merge-token: ${{ secrets.AUTO_MERGE_TOKEN }}
```

A merge done with `GITHUB_TOKEN` starts no workflows: the default branch gets no CI run, so no
dependency graph snapshot either, and the next pull request fails `dependency-review` because every
dependency looks new. `merge-token` avoids that: a fine-grained personal access token with the
repository permissions **Contents: Read and write** and **Pull requests: Read and write**, stored as
**Dependabot** secret `AUTO_MERGE_TOKEN` (runs started by Dependabot cannot read Actions secrets).
Without it the job still works and logs a warning. Add `workflow_dispatch:` to the `on:` of `ci.yml`
to start CI on the default branch by hand when a snapshot is missing.

`.github/workflows/release.yml` of an app with a Dockerfile `prebuilt` target:

```yaml
name: Release

on:
  push:
    tags: ["v*"]
  # Manual run = dry run: builds, tests and scans the image, never pushes.
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false

jobs:
  release:
    uses: christoph-sens/ci-workflows/.github/workflows/container-release.yml@<commit-sha> # v1.1.0
    permissions:
      contents: write          # GitHub release
      packages: write          # push to GHCR
      id-token: write          # Sigstore signing of the attestations
      attestations: write
      artifact-metadata: write
    with:
      docker-jar-dir: bootstrap/build/libs
```

`.github/workflows/security-alert-issues.yml`:

```yaml
name: Security alert issues

on:
  schedule:
    - cron: "0 6 * * *"
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}
  cancel-in-progress: false

jobs:
  sync:
    uses: christoph-sens/ci-workflows/.github/workflows/security-alert-issues.yml@<commit-sha> # v1.2.1
    permissions:
      issues: write
    secrets:
      alerts-token: ${{ secrets.DEPENDABOT_ALERTS_TOKEN }}
```

### Inputs of `gradle-ci.yml`

| Input | Default | Description |
| --- | --- | --- |
| `java-version` | `25` | Temurin JDK version |
| `gradle-tasks` | `build` | Gradle tasks of the main build step, space-separated |
| `integration-tests` | `false` | Run `./gradlew integrationTest` and compile that source set for CodeQL |
| `docker-jar-dir` | `""` | Directory of the boot jar; when set, the Dockerfile's `prebuilt` target is built with it as build context `app` and scanned with Trivy |

### Inputs of `node-ci.yml`

| Input | Default | Description |
| --- | --- | --- |
| `node-version` | `24` | Node.js version for `actions/setup-node` |
| `npm-scripts` | `check build` | npm scripts run in order after `npm ci`, space-separated |

GitHub reads `package-lock.json` natively, so dependency review works without a dependency
submission job. The license allow-list additionally permits OFL-1.1 (web fonts), BlueOak-1.0.0 and
Python-2.0.

### Inputs of `flutter-ci.yml`

| Input | Default | Description |
| --- | --- | --- |
| `flutter-version` | (required) | Exact Flutter version of the stable channel, e.g. `3.47.8` |
| `java-version` | `25` | Temurin JDK version for the Android build |
| `ios` | `true` | Build for iOS (`flutter build ios --no-codesign`) on `macos-latest` |

The `build` job checks `dart format`, runs `flutter gen-l10n` if the repository has an `l10n.yaml`
(it fails on untranslated messages and on generated code that is not committed), then
`flutter analyze` and `flutter test`. The `android` job builds a release app bundle with the
signing configuration of the project. Like for npm, GitHub reads `pubspec.lock` natively; CodeQL
covers only the workflows, because it does not support Dart. The jobs report as `ci / build`,
`ci / android`, `ci / ios`, `ci / codeql-actions` and `ci / dependency-review`.

### Status checks

The jobs report as `<caller job> / <job>`, e.g. `ci / build`, `ci / codeql`,
`ci / codeql-actions` and `ci / dependency-review`. Use these names as required status checks in
branch rulesets.

### Dependency review

On pull requests the Gradle dependency graph is submitted for the PR head, then
[dependency-review-action](https://github.com/actions/dependency-review-action) fails the PR if it
adds or updates a dependency with a known **high/critical** vulnerability or a license outside the
allow-list: permissive licenses plus weak copyleft (LGPL, EPL, MPL, CDDL, GPL-2.0 with Classpath
exception). Strong copyleft (GPL, AGPL) and source-available licenses (SSPL, BUSL) are blocked.
Packages whose license data on GitHub is missing or unusable are listed in
`allow-dependencies-licenses` and skip the license check. PRs from forks skip both jobs (no write
token for the submission).

### Security alert issues

Dependabot opens security pull requests only for dependencies it can update in a manifest. Alerts
for dependencies managed by a BOM (e.g. Jackson via the Spring Boot BOM) or known only from the
Gradle dependency graph snapshot get no pull request and are easy to miss.
`security-alert-issues.yml` therefore keeps one issue labelled `security` per vulnerable package:
created for new alerts, updated when they change, reopened when alerts come back and closed once no
alert is open any more (fixed or dismissed). The issue body lists every alert with advisory,
severity, vulnerable range and fixed version. Input `dry-run` (default `false`) only logs the
changes. Dependabot alerts must be enabled in the repository.

`GITHUB_TOKEN` cannot read Dependabot alerts, so the caller passes `alerts-token`: a fine-grained
personal access token with only the repository permission **Dependabot alerts: Read-only** for the
calling repositories, stored as repository secret `DEPENDABOT_ALERTS_TOKEN`. It is used for nothing
but reading the alerts; issues are written with `GITHUB_TOKEN`.

### Inputs of `container-release.yml`

| Input | Default | Description |
| --- | --- | --- |
| `java-version` | `25` | Temurin JDK version |
| `gradle-tasks` | `build` | Gradle tasks that build and test the app |
| `docker-jar-dir` | (required) | Directory of the boot jar, build context `app` of the `prebuilt` target |
| `image-name` | `ghcr.io/<owner>/<repository>` | Image name without tag |

The image is tagged with the version from the Git tag (`v1.2.3` → `1.2.3`). It is only pushed if
Trivy finds no HIGH/CRITICAL vulnerability that has a fix. Deploy it by digest (see the GitHub
release notes) and verify it with:

```bash
gh attestation verify oci://ghcr.io/<owner>/<repository>@sha256:<digest> --repo <owner>/<repository> \
  --signer-repo christoph-sens/ci-workflows
```

`--signer-repo` is needed because the attestation is signed by this repository's reusable
workflow, not by the app repository itself.

## Versioning

Reference the workflows by full commit SHA with the version as a comment, like any other action;
Dependabot (`github-actions` ecosystem) keeps both up to date.

- Every change to a reusable workflow merged to `main` is released automatically by
  [`release.yml`](.github/workflows/release.yml): as a **patch** by default, as **minor** or
  **major** when the pull request has the label `release:minor` (new workflow or input) or
  `release:major` (breaking change for callers).
- The Release workflow can also be run manually with the desired `bump`.
- Releases are immutable: a published version can never be changed.

## License

[Apache License 2.0](LICENSE)
