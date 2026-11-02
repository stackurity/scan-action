# Stackurity scan action

Upload a repository's dependencies to your [Stackurity](https://stackurity.ai) server from GitHub Actions, and fail the job when it has vulnerable findings above a threshold you choose.

The action downloads the `stackurity` release binary, checks its Sigstore signature and checksum, and runs `stackurity scan`. Lockfiles are read as data; nothing in your repository is executed.

## Usage

### Without a stored secret (recommended)

The job proves which repository it runs for with GitHub's OpenID Connect ID token, and Stackurity gives it a collector token that lasts 15 minutes and can report only this repository. There is no secret to store or rotate.

1. In Stackurity, an administrator trusts the repository under **Settings → Integrations → CI pipeline → Trust a repository** (optionally only a branch, an environment or one workflow file). The server needs its public URL set (`STACKURITY_BASE_URL`).
2. Give the job `permissions: id-token: write` and leave out `token`:

```yaml
name: stackurity
on:
  push:
    branches: [main]
permissions:
  contents: read
  id-token: write   # lets the job request an ID token; grants no access to the repository
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: stackurity/scan-action@v1
        with:
          server: https://stackurity.example.com
          fail-on: critical
          only-fixable: true
```

The action requests an ID token for the server's audience (its public URL, read from the server, or the `audience` input), exchanges it at `POST /api/v1/ci/token`, masks both tokens in the log and scans with the repository's URL as its identity. It needs a Stackurity server with keyless CI scans; an older server, a job without `id-token: write`, or a repository no trust accepts fails the job with the reason.

### With a stored token

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

Create the token in Stackurity under **Settings → API tokens** with the `collector` role (or run `stackurity admin create-token --name ci` on the server) and store it as a repository or organisation secret. When `token` is set, the action works as before and requests no ID token.

## Inputs

| Input | Default | Description |
|---|---|---|
| `server` | (required) | Your Stackurity server URL |
| `token` | | A collector token, stored as a secret. Empty: exchange the job's ID token (needs `permissions: id-token: write`) |
| `audience` | the server's public URL | Without a token: the audience the ID token is requested for |
| `ci-trust` | | Without a token: the id of the CI trust to use when several accept this repository |
| `path` | `.` | Directory to scan |
| `git-url` | this repository's URL | Identifies the asset; use the same URL as a server-side repository integration so both land on one asset. A token from the job's ID token can report only this repository's URL |
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
