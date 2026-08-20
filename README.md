# Security Self-Service Secret Scan

Scans a repository for secrets using TruffleHog with a JetBrains-specific
detector config.

## Usage

`runs-on: sre-eqx-kata` and `fetch-depth: 0` are both required — the
runner needs network access to the internal secrets verifier
(`secrets-verifier.labs.jb.gg`), and TruffleHog needs full git history to
resolve `since-commit` (a shallow checkout will fail with `unable to
resolve commit: object not found`).

### Verified-only scan

Fails only on secrets confirmed live. Good for a blocking gate you don't
want to be noisy.

```yaml
name: trufflehog_scan
on: push

jobs:
  scan-verified-only:
    runs-on: sre-eqx-kata
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

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
          only-verified: 'true'
```

### Full scan (verified, unverified, and unknown)

Reports everything TruffleHog's detectors match, regardless of
verification outcome. Good for a non-blocking or exploratory scan where
you'd rather review noise than miss something.

```yaml
name: trufflehog_scan
on: push

jobs:
  scan-all-results:
    runs-on: sre-eqx-kata
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

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
          only-verified: 'false'
```

Both can run as separate jobs in the same workflow — see
[`trufflehog_scan.yml`](https://github.com/JetBrains/code-security-automations/blob/main/.github/workflows/trufflehog_scan.yml)
in `code-security-automations` for a live example.

`only-verified: 'true'` restricts results to secrets the verifier
confirmed are currently live. `'false'` (default) also includes matches
that failed verification or that the verifier couldn't reach — more
noise, but nothing gets silently dropped.

## ACTION_REPO_PAT

`security-self-service-action` is a private repo, for some reason even with
all permissions being enabled, trying to use the action direvtly got blocked.
PAT token is a workaround that solves this problem. `ACTION_REPO_PAT` is a fine-grained PAT
scoped to only this repo, with `Contents: Read-only` permission.

### Creating the token

1. GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens** → **Generate new token**.
2. Resource owner: `JetBrains`. Repository access: **Only select repositories** → `security-self-service-action`.
3. Permissions → Repository permissions → **Contents: Read-only**.
4. Generate, and copy the token.
5. In the *consuming* repo: **Settings** → **Secrets and variables** → **Actions** → **New repository secret** → name it `ACTION_REPO_PAT`, paste the token as the value.
