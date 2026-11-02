# swift-ci

Continuous Integration tools for Swift repos: one reusable GitHub Actions workflow and one base SwiftLint config, shared by the Haelix Swift packages. A fix here reaches every repo that uses it.

## Using it in a package

`.github/workflows/ci.yml`. The repo owns its triggers and branch rules; this repo owns how the checks run.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  swift:
    uses: Haelix-Code/swift-ci/.github/workflows/swift-package.yml@v1
    with:
      scheme: MyLibrary   # builds for the iOS Simulator; omit to skip
```

`.swiftlint.yml`: inherit the shared rules, then add only this repo's exceptions.

```yaml
parent_config: https://raw.githubusercontent.com/Haelix-Code/swift-ci/v1/swiftlint/base.yml

# e.g. this repo requires trailing commas
trailing_comma:
  mandatory_comma: true
```

A rule configured in the repo replaces the shared one; `excluded` paths add to the shared list. SwiftLint caches the fetched config in `.swiftlint/RemoteConfigCache` and adds that to `.gitignore` itself.

If SwiftLint can't fetch the config and has no cached copy, it prints `Unable to load remote config` and falls back to its default rules. That ignores the repo's own config too, so it reports violations in `.build` and on maths identifiers. If you see those, check the URL and tag, and that this repo is public.

Don't put a `paths:` filter on the caller's `pull_request` trigger: a required check that a PR never triggers stays pending and blocks the merge.

## Inputs

| Input | Default | |
|---|---|---|
| `scheme` | `""` | Scheme to build for the iOS Simulator. Empty skips the build. |
| `xcode-version` | `26.x` | Any version `maxim-lobanov/setup-xcode` accepts. |
| `swiftlint-version` | `0.65.1` | Tag of `ghcr.io/realm/swiftlint`. Keep in step with local installs. |
| `lint` | `true` | Run SwiftLint. |

## Jobs and required checks

- **SwiftLint**: `swiftlint lint --strict` in the pinned SwiftLint container on Linux.
- **swift test (macOS) + iOS build**: `swift test --parallel`, then the iOS Simulator build if `scheme` is set.

GitHub names called jobs `<caller job> / <job>`, so with the caller above the checks are `swift / SwiftLint` and `swift / swift test (macOS) + iOS build`. Branch protection's required checks need those names.

## Versioning

Callers pin `@v1`, which applies to both the workflow and the SwiftLint config URL.

- **Non-breaking change** (fix, SwiftLint bump, new optional input): commit, then move the tag so every repo picks it up on its next run.

  ```sh
  git tag -f v1 && git push -f origin v1
  ```

- **Breaking change** (new rule that fails existing code, renamed input or check): tag `v2` and move repos across one at a time.

This repo must stay **public**: public repos can't call workflows stored in a private repo, and SwiftLint fetches the base config without credentials.
