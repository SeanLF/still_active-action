# still_active-action

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-still__active-2ea44f?logo=github)](https://github.com/marketplace/actions/still_active)
[![Self-test](https://github.com/SeanLF/still_active-action/actions/workflows/test.yml/badge.svg)](https://github.com/SeanLF/still_active-action/actions/workflows/test.yml)

GitHub Action wrapping the [`still_active`](https://github.com/SeanLF/still_active) gem — audit your Gemfile for maintenance signals (last commit dates, archived repos, OpenSSF Scorecard, vulnerabilities, libyear, Ruby EOL) directly from CI, with optional SARIF output to GitHub Code Scanning.

## Quick start

```yaml
# .github/workflows/deps.yml
name: Dependency audit
on: [pull_request, push, workflow_dispatch]

permissions:
  contents: read
  security-events: write  # only if uploading SARIF

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with: { ruby-version: '3.4' }
      - uses: SeanLF/still_active-action@v0
        with:
          github-token: ${{ github.token }}
          fail-if-warning: 'true'
          fail-if-vulnerable: 'high'
          sarif: still_active.sarif.json
      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: still_active.sarif.json
```

Findings will appear in the **Security → Code scanning** tab and as inline annotations on `Gemfile.lock` in pull requests.

## Inputs

| Name | Description | Default |
| --- | --- | --- |
| `gemfile-path` | Path to the Gemfile to audit | `./Gemfile` |
| `gems` | Comma-separated gem list (alternative to `gemfile-path`) | – |
| `ignore` | Gems to exclude from pass/fail (comma list or YAML block) | – |
| `fail-if-warning` | Exit 1 on stale/critical/archived activity (`true`/`false`) | `false` |
| `fail-if-vulnerable` | Exit 1 on vulns; `true`/`false` or `low`/`medium`/`high`/`critical` | `false` |
| `fail-if-outdated` | Exit 1 if any gem exceeds N libyears behind latest | – |
| `output-format` | `terminal`, `markdown`, or `json` | `json` |
| `sarif` | Path to write SARIF 2.1.0 output to (e.g. `still_active.sarif.json`) | – |
| `baseline` | Path to baseline JSON snapshot; emits markdown delta, exits 1 on regressions | – |
| `cyclonedx` | Path to write a CycloneDX SBOM to, or `-` for stdout (still_active ≥ 1.5.0) | – |
| `cyclonedx-version` | CycloneDX spec version: `1.6` (default) or `1.7`; only with `cyclonedx` | – |
| `alternatives` | Suggest maintained alternatives (Ruby Toolbox leads) for archived/critical gems (`true`/`false`, still_active ≥ 1.6.0) | `false` |
| `bundler-audit` | Install bundler-audit + fetch ruby-advisory-db for dual-source vulns (`true`/`false`, still_active ≥ 1.5.0) | `true` |
| `cvss-suite` | Install cvss-suite to score CVSS-4.0-only advisories (`true`/`false`, still_active ≥ 3.0.0; see [Upgrading to still_active 3.0](#upgrading-to-still_active-30)) | `false` |
| `github-token` | GitHub token — pass `${{ github.token }}` explicitly to avoid rate limits | – |
| `gitlab-token` | GitLab token (optional for public repos) | – |
| `version` | still_active gem version (`latest` or pinned) | `latest` |
| `working-directory` | Cwd for the audit | `.` |
| `extra-args` | Raw flag passthrough (shell-split — don't pass user input) | – |

## Outputs

| Name | Description |
| --- | --- |
| `exit-code` | 0 = pass, 1 = gating flag tripped or regression |
| `report-path` | Path to the captured report inside the runner |
| `sarif-path` | Path to the SARIF file when `--sarif` was requested |
| `cyclonedx-path` | Path to the CycloneDX SBOM when `cyclonedx` wrote to a file (empty for stdout) |

## Modes

The action runs in one of four output modes, in this precedence:

1. **Baseline diff** — if `baseline` is set, compares current state against the file and emits a markdown delta. Exits 1 on regressions. Other format inputs are ignored.
2. **SARIF** — if `sarif` is set, writes SARIF 2.1.0 to the given path (`-` for stdout).
3. **CycloneDX** — if `cyclonedx` is set, emits a CycloneDX SBOM to the given path (`-` for stdout). Writing to a file? Add your own upload step (e.g. `actions/upload-artifact` pointing at the `cyclonedx-path` output) — the action does not persist the file. `-` (stdout) is captured into `report-path`.
4. **Format** — otherwise emits `output-format` (terminal/markdown/json). Markdown also lands in the job summary.

`bundler-audit` and `alternatives` are **not** output modes — they're independent toggles (a second vulnerability source, on by default; and Ruby Toolbox replacement leads for archived/critical gems, opt-in) that combine with any of the modes above. Leads render in terminal/markdown/json/sarif; they have no effect under `baseline` or `cyclonedx`.

## Upgrading to still_active 3.0

The action installs `still_active` at `version: latest` by default, so **still_active 3.0 rolls out on your next run with no change to your workflow.** One behaviour change can turn a previously-green run red without you touching anything:

**`fail-if-vulnerable` now fails closed on unscored advisories.** Before 3.0, an advisory with no CVSS score read as "below threshold" and silently cleared the severity gate — so a real, freshly disclosed CVE could pass (scoring lags disclosure, worst case). From 3.0, an unscored advisory that isn't explicitly suppressed **trips the gate** and the run exits 1, with a per-gem note on stderr explaining why. This is a fix, not a regression: those advisories should have been failing all along.

If a run goes green → red after the upgrade, you have three levers:

- **Enable `cvss-suite: 'true'`.** Some advisories are unscored only because they publish a CVSS **4.0** vector and nothing else; still_active needs the optional `cvss-suite` gem to read a 4.0 vector. Installing it lets those advisories score, and they clear the gate when they land below your threshold.
- **Accept a specific finding** in a committed `.still_active.yml` (a vulnerability suppression must name an explicit advisory id, so a *newly* disclosed CVE on the same gem still fails). This is the right move for an advisory you've reviewed and decided to carry.
- **Pin `version:`** to a `2.x` release to defer the change while you triage. Deferral, not a fix — the fail-open it closes is a real one.

The `cvss-suite` input is opt-in (`false` by default) and mirrors the `bundler-audit` toggle: it adds one `gem install` step, best-effort (a failed install never aborts the audit — the advisory just stays unscored).

## Pinning

Use the floating `@v0` tag for convenience, or pin to an immutable SHA for supply-chain strictness (recommended after [tj-actions/changed-files](https://www.cisa.gov/news-events/alerts/2025/03/18/supply-chain-compromise-third-party-tj-actionschanged-files-cve-2025-30066-and-reviewdogaction)):

```yaml
# Convenience (Dependabot keeps it current):
uses: SeanLF/still_active-action@v0

# Immutable (recommended for production):
uses: SeanLF/still_active-action@<full-sha>  # v0.1.0
```

Dependabot's `github-actions` ecosystem updates SHA-pinned actions and adjusts the trailing version comment.

## Permissions

By default the action only needs `contents: read` (inherited). Add `security-events: write` to the workflow's `permissions:` block if you upload SARIF via `github/codeql-action/upload-sarif`.

## License

MIT. See [`LICENSE`](LICENSE).
