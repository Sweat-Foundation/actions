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

### Optional: store the release binary on a DAO contract

Set `dao-account-id` (and pass `secrets.proposer-private-key`) to also call `store_blob` on a
[sputnik-dao](https://github.com/near-daos/sputnik-dao-contract) DAO contract after the release is published,
via [`near-cli-rs`](https://github.com/near/near-cli-rs) (installed checksum-verified) signed non-interactively
with `sign-with-plaintext-private-key`.

```yaml
jobs:
  release:
    uses: Sweat-Foundation/actions/.github/workflows/release.yml@v1
    with:
      files: |
        res/sweat_jar.wasm
      dao-account-id: sweat.sputnik-dao.near
      proposer-account-id: bot.sweat.near
      blob-path: res/sweat_jar.wasm
    secrets:
      proposer-private-key: ${{ secrets.PROPOSER_PRIVATE_KEY }}
```

| Name                   | Required             | Default | Description                                                                 |
|------------------------|-----------------------|---------|-------------------------------------------------------------------------------|
| `dao-account-id`       | no                    | `''`    | DAO contract to call `store_blob` on. Leave empty to skip this job entirely.  |
| `proposer-account-id`  | yes if `dao-account-id` set | `''` | Account that signs and pays for the call.                                |
| `blob-path`            | yes if `dao-account-id` set | `''` | File to upload as the blob.                                              |
| `network`              | no                    | `mainnet` | NEAR network.                                                              |
| `prepaid-gas`          | no                    | `300 TGas` | Gas attached to the call.                                                |
| `attached-deposit`     | no                    | `''` (auto) | Deposit attached to the call. Leave empty to auto-calculate the exact minimum from [NEAR's storage staking price](https://docs.near.org/protocol/storage/storage-staking) (1e19 yoctoNEAR/byte) for the file at `blob-path`, plus the 32-byte bookkeeping overhead `store_blob` charges per blob (verified against `near-daos/sputnik-dao-contract`'s source). Set explicitly if the target DAO contract charges differently. |
| `secrets.proposer-private-key` | yes if `dao-account-id` set | | NEAR private key (`ed25519:...`) for `proposer-account-id`. |

### Optional: notify Slack

Set `contract-name` (and pass `secrets.slack-webhook-url`) to post a message to Slack after the release is
published, with the contract name, version, wasm sha256 hash, and a link to the release. The message body reuses
the GitHub release's auto-generated notes (capped to 300 characters) as the description.

```yaml
jobs:
  release:
    uses: Sweat-Foundation/actions/.github/workflows/release.yml@v1
    with:
      files: |
        res/sweat_jar.wasm
      contract-name: sweat_jar
      wasm-path: res/sweat_jar.wasm
    secrets:
      slack-webhook-url: ${{ secrets.SLACK_WEBHOOK_URL }}
```

| Name                        | Required                    | Default | Description                                                        |
|------------------------------|------------------------------|---------|----------------------------------------------------------------------|
| `contract-name`              | no                           | `''`    | Contract/project name shown in the message. Leave empty to skip this job entirely. |
| `wasm-path`                  | yes if `contract-name` set   | `''`    | File to compute the sha256 hash of.                                 |
| `secrets.slack-webhook-url`  | yes if `contract-name` set   |         | Slack incoming webhook URL.                                         |

## remove-blob

Manually invokable: calls `remove_blob` on a sputnik-dao contract to delete a previously stored blob and refund its
storage deposit to the account that originally stored it (`remove_blob` only allows the original storer to remove
it). Not triggered automatically — wire it behind `workflow_dispatch` in the calling repo so it only runs when
someone deliberately runs it.

```yaml
name: Remove DAO Blob

on:
  workflow_dispatch:
    inputs:
      hash:
        description: Base58 sha256 hash of the blob to remove (as returned by store_blob).
        required: true
        type: string

jobs:
  remove-blob:
    uses: Sweat-Foundation/actions/.github/workflows/remove-blob.yml@v1
    with:
      dao-account-id: sweat.sputnik-dao.near
      proposer-account-id: bot.sweat.near
      hash: ${{ inputs.hash }}
    secrets:
      proposer-private-key: ${{ secrets.PROPOSER_PRIVATE_KEY }}
```

| Name                          | Required | Default    | Description                                                        |
|--------------------------------|----------|------------|------------------------------------------------------------------------|
| `dao-account-id`               | yes      |            | DAO contract to call `remove_blob` on.                                |
| `proposer-account-id`          | yes      |            | Account that originally stored the blob; must match to succeed.       |
| `hash`                         | yes      |            | Base58 sha256 hash of the blob, as returned by `store_blob`.          |
| `network`                      | no       | `mainnet`  | NEAR network.                                                          |
| `prepaid-gas`                  | no       | `100 TGas` | Gas attached to the call.                                              |
| `secrets.proposer-private-key` | yes      |            | NEAR private key (`ed25519:...`) for `proposer-account-id`.           |

`remove_blob` isn't `#[payable]`, so no deposit is attached (near-sdk rejects any nonzero deposit on a non-payable
method) — the contract itself refunds the original storage deposit to `proposer-account-id`.

## Versioning

Tag releases of this repo (`v1`, `v2`, ...) and have callers pin to a tag. Move the major tag (`v1`) forward as
non-breaking changes land; cut a new major tag for breaking changes to inputs.
