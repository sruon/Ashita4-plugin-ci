# Ashita4-plugin-ci

Reusable workflow to build Ashita v4 plugins on GitHub Actions: 32-bit MSVC, CMake and Ninja, against the Ashita SDK pulled from upstream at a pinned tag.

- `format`: `clang-format --dry-run --Werror` on every tracked C/C++ file, using the repo's `.clang-format`.
- `build`: fetches `plugins/sdk` from `AshitaXI/Ashita-v4beta`, sets `ASHITA4_SDK_PATH`, builds the targets and uploads the DLL as an artifact.
- `release`: attaches the DLL to a release when `release-tag` is set.

## Usage

`.github/workflows/build.yml` in the plugin repo:

```yaml
name: build

on:
  push:
  pull_request:

jobs:
  build:
    uses: sruon/Ashita4-plugin-ci/.github/workflows/build-plugin.yml@v1
    with:
      targets: myplugin
      output-dir-variable: MYPLUGIN_OUTPUT_DIR
```

The plugin's CMake has to find the SDK through `ASHITA4_SDK_PATH` (the upstream `FindAshitaSDK.cmake` does) and either take its output directory from a cache variable or set `artifact-path` to wherever the DLL lands under `build/`.

## Inputs

| Input | Default | |
|---|---|---|
| `targets` | required | CMake targets, space separated |
| `output-dir-variable` | | CMake variable for the output directory, set to `out/` |
| `artifact-path` | `out/*.dll` | files to upload |
| `artifact-name` | repo name | |
| `cmake-args` | | extra configure arguments, space separated |
| `build-type` | `Release` | |
| `submodules` | `false` | `false`, `true` or `recursive` |
| `sdk-repository` | `AshitaXI/Ashita-v4beta` | |
| `sdk-ref` | `v4.3.1.1` | tag or branch |
| `clang-format` | `true` | |
| `clang-format-version` | `22.1.4` | from PyPI |
| `release-tag` | | attach the DLL to this release |

## Releases with release-please

release-please opens a release PR from Conventional Commit subjects; merging it tags the release and the build attaches the DLL. The plugin repo needs `release-please-config.json` and `.release-please-manifest.json`, and Actions allowed to create pull requests.

```yaml
name: build

on:
  push:
    branches: [master]
  pull_request:

permissions:
  contents: read

jobs:
  release-please:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    outputs:
      release_created: ${{ steps.release.outputs.release_created }}
      tag_name: ${{ steps.release.outputs.tag_name }}
    steps:
      - id: release
        uses: googleapis/release-please-action@a02a34c4d625f9be7cb89156071d8567266a2445 # v4.2.0

  build:
    needs: release-please
    if: ${{ !cancelled() && needs.release-please.result != 'failure' }}
    permissions:
      contents: write
    uses: sruon/Ashita4-plugin-ci/.github/workflows/build-plugin.yml@v1
    with:
      targets: myplugin
      output-dir-variable: MYPLUGIN_OUTPUT_DIR
      release-tag: ${{ needs.release-please.outputs.release_created && needs.release-please.outputs.tag_name || '' }}
```
