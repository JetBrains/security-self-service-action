# Security Self-Service Secret Scan

Scans a repository for secrets using TruffleHog with a JetBrains-specific
detector config.

## Config

The TruffleHog detector config is fetched at scan time from an internal
JetBrains endpoint. The job fails if that fetch fails, so a broken or
unreachable config server blocks the scan rather than silently running
with stale/no detectors.

## Usage

`runs-on: sre-eqx-kata` and `fetch-depth: 0` are both required — the
runner needs network access to the internal config endpoint (and, during
verification, the detector validator endpoints), and TruffleHog needs
full git history to resolve `since-commit` (a shallow checkout will fail
with `unable to resolve commit: object not found`).

## Inputs

| Input | Default | Description |
|---|---|---|
| `path` | `.` | Path to scan, relative to the checked-out repository. Passed to TruffleHog as its `git` source (`file://<path>`). |
| `since-commit` | `` (falls back to `github.event.before`) | Commit SHA to scan from, exclusive. On a `push` event this defaults to the commit the branch pointed to before the push, so only the newly pushed commits are scanned. On the first push to a branch, or on non-`push` triggers, `github.event.before` isn't usable and the action falls back to scanning full history. Override to scan an explicit range. |
| `branch` | `` (falls back to `github.ref_name`) | Branch to scan. Defaults to the current ref; override for non-`push` triggers or to scan a different branch than the one checked out. |
| `only-verified` | `'false'` | If `'true'`, only report/fail on secrets verified as currently live (confirmed via API check), suppressing unverified and unknown matches. `'false'` (default) reports verified, unverified, and unknown results — noisier, but nothing is silently dropped. |
| `extra-args` | `''` | Additional space-separated arguments appended after the action's default TruffleHog flags (e.g. `--exclude-paths=path/to/file`). Use this for anything the action doesn't expose as a first-class input. |

### Verified-only scan

Fails only on secrets confirmed live. Good for a blocking gate you don't
want to be noisy.

```yaml
on: push

jobs:
  test-verified-only:
    runs-on: sre-eqx-kata
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: JetBrains/security-self-service-action@v4
        with:
          only-verified: 'true'
```

### Full scan (verified, unverified, and unknown)

Reports everything TruffleHog's detectors match, regardless of
verification outcome. Good for a non-blocking or exploratory scan where
you'd rather review noise than miss something.

```yaml
on: push

jobs:
  test-all-results:
    runs-on: sre-eqx-kata
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: JetBrains/security-self-service-action@v4
        with:
          only-verified: 'false'
```

`only-verified: 'true'` restricts results to secrets the verifier
confirmed are currently live. `'false'` (default) also includes matches
that failed verification or that the verifier couldn't reach — more
noise, but nothing gets silently dropped.

### Scoping the scan or passing extra TruffleHog flags

```yaml
on: push

jobs:
  scan-subdir-excluding-fixtures:
    runs-on: sre-eqx-kata
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: JetBrains/security-self-service-action@v4
        with:
          path: './services/api'
          extra-args: '--exclude-paths=.github/trufflehog-exclude.txt'
```

`path` scopes which directory TruffleHog treats as its scan target.
`extra-args` is a passthrough for any TruffleHog CLI flag not already
covered by a first-class input — it's appended after the action's own
flags, so it can add restrictions (`--exclude-paths`) but can't override
flags the action already sets (e.g. `--results`, `--fail`).
