[![Roc-Lang][roc_badge]][roc_link]

[roc_badge]: https://img.shields.io/endpoint?url=https%3A%2F%2Fpastebin.com%2Fraw%2FcFzuCCd7
[roc_link]: https://github.com/roc-lang/roc

# Setup Roc

A GitHub Action to download and setup the Roc compiler for Linux and macOS.

## Usage

Add this step to your CI workflow:

### For New Compiler Nightly Releases

```yaml
- uses: roc-lang/setup-roc@cbe782d6f165b89c87d99f50a59ac4f5f73b4427
  with:
    version: nightly-new-compiler
    nightly-tag: nightly-2026-June-27-127861d # remove nightly-tag to just get the latest one
```

### For Old Major Releases

```yaml
- uses: roc-lang/setup-roc@cbe782d6f165b89c87d99f50a59ac4f5f73b4427
  with:
    version: alpha4-rolling
```
> Note: we recommend using this @commit-sha way to specify the setup-roc version. This makes sure that the alpha4 release can not be altered if one of our github accounts is hacked.  

### For Old Nightly Releases

```yaml
- uses: roc-lang/setup-roc@cbe782d6f165b89c87d99f50a59ac4f5f73b4427
  with:
    # Note: nightly hashes are not verified because they are updated regularly.
    version: nightly
```

## Platform Support

This action supports the following platforms:

| OS | Architecture | Status |
|----|--------------|--------|
| Linux | x86_64 | ✅ |
| Linux | arm64 | ✅ |
| macOS | x86_64 (Intel) | ✅ |
| macOS | arm64 (Apple Silicon) | ✅ |
| Windows | x86_64 | ✅ |
| Windows | arm64 | ❌ |

Windows arm64 can be made available again with the next zig release (after 0.16.0).

## What it does

1. Detects your operating system and architecture
2. Downloads the appropriate Roc compiler release for your platform
3. Verifies the SHA256 checksum to ensure file integrity (skipped for nightly releases)
4. Extracts the compiler
5. Adds the Roc executable to the PATH
6. Restores and saves the Roc cache (`nightly-new-compiler` only)

For new-compiler nightlies that include glue support, this action also exports:

| Environment variable | Description |
| --- | --- |
| `ROC_GLUE_DIR` | Directory containing generated glue specs for the installed Roc version |
| `ROC_RUST_GLUE` | Rust glue spec path |
| `ROC_ZIG_GLUE` | Zig glue spec path |
| `ROC_C_GLUE` | C glue spec path |
| `ROC_GLUE_PLATFORM_URL` | URL of the matching glue platform package |

Example:

```yaml
- uses: roc-lang/setup-roc@<commit-sha-with-glue-support>
  with:
    version: nightly-new-compiler

- run: roc glue "$ROC_RUST_GLUE" ./platform/main.roc --output-dir ./platform
```

Pin to a `setup-roc` commit that includes glue support; older pinned commits do
not export these variables.

## Caching

For `nightly-new-compiler`, the action restores the Roc cache at the start of the job and saves it when the job succeeds (using [actions/cache](https://github.com/actions/cache)). Cached `roc test` results are part of that cache, so unchanged tests are not run again; the output shows `(cached)`.

- The cache lives in `$RUNNER_TEMP/roc-cache`, the action sets `ROC_CACHE_DIR` to this path. A `ROC_CACHE_DIR` you set yourself is respected.
- Each OS, architecture and job name gets its own cache. A run restores the newest cache from the same Roc nightly, or else from any other nightly: Roc keys its entries by the compiler's content, so entries from a different compiler are ignored, while a nightly with an unchanged compiler reuses them. Add `cache-key-suffix` when jobs share a name, for example matrix jobs:

```yaml
- uses: roc-lang/setup-roc@<commit-sha-with-cache-support>
  with:
    version: nightly-new-compiler
    cache-key-suffix: ${{ matrix.os }}
```

- The `cache-hit` output is `true` when the cache was restored with an exact key match.
- Set `cache: false` to disable caching.

The cache is saved automatically when the job succeeds. To also save it when the job fails (so tests that passed stay cached for the next run), add this as the last step of the job:

```yaml
- id: roc
  uses: roc-lang/setup-roc@<commit-sha-with-cache-support>
  with:
    version: nightly-new-compiler

# ... your steps, including `roc test` ...

- uses: actions/cache/save@<commit-sha>
  if: always() && steps.roc.outputs.cache-key != ''
  with:
    path: ${{ steps.roc.outputs.cache-path }}
    key: ${{ steps.roc.outputs.cache-key }}
```

When the job succeeds, the automatic save finds this key already reserved and does nothing.

### Warm the cache on your default branch

GitHub only lets a run restore caches from its own branch, its base branch and the default branch, and a fork PR cannot write to the default branch's caches. A PR's first run is therefore cold unless a run on the default branch has saved a cache. Run your Roc workflow on the default branch too, and on a schedule, so the cache on the default branch follows the latest nightly:

```yaml
on:
  pull_request:
  push:
    branches: [main]
  schedule:
    - cron: '0 18 * * *' # nightlies are published around 16:00-17:00 UTC
```

Matrix jobs need a distinct `cache-key-suffix` so that the entries saved on the default branch match the same jobs on PR branches. GitHub deletes caches unused for 7 days and evicts the least recently used ones when a repository exceeds 10 GB.

## Security

For major releases, the action verifies the SHA256 checksum of the downloaded file to ensure it hasn't been tampered with. If the checksum doesn't match, the action will fail.

For nightly releases, SHA256 verification is skipped because the files are updated regularly.
