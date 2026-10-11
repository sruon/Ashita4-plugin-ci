# Ashita4-plugin-ci

Builds Ashita v4 plugins on GitHub Actions with 32-bit MSVC, CMake and Ninja, against a pinned tag of the Ashita SDK.

- `format`: checks every C/C++ file against the repo's `.clang-format`.
- `build`: fetches the SDK from `AshitaXI/Ashita-v4beta`, builds the targets and uploads the DLL.
- `release`: attaches the DLL to the release named in `release-tag`.

## Usage

Add `.github/workflows/build.yml` to the plugin repo:

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

The plugin's CMake must find the SDK through `ASHITA4_SDK_PATH`, as the upstream `FindAshitaSDK.cmake` does. If it has no output directory variable, point `artifact-path` at the DLL under `build/`.

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

release-please opens a release PR from `feat:` and `fix:` commits. Merge it to tag the release, and the build attaches the DLL. The plugin repo needs `release-please-config.json`, `.release-please-manifest.json`, and Actions allowed to create pull requests.

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
        uses: googleapis/release-please-action@45996ed1f6d02564a971a2fa1b5860e934307cf7 # v5.0.0

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
