# Setup mcpdev GitHub Action

CLI documentation: [phillarmonic.github.io/mcpdev](https://phillarmonic.github.io/mcpdev/)

A GitHub Action to install and setup [the mcpdev cli](https://github.com/phillarmonic/mcpdev) - a direct CLI for MCP servers over stdio and Streamable HTTP.

mcpdev inspects, invokes, snapshots, diffs, and replays MCP servers. It was built as a local debugging tool, but the same interface contract checks are useful in CI: a workflow can inspect a server, compare its catalog against a committed baseline, and fail the build when a tool disappears. This action puts the release binary on `PATH` so those checks run against the exact build the release published.


## Features

- **Cross-platform support**: Linux, macOS, and Windows
- **Multi-architecture**: AMD64 and ARM64
- **Checksum verification**: every archive is matched against the release `SHA256SUMS`
- **Version flexibility**: install latest or a specific tag
- **Major version pinning**: resolve `v0` or `v0.1` to the newest matching release
- **Caching support**: cache the installed binary for faster builds
- **GitHub token support**: avoid API rate limiting

## Usage

### Basic Usage

```yaml
- name: Setup mcpdev
  uses: phillarmonic/setup-mcpdev@v1
```

### Specify Version

```yaml
- name: Setup mcpdev
  uses: phillarmonic/setup-mcpdev@v1
  with:
    version: 'v0.1.0'
```

### Pin to a Version Line

```yaml
- name: Setup mcpdev
  uses: phillarmonic/setup-mcpdev@v1
  with:
    version: 'v0'
```

This resolves to the newest stable `v0.x.y` release. Prefixes with two components also work: `v0.1` resolves to the newest `v0.1.y` release.

### With Caching Disabled

```yaml
- name: Setup mcpdev
  uses: phillarmonic/setup-mcpdev@v1
  with:
    version: 'latest'
    cache: 'false'
```

### With Custom GitHub Token

```yaml
- name: Setup mcpdev
  uses: phillarmonic/setup-mcpdev@v1
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
```

## Inputs

| Input     | Description                                                     | Required | Default               |
| --------- | --------------------------------------------------------------- | -------- | --------------------- |
| `version` | Version of mcpdev to install (e.g., "v0", "v0.1.0", "latest")   | No       | `latest`              |
| `token`   | GitHub token for API requests (to avoid rate limiting)          | No       | `${{ github.token }}` |
| `cache`   | Enable caching of downloaded binaries                           | No       | `true`                |

## Outputs

| Output      | Description                                |
| ----------- | ------------------------------------------ |
| `version`   | The version of mcpdev that was installed   |
| `path`      | Path to the installed mcpdev binary        |
| `cache-hit` | Whether the binary was restored from cache |

## Example Workflows

### CLI Checks in CI

```yaml
name: MCP contract

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  contract:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v7

      - name: Setup mcpdev
        uses: phillarmonic/setup-mcpdev@v1
        id: mcpdev

      - name: Show mcpdev info
        run: |
          echo "Installed version: ${{ steps.mcpdev.outputs.version }}"
          echo "Binary path: ${{ steps.mcpdev.outputs.path }}"
          echo "Cache hit: ${{ steps.mcpdev.outputs.cache-hit }}"
          mcpdev --format json version

      - name: Inspect the server under test
        run: mcpdev inspect tools --command ./bin/my-mcp-server
```

### Multi-platform Testing

```yaml
name: Multi-platform MCP checks

on: [push, pull_request]

jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]

    runs-on: ${{ matrix.os }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v7

      - name: Setup mcpdev
        uses: phillarmonic/setup-mcpdev@v1

      - name: Inspect the server under test
        run: mcpdev inspect tools --command ./bin/my-mcp-server
```

### Snapshot Diff Gate

Commit a snapshot of the server catalog and fail the build when the interface changes beyond the policy you allow.

```yaml
name: Interface baseline

on: [pull_request]

jobs:
  baseline:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v7

      - name: Setup mcpdev
        uses: phillarmonic/setup-mcpdev@v1

      - name: Capture the current interface
        run: mcpdev snapshot --command ./bin/my-mcp-server --out contract.json

      - name: Compare against the committed baseline
        run: mcpdev diff --from .mcpdev/baseline.json --to contract.json --fail-on breaking
```

`--fail-on` accepts `metadata`, `additive`, or `breaking`. A triggering change exits with class `40`.

### Version Matrix Testing

```yaml
name: Version matrix

on: [workflow_dispatch]

jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        mcpdev-version: ['v0.1.0', 'v0', 'latest']
        os: [ubuntu-latest, macos-latest]

    runs-on: ${{ matrix.os }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v7

      - name: Setup mcpdev ${{ matrix.mcpdev-version }}
        uses: phillarmonic/setup-mcpdev@v1
        with:
          version: ${{ matrix.mcpdev-version }}

      - name: Test with mcpdev
        run: mcpdev version
```

## Supported Platforms

| OS      | Architecture | Status    |
| ------- | ------------ | --------- |
| Linux   | AMD64        | Supported |
| Linux   | ARM64        | Supported |
| macOS   | AMD64        | Supported |
| macOS   | ARM64        | Supported |
| Windows | AMD64        | Supported |
| Windows | ARM64        | Supported |

## What Gets Installed

mcpdev publishes one archive per target:

- `mcpdev_<version>_<os>_<arch>.tar.gz` for Linux and macOS
- `mcpdev_<version>_<os>_<arch>.zip` for Windows

Each archive holds a single binary (`mcpdev` or `mcpdev.exe`). The action downloads the matching archive and `SHA256SUMS`, checks the digest, unpacks the binary into `${{ runner.temp }}/mcpdev-bin`, and adds that directory to `PATH`.

The installed binary is then checked with `mcpdev --format json version`, which must report the version that was requested. A release tag and its stamped archive cannot drift without failing the step.

## Caching

The action caches the installed binary under a key built from the mcpdev version and the runner platform. Repeating a job restores the binary instead of downloading it again.

To disable caching, set the `cache` input to `'false'`.

## Rate Limiting

The action uses the GitHub API to fetch release information. To avoid rate limiting:

1. The action uses `${{ github.token }}` by default
2. For higher rate limits, provide a custom token via the `token` input

## Troubleshooting

### Binary Not Found

If you see "mcpdev: command not found":

1. Check that the action completed successfully
2. Verify the platform is supported
3. Check the GitHub Actions logs for download or checksum errors

### Checksum Mismatch

If the action reports a checksum mismatch:

1. Confirm the release is intact on the [releases page](https://github.com/phillarmonic/mcpdev/releases)
2. Report the release instead of working around the check, since the archive and `SHA256SUMS` no longer agree

### Rate Limiting

If you encounter GitHub API rate limiting:

1. Provide a GitHub token via the `token` input
2. Use a specific version instead of "latest" to reduce API calls

### Version Not Found

If a specific version is not found:

1. Check available releases: https://github.com/phillarmonic/mcpdev/releases
2. Ensure the version format is correct (e.g., "v0.1.0")
3. Use "latest" to get the most recent release

## Versioning

The action is versioned separately from mcpdev. `v1` tracks the newest 1.x action release and is what the examples pin. For a fully reproducible workflow, pin the exact tag (`v1.0.0`).

The `version` input selects the mcpdev release independently from the action tag, so new action code can install an older mcpdev build when you need that.

## Contributing

Issues and pull requests are welcome! Please see the [main repository](https://github.com/phillarmonic/mcpdev) for contribution guidelines.

## License

This action is distributed under the same license as mcpdev. See the [LICENSE](LICENSE) file for details.
