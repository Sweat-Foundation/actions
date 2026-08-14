# actions

Reusable GitHub Actions workflows shared across Sweat-Foundation repos.

## release

Thin wrapper around [`softprops/action-gh-release`](https://github.com/softprops/action-gh-release) (SHA-pinned) for
publishing a GitHub release when a tag is pushed. Assumes the tagged commit already has the artifacts to attach
(e.g. checked-in build outputs); it does not build anything itself.

```yaml
name: Release

on:
  push:
    tags: ['v*']

jobs:
  release:
    uses: Sweat-Foundation/actions/.github/workflows/release.yml@v1
    with:
      files: |
        res/sweat_jar.wasm
        res/sweat_jar_abi.json
        res/sweat_jar_abi.zst
```

### Inputs

| Name              | Required | Default                          | Description                                                          |
|-------------------|----------|-----------------------------------|------------------------------------------------------------------------|
| `files`           | yes      |                                    | Newline-separated list of file paths/globs to attach to the release.  |
| `tag`             | no       | `${{ github.ref_name }}`          | Tag to release.                                                       |
| `title`           | no       | tag                                | Release title.                                                        |
| `generate-notes`  | no       | `true`                             | Auto-generate release notes from merged PRs since the previous tag.   |

## Versioning

Tag releases of this repo (`v1`, `v2`, ...) and have callers pin to a tag. Move the major tag (`v1`) forward as
non-breaking changes land; cut a new major tag for breaking changes to inputs.
