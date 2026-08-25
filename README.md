# Security Self-Service Secret Scan

Scans a repository for secrets using TruffleHog with a JetBrains-specific
detector config.

## Config

The TruffleHog detector config is fetched at scan time. 
The job fails if that fetch fails, so a broken or
unreachable config server blocks the scan rather than silently running
with stale/no detectors.

## Usage

`runs-on: sre-eqx-kata` and `fetch-depth: 0` are both required — the
runner needs network access to the internal secrets verifier, and TruffleHog needs full git history to
resolve `since-commit` (a shallow checkout will fail with `unable to
resolve commit: object not found`).

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
