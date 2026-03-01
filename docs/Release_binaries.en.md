# Argus Release Step 4 (Prebuilt Binaries)

This document describes Step 4 of the release pipeline:
build and attach prebuilt binaries to a GitHub Release.

## Workflow file

- `.github/workflows/release-binaries.yml`

## Trigger rules

- Automatic: push tag matching `v*`
- Manual: GitHub Actions `workflow_dispatch` (optionally set `tag`)

## What the workflow does

1. Checks out repository code at the target tag
2. Sets up Go version from `go.mod`
3. Runs `go test ./...`
4. Cross-compiles:
   - `linux/amd64`
   - `darwin/arm64`
5. Uses reproducible flags:
   - `-trimpath`
   - `-ldflags "-s -w"`
6. Tries `CGO_ENABLED=0` first, falls back if needed
7. Produces release asset filenames:
   - `argus-linux-amd64`
   - `argus-macos-arm64`
8. Uploads workflow artifacts
9. Creates or updates the GitHub Release and attaches the binaries

## How to trigger for v1.0.0-validation

```bash
git tag v1.0.0-validation
git push origin v1.0.0-validation
```

Release title configured in workflow:

- `Argus Validation Protocol v1.0`
