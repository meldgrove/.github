# Meldgrove shared files

Files that every [Meldgrove](https://github.com/meldgrove) repository uses:

- [`.github/workflows/release.yml`](.github/workflows/release.yml): the release
  workflow. Each repository calls it from a short workflow of its own.
- [`SECURITY.md`](SECURITY.md): the security policy of repositories that have
  none of their own.

## Release workflow

A maintainer starts a release in the repository: Actions > Release > Run
workflow, with patch, minor or major, "Release candidate", and "Dry run". The
workflow:

1. stops unless it was started from `main`, CI passed for the commit, and
   something changed since the last release;
2. stops when the chosen version is too small for a merged pull request
   labelled `breaking` (see [version numbers](#version-numbers));
3. raises the latest `vX.Y.Z` tag by the chosen part, tags the commit, and
   pushes only the tag. No file holds the version;
4. builds and starts the image on amd64 and arm64, and publishes it as
   `X.Y.Z`, `X.Y`, from 1.0.0 also `X`, and `latest`;
5. attests where the image was built and creates the GitHub Release, with
   notes from the pull request titles;
6. moves the branch `stable` to the released commit. Platforms that read
   `compose.yaml` from the repository follow `stable`, so that file belongs
   to the newest release and not to unreleased work on `main`.

A release candidate publishes only `X.Y.Z-rc.N` and leaves `latest` and
`stable` alone. A dry run builds and starts both images and shows the version
and notes, without pushing or publishing. Pull requests that change the image
build run the same build without publishing.

### Version numbers

The version says whether a release installs without anyone acting:

| | Below 1.0.0 | From 1.0.0 |
| --- | --- | --- |
| Operators must act (label `breaking`) | minor | major |
| New features, automatic migrations | patch | minor |
| Fixes, dependency updates | patch | patch |

A release that operators must act on starts a new series. An image tag that
moves within a series (`0.3`, from 1.0.0 the major version `1`) therefore
never brings a release that needs manual steps, and stays current until the
next such release. The workflow refuses a smaller version when a pull request
merged since the last release is labelled `breaking`.

### Calling it

`.github/workflows/release.yml` of a repository:

```yaml
name: Release

on:
  workflow_dispatch:
    inputs:
      bump:
        description: Version part to raise (the first release starts from 0.0.0)
        type: choice
        options: [patch, minor, major]
        default: patch
      candidate:
        description: "Release candidate: publish vX.Y.Z-rc.N for testing (no latest)"
        type: boolean
        default: false
      dry_run:
        description: "Dry run: build, show the version and notes, publish nothing"
        type: boolean
        default: false
  pull_request:
    paths:
      - Dockerfile
      - .dockerignore
      - .github/workflows/release.yml

permissions:
  contents: read

concurrency:
  group: ${{ github.event_name == 'pull_request' && format('release-pr-{0}', github.ref) || 'release' }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  release:
    permissions:
      actions: read # find the CI run of the commit
      contents: write # push the tag, create the GitHub Release, move stable
      pull-requests: read # read the labels of the merged pull requests
      packages: write # push the image, store the attestation
      id-token: write # sign the attestation
      attestations: write
      artifact-metadata: write
    uses: meldgrove/.github/.github/workflows/release.yml@<commit SHA> # v1.0.0
    with:
      bump: ${{ inputs.bump || 'patch' }}
      candidate: ${{ inputs.candidate || false }}
      dry_run: ${{ inputs.dry_run || false }}
      image: ghcr.io/meldgrove/example
      upgrade_doc: docs/deployment.md
      title: Meldgrove Example
      description: One sentence about the project.
```

| Input | Meaning |
| --- | --- |
| `image` | The image to publish, e.g. `ghcr.io/meldgrove/logbuch`. |
| `upgrade_doc` | Path of the upgrade instructions; the release notes link it. |
| `title`, `description` | Title and one-sentence description of the image. |
| `target` | Dockerfile stage to publish. Default: the last stage. |
| `ci_workflow` | File name of the workflow that must have passed for the commit. Default `ci.yml`. |
| `license` | SPDX identifier of the license. Empty while the repository has none. |
| `sentry_org`, `sentry_project` | Sentry project that releases upload their source maps to. Empty uploads nothing. |
| `preview`, `revision` | Publish the commit `revision` on `main` as a preview (see [Previews](#previews)). |

With `sentry_project`, a release passes the caller's secret `SENTRY_AUTH_TOKEN`
(`secrets: SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}` in the calling
job) to the image build as the build secret `sentry_auth_token`, with the
build arguments `SENTRY_ORG` and `SENTRY_PROJECT`; the Dockerfile uploads the
source maps when the secret is there. Dry runs and pull requests upload
nothing. A release without the secret warns and uploads nothing.

The repository also needs `.github/release.yml` (sections of the release
notes), the label `breaking`, a Dockerfile whose runtime stage takes
`ARG APP_VERSION`, and immutable releases switched on.

### Previews

A preview publishes each commit on `main` once CI passed for it, for
installations on the Preview channel of the
[Meldgrove Updater](https://github.com/meldgrove/updater). Its version is the
next patch after the newest release that the commit contains, plus the
number of commits since that release: `0.2.1-preview.3` is three commits
past 0.2.0, and sorts before 0.2.1 and 0.3.0. The image gets that tag and
`preview`, which moves only to newer previews. A preview creates no git tag
and no GitHub Release, leaves `stable` alone, and uploads no source maps.
A released commit, or one published already, publishes nothing.

A preview publishes the newest commit on `main` that passed CI, so a run
that ends late or replaces a pending one never leaves the newest commit
out. Callers run one preview at a time (`concurrency` on the job below).

`.github/workflows/preview.yml` of a repository, with the same values as its
release caller (`name` in `workflows:` is the `name:` of its CI workflow):

```yaml
name: Preview

on:
  workflow_run:
    workflows: [CI]
    types: [completed]
    branches: [main]

permissions:
  contents: read

jobs:
  preview:
    # Pushes to main only; CI also runs for pull requests.
    if: github.event.workflow_run.conclusion == 'success' && github.event.workflow_run.event == 'push'
    # One preview at a time. On the job, not the workflow: a failed CI run,
    # which skips the job, must not replace a waiting preview.
    concurrency:
      group: preview
      cancel-in-progress: false
    permissions:
      actions: read # find the CI run of the commit
      contents: write # requested by the shared jobs; a preview pushes nothing
      pull-requests: read
      packages: write # push the image
      id-token: write # sign the attestation
      attestations: write
      artifact-metadata: write
    uses: meldgrove/.github/.github/workflows/release.yml@<commit SHA> # v1.1.0
    with:
      preview: true
      revision: ${{ github.event.workflow_run.head_sha }}
      image: ghcr.io/meldgrove/example
      upgrade_doc: docs/deployment.md
      title: Meldgrove Example
      description: One sentence about the project.
```

### Checking an image

The workflow in this repository signs the attestation of every image of a
public repository, so the check names it:

```sh
gh attestation verify oci://ghcr.io/meldgrove/<project>:X.Y.Z \
  --repo meldgrove/<project> --signer-repo meldgrove/.github
```

Each release of a public repository runs this check itself after
publishing. Images of a private repository get no attestation: GitHub offers attestations for private
repositories only on Enterprise Cloud, so the workflow skips both steps there
and the release notes leave out the check.

### Changing the workflow

1. Open a pull request here.
2. Test it from a project: in a branch there, point `uses:` at the commit of
   the pull request and open a pull request. It builds and starts both images
   with the changed workflow, without publishing.
3. After merging, tag the merge commit (`vX.Y.Z`; a major version when the
   callers must change) and push the tag. Dependabot then proposes the new
   commit to every project.

Projects pin the workflow by commit SHA, so a change here reaches a project
only with such an update.
