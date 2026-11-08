# Backport image build handoff

Use this repository as an independent, manual GitHub Actions builder. Leave Actions disabled on the Paperclip fork; checking out its source does not run its workflows.

## Implementation

- Confirmed source: `everlastingcrown/paperclip`, intended branch `backport/v2026.1106.0-app-bridge`, deployment architecture `linux/amd64`.
- Add `.github/workflows/build.yml` on this repository's default branch with only `workflow_dispatch`. Fix the source repository in the workflow and accept a branch, tag, or commit SHA input. Prefer an exact SHA for repeatable builds.
- Check out that source using `actions/checkout` with `repository`, `ref`, and `persist-credentials: false`. Public source needs no extra secret. Private source needs a narrowly scoped read credential; this builder's `GITHUB_TOKEN` cannot read another private repository.
- Build with explicit `context: .` and `target: production`, confirming that target and any required build arguments exist in the backport Dockerfile. Start with one standard Linux runner and the deployment architecture; avoid emulation and a multi-architecture matrix.
- Resolve the source SHA with `git rev-parse HEAD`; use it for image tags, the OCI revision label, and supported build-commit arguments. `github.sha` identifies the builder repository commit, not the source commit.
- Publish `ghcr.io/everlastingcrown/paperclip-backport:sha-<full-source-sha>`, optionally also tagging `backport`. Use this repository's `GITHUB_TOKEN` with only `contents: read` and `packages: write`. Associate the package with this builder repository.
- Pin actions to reviewed full commit SHAs. Set a 45-minute timeout and a fixed concurrency group with `cancel-in-progress: true`. Build trusted source revisions only. Keep push, PR, schedule, release, and deployment triggers absent.

## Acceptance and usage

Manually dispatch a known source SHA, pull/run the image on the target architecture, and include its source SHA and registry digest in the run summary. Confirm ordinary commits do not trigger builds. Set package visibility deliberately if anonymous pulls are required.

For a private builder, configure an Actions budget with stop-usage enabled. Standard GitHub-hosted runner compute is free for public repositories; manual execution still avoids wasted usage. Add caching only when repeated builds justify it.

The workflow is implemented in `.github/workflows/build.yml`; see README.md for
dispatch and verification instructions. It publishes only the full source-SHA
tag. The source Dockerfile's `production` target and `PAPERCLIP_BUILD_COMMIT`
argument were confirmed during implementation. Actions settings, package
visibility, and billing settings have not been changed. Operational acceptance
still requires a successful manual build and pull/run check.
