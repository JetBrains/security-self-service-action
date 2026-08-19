# Security Self-Service Secret Scan

Scans a repository for secrets using TruffleHog with a JetBrains-specific
detector config.

## Usage

```yaml
name: trufflehog_scan
on: push

jobs:
  scan:
    runs-on: sre-eqx-kata
    steps:
      - uses: actions/checkout@v4

      - uses: actions/checkout@v4
        with:
          repository: JetBrains/security-self-service-action
          ref: main
          path: .security-action
          token: ${{ secrets.ACTION_REPO_PAT }}

      - uses: ./.security-action
        with:
          since-commit: ${{ github.event.before }}
          branch: ${{ github.ref_name }}
```

`runs-on: sre-eqx-kata` is required — the runner needs network access to the
internal secrets verifier.

## ACTION_REPO_PAT

A fine-grained GitHub PAT scoped to only `JetBrains/security-self-service-action`,
with `Contents: Read-only` permission. 
