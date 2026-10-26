# Stackurity scan action

Upload a repository's dependencies to your [Stackurity](https://stackurity.dev) server from GitHub Actions, and fail the job when it has vulnerable findings above a threshold you choose.

The action downloads the `stackurity` release binary, checks its Sigstore signature and checksum, and runs `stackurity scan`. Lockfiles are read as data; nothing in your repository is executed.

## Usage

```yaml
name: stackurity
on: [push, pull_request]
permissions:
  contents: read
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: stackurity/scan-action@v1
        with:
          server: https://stackurity.example.com
          token: ${{ secrets.STACKURITY_TOKEN }}
          environment: production
          fail-on: critical
          only-fixable: true
```

Create the token in Stackurity under **Settings → API tokens** with the `collector` role (or run `stackurity admin create-token --name ci` on the server) and store it as a repository or organisation secret.

## Inputs

| Input | Default | Description |
|---|---|---|
| `server` | (required) | Your Stackurity server URL |
| `token` | (required) | A collector token |
| `path` | `.` | Directory to scan |
| `git-url` | this repository's URL | Identifies the asset; use the same URL as a server-side repository integration so both land on one asset |
| `environment` | | Environment label, for example `production` |
| `fail-on` | | Fail on an open finding of this severity or higher: `low`, `medium`, `high`, `critical` |
| `fail-on-priority` | | Fail on this priority or higher: `P1` to `P4` (accounts for exposure, environment and exploitation) |
| `fail-on-kev` | `false` | Fail on any known exploited vulnerability |
| `only-fixable` | `false` | With a fail-on input, ignore findings that have no fixed version yet |
| `wait` | `5m` | How long to wait for the server to evaluate the scan |
| `version` | `latest` | Stackurity release to use, for example `v0.3.1` |

Without a `fail-on` input the scan is uploaded and the job passes. Findings that are suppressed or risk-accepted on the server do not fail the gate, so exceptions are recorded in one place. Components Stackurity cannot evaluate are listed as a warning and cannot fail the gate.

The job's exit status is 3 when findings meet the gate and 1 when the scan itself failed.

## Pinning

`@v1` follows compatible updates of this action. For the strictest supply-chain posture, pin the action to a commit SHA and set `version` to a specific Stackurity release.

## Tests

The action is tested daily against a real Stackurity server and on Linux, macOS and Windows runners in [stackurity/scan-action-tests](https://github.com/stackurity/scan-action-tests) (GitHub Marketplace actions may not contain workflow files).

## Licence

Apache-2.0. See [LICENSE](LICENSE).
