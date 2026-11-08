# paperclip-backport-images

Manual GitHub Actions builder for the public
[`everlastingcrown/paperclip`](https://github.com/everlastingcrown/paperclip) fork.
Builds its Dockerfile's `production` target for `linux/amd64` and `linux/arm64`
and publishes one multi-platform image tag:

```text
ghcr.io/everlastingcrown/paperclip-backport:sha-<full-source-sha>
```

## Build

1. Merge the workflow onto this repository's default branch and enable Actions
   here. Leave Actions disabled on the Paperclip fork.
2. Open **Actions → Build backport image → Run workflow**.
3. Enter a trusted source branch, tag, or commit SHA as `source_ref`. The intended
   branch is `backport/v2026.1106.0-app-bridge`; prefer its full commit SHA for
   repeatable source selection. Only build revisions you trust to execute code.
4. Read the source SHA, image tag, and registry digest in the completed run summary.

The GitHub CLI equivalent is:

```sh
gh workflow run build.yml --repo everlastingcrown/paperclip-backport-images \
  -f source_ref=<full-source-sha>
```

Only manual dispatch starts builds. Commits and pull requests do not. A new run
cancels the previous run. The source commit is resolved once, then separate matrix
jobs build on native `ubuntu-24.04` (AMD64) and `ubuntu-24.04-arm` (ARM64) runners,
each with a 45-minute timeout. Both jobs push images by digest; a final job
combines their exact digests into the source-SHA tag after both builds succeed.
Source resolution and publishing have 5- and 15-minute timeouts respectively.
There is no persistent build cache.

Checkout needs no extra secret for this public fork. Publishing uses this
repository's `GITHUB_TOKEN` with `contents: read` and `packages: write`. The OCI
source label associates the package with this builder repository. If the package
already exists, ensure this repository has Actions write access to it. A private
source fork would require a separate, narrowly scoped read credential.

## Pull and verify

Copy the image index digest from the run summary. Verify that both platforms are
present, then pull and run on an AMD64 or ARM64 Docker host:

```sh
IMAGE=ghcr.io/everlastingcrown/paperclip-backport@sha256:<digest>
docker buildx imagetools inspect "$IMAGE"
docker pull "$IMAGE"
docker image inspect "$IMAGE" --format '{{.Os}}/{{.Architecture}} {{index .Config.Labels "org.opencontainers.image.revision"}}'
docker run --rm -p 3100:3100 "$IMAGE"
```

Confirm the index includes `linux/amd64` and `linux/arm64`. Docker selects the
host architecture when pulling. Repeat the pull/run check on each architecture,
confirming the expected platform, full source SHA, and successful startup in the
container logs. Open `http://localhost:3100` to check the application. This is a
temporary smoke test; use the deployment's normal volumes and configuration for
production. Pulling by digest selects the exact published image index, even if a SHA
tag is rebuilt. Source-SHA tags identify source revisions, not reproducible image
contents: the source Dockerfile also downloads dependencies and CLI tools.

Set package visibility deliberately: choose public if anonymous pulls are
required; otherwise authenticate Docker to GHCR before pulling. For a private
builder repository, configure an Actions budget with stop-usage enabled.

Local workflow validation does not replace a successful manual build and
pull/run check. Deployment and package/account settings are managed separately.
