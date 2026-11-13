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

With `sentry_project`, a release passes the caller's secret `SENTRY_AUTH_TOKEN`
(`secrets: SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}` in the calling
job) to the image build as the build secret `sentry_auth_token`, with the
build arguments `SENTRY_ORG` and `SENTRY_PROJECT`; the Dockerfile uploads the
source maps when the secret is there. Dry runs and pull requests upload
nothing. A release without the secret warns and uploads nothing.

The repository also needs `.github/release.yml` (sections of the release
notes), the label `breaking`, a Dockerfile whose runtime stage takes
`ARG APP_VERSION`, and immutable releases switched on.

### Checking an image

The workflow in this repository signs the attestation of every image, so the
check names it:

```sh
gh attestation verify oci://ghcr.io/meldgrove/<project>:X.Y.Z \
  --repo meldgrove/<project> --signer-repo meldgrove/.github
```

Each release runs this check itself after publishing.

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
