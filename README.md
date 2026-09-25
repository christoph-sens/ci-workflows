# ci-workflows

Reusable GitHub Actions workflows shared by the Kotlin/Gradle repositories of
[christoph-sens](https://github.com/christoph-sens): one place to maintain CI instead of five
copies that drift apart.

| Workflow | Purpose |
| --- | --- |
| [`gradle-ci.yml`](.github/workflows/gradle-ci.yml) | Build + tests, optional integration tests, optional Docker image with Trivy scan, CodeQL (Kotlin/Java and workflows), Gradle dependency submission |
| [`dependabot-auto-merge.yml`](.github/workflows/dependabot-auto-merge.yml) | Auto-merge (squash) for Dependabot minor/patch PRs once CI is green |

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

### Inputs of `gradle-ci.yml`

| Input | Default | Description |
| --- | --- | --- |
| `java-version` | `25` | Temurin JDK version |
| `gradle-tasks` | `build` | Gradle tasks of the main build step, space-separated |
| `integration-tests` | `false` | Run `./gradlew integrationTest` and compile that source set for CodeQL |
| `docker-jar-dir` | `""` | Directory of the boot jar; when set, the Dockerfile's `prebuilt` target is built with it as build context `app` and scanned with Trivy |

### Status checks

The jobs report as `<caller job> / <job>`, e.g. `ci / build`, `ci / codeql` and
`ci / codeql-actions`. Use these names as required status checks in branch rulesets.

## Versioning

Reference the workflows by full commit SHA with the version as a comment, like any other action;
Dependabot (`github-actions` ecosystem) keeps both up to date.

- Every change to a reusable workflow merged to `main` is released automatically as a **patch**
  version by [`release.yml`](.github/workflows/release.yml).
- Changes that add inputs (**minor**) or break callers (**major**) are released by running the
  Release workflow manually with the matching `bump`.
- Releases are immutable: a published version can never be changed.

## License

[Apache License 2.0](LICENSE)
