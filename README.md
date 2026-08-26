# security-self-service-action

This repository hosts self-service security GitHub Actions for JetBrains teams. It will grow to include more than one action over time — each one lives in its own subfolder with its own README section below.

## Trufflehog action

The action lives in [`trufflehog/`](./trufflehog/action.yaml) and is referenced as `JetBrains/security-self-service-action/trufflehog@main`. It scans a repository for secrets using TruffleHog with a JetBrains-specific detector config. It scans all of the commits on the current feature branch; on the default branch it performs a full scan instead.

### Usage

`runs-on: sre-eqx-kata` and `fetch-depth: 0` are both required — the runner needs network access to the internal config endpoint (and, during verification, the detector validator endpoints), and the action needs full git history (including the default branch) to resolve `since-commit` via merge-base or an explicit override; a shallow checkout will fail with `unable to resolve commit: object not found` or silently fall back to a full-history scan.

### Default trufflehog action (verified-only)

Fails only on secrets confirmed live. Good for a blocking gate you don't want to be noisy.

```yaml
on: push

jobs:
  test-verified-only:
    runs-on: sre-eqx-kata
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: JetBrains/security-self-service-action/trufflehog@main
```

### Trufflehog full scan (verified, unverified, and unknown)

Reports everything TruffleHog's detectors match, regardless of verification outcome. Good for a non-blocking or exploratory scan where you'd rather review noise than miss something.

```yaml
on: push

jobs:
  test-all-results:
    runs-on: sre-eqx-kata
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: JetBrains/security-self-service-action/trufflehog@main
        with:
          only-verified: 'false'
```

`only-verified: 'true'` (default) restricts results to secrets the verifier confirmed are currently live. `'false'` also includes matches that failed verification or that the verifier couldn't reach — more noise, but nothing gets silently dropped.

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
      - uses: JetBrains/security-self-service-action/trufflehog@main
        with:
          path: './services/api'
          extra-args: '--exclude-paths=.github/trufflehog-exclude.txt'
```

`path` scopes which directory TruffleHog treats as its scan target. `extra-args` is a passthrough for any TruffleHog CLI flag not already covered by a first-class input — it's appended after the action's own flags, so it can add restrictions (`--exclude-paths`) but can't override flags the action already sets (e.g. `--results`, `--fail`). See [TruffleHog's own documentation](https://github.com/trufflesecurity/trufflehog#usage) for the full list of available flags.

### Inputs

| Input | Default | Description |
|---|---|---|
| `path` | `.` | Path to scan, relative to the checked-out repository. Passed to TruffleHog as its `git` source (`file://<path>`). |
| `since-commit` | `` (falls back to merge-base with the default branch) | Commit SHA to scan from, exclusive. By default (no explicit value), the action scans every commit on the current branch since it diverged from the repo's default branch — computed via `git merge-base <default-branch> HEAD` — regardless of how many pushes came before. On the default branch itself, or if merge-base can't be resolved (e.g. a shallow checkout missing the required history), it falls back to scanning full history instead. Override to scan an explicit range. |
| `branch` | `` (falls back to `github.ref_name`) | Branch to scan. Defaults to the current ref; override for non-`push` triggers or to scan a different branch than the one checked out. |
| `only-verified` | `'true'` | If `'true'` (default), only report/fail on secrets verified as currently live (confirmed via API check), suppressing unverified and unknown matches. Set `'false'` to also report unverified and unknown results — noisier, but nothing is silently dropped. |
| `extra-args` | `''` | Additional space-separated arguments appended after the action's default TruffleHog flags (e.g. `--exclude-paths=path/to/file`). Use this for anything the action doesn't expose as a first-class input. |
