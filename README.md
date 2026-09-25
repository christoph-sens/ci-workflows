# ci-workflows

Reusable GitHub Actions workflows shared by the Kotlin/Gradle repositories of
[christoph-sens](https://github.com/christoph-sens): one place to maintain CI instead of five
copies that drift apart.

| Workflow | Purpose |
| --- | --- |
| [`gradle-ci.yml`](.github/workflows/gradle-ci.yml) | Build + tests, optional integration tests, optional Docker image with Trivy scan, CodeQL (Kotlin/Java and workflows), Gradle dependency submission, dependency review on pull requests |
| [`dependabot-auto-merge.yml`](.github/workflows/dependabot-auto-merge.yml) | Auto-merge (squash) for Dependabot minor/patch PRs once CI is green |
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
    uses: christoph-sens/ci-workflows/.github/workflows/dependabot-auto-merge.yml@<commit-sha> # v1.0.0
```

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

### Inputs of `gradle-ci.yml`

| Input | Default | Description |
| --- | --- | --- |
| `java-version` | `25` | Temurin JDK version |
| `gradle-tasks` | `build` | Gradle tasks of the main build step, space-separated |
| `integration-tests` | `false` | Run `./gradlew integrationTest` and compile that source set for CodeQL |
| `docker-jar-dir` | `""` | Directory of the boot jar; when set, the Dockerfile's `prebuilt` target is built with it as build context `app` and scanned with Trivy |

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
gh attestation verify oci://ghcr.io/<owner>/<repository>@sha256:<digest> --repo <owner>/<repository>
```

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
