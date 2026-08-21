# still_active-action

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-still__active-2ea44f?logo=github)](https://github.com/marketplace/actions/still_active)
[![Self-test](https://github.com/SeanLF/still_active-action/actions/workflows/test.yml/badge.svg)](https://github.com/SeanLF/still_active-action/actions/workflows/test.yml)

GitHub Action wrapping the [`still_active`](https://github.com/SeanLF/still_active) gem — audit your Gemfile for maintenance signals (last commit dates, archived repos, OpenSSF Scorecard, vulnerabilities, libyear, Ruby EOL) directly from CI, with optional SARIF output to GitHub Code Scanning. With still_active ≥ 3.0.0 it audits a [CycloneDX SBOM](#cross-ecosystem-audits-sbom) too, applying the same lens to npm, PyPI, Cargo, Go, Maven, and NuGet packages.

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

## Upgrading to still_active 3.0

The action installs `still_active` at `version: latest` by default, so **the day still_active 3.0.0 is published as a non-prerelease, every workflow on the default moves from 2.0.0 to 3.0.0 in a single step.** No flag was removed or renamed and the JSON `schema_version` stays `1`, but two behaviour changes can turn a previously-green run red without you touching anything.

**1. `fail-if-vulnerable` now fails closed on unscored advisories.** Before 3.0, an advisory with no CVSS score read as "below threshold" and silently cleared the severity gate, so a real, freshly disclosed CVE could pass (scoring lags disclosure, worst case). From 3.0, an unscored advisory that isn't explicitly suppressed **trips the gate** and the run exits 1, with a per-gem note on stderr explaining why. Bare `fail-if-vulnerable: 'true'` also now fires on advisories a failed deps.dev fetch used to drop silently. This is a fix, not a regression: those advisories should have been failing all along.

If a run goes green → red after the upgrade, you have three levers:

- **Enable `cvss-suite: 'true'`.** Some advisories are unscored only because they publish a CVSS **4.0** vector and nothing else; still_active needs the optional `cvss-suite` gem to read a 4.0 vector. Installing it lets those advisories score, and they clear the gate when they land below your threshold.
- **Accept a specific finding** in a committed `.still_active.yml` (a vulnerability suppression must name an explicit advisory id, so a *newly* disclosed CVE on the same gem still fails). This is the right move for an advisory you've reviewed and decided to carry.
- **Pin `version:`** to a `2.x` release to defer the change while you triage. Deferral, not a fix; the fail-open it closes is a real one.

The `cvss-suite` input is opt-in (`false` by default) and mirrors the `bundler-audit` toggle: it adds one `gem install` step, best-effort (a failed install never aborts the audit, the advisory just stays unscored).

**2. Tokenless runs now resolve GitHub repo signals through ecosyste.ms.** Without a `github-token`, gems that used to hit the 60/hour API wall and degrade to `unknown` now resolve to a real `stale`/`critical`/`archived`, which can trip `fail-if-warning` for the first time. Runs that pass a token are unaffected.

Also worth knowing before the jump: bad CLI input now exits **2** (was 1), so anything branching on the `exit-code` output should treat 2 as "bad invocation"; a `baseline` snapshot should be re-captured after upgrading, since the new signals show up as changes on the first run; and in SARIF, an unscored advisory now maps to `warning` (was `note`) while two new rules (SA008 poison-pill, SA009 runtime ceiling) start appearing, so a code-scanning policy that fails on `warning` can newly fire. Rule IDs SA001-SA007 are unchanged.

Egress-restricted runners need four more hosts allowlisted: `api.osv.dev`, `endoflife.date`, `*.ecosyste.ms`, and `pypi.org` (the Python runtime ceiling reads `requires_python` from it). Each degrades to best-effort rather than crashing if blocked, which for a blocked host means the signal it feeds quietly never fires.

The new `fail-if-poison` and `fail-if-language-ceiling` gates default to off, so they never break an existing pipeline unless you opt in.

Read the gem's [Upgrading to 3.0](https://github.com/SeanLF/still_active/blob/main/CHANGELOG.md#upgrading-to-30) notes for the full list. **For reproducible CI, pin `version:` to a specific release** rather than riding `latest`:

```yaml
- uses: SeanLF/still_active-action@v0
  with:
    version: '3.0.0'  # pin the gem; upgrade deliberately
```

## Inputs

| Name | Description | Default |
| --- | --- | --- |
| `gemfile-path` | Path to the Gemfile to audit | `./Gemfile` |
| `gems` | Comma-separated gem list (alternative to `gemfile-path`) | – |
| `sbom` | Path to a CycloneDX SBOM to audit cross-ecosystem instead of a Gemfile (still_active ≥ 3.0.0) | – |
| `ignore` | Gems to exclude from pass/fail (comma list or YAML block) | – |
| `fail-if-warning` | Exit 1 on stale/critical/archived activity (`true`/`false`) | `false` |
| `fail-if-vulnerable` | Exit 1 on vulns; `true`/`false` or `low`/`medium`/`high`/`critical` | `false` |
| `fail-if-outdated` | Exit 1 if any gem exceeds N libyears behind latest | – |
| `fail-if-poison` | Exit 1 on a poison-pill cap; `true`/`false` or `note`/`warning`/`critical` (still_active ≥ 3.0.0) | `false` |
| `fail-if-language-ceiling` | Exit 1 on an EOL language-runtime ceiling; `true`/`false` or `note`/`warning`/`critical` (still_active ≥ 3.0.0) | `false` |
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
| `ecosystems-email` | Contact email for the ecosyste.ms polite pool (still_active ≥ 3.0.0) | – |
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

In `sbom` mode (below) the same precedence applies minus `baseline`, the one mode that needs the native Ruby audit's snapshot and is rejected up front. `cyclonedx` is supported there and emits an [enriched SBOM](#cross-ecosystem-audits-sbom). Setting more than one of `baseline`, `sarif` and `cyclonedx` emits a `::warning::` naming the one that runs; the action emits a single mode, so run it twice if you need two. The warning annotates, it does not stop the run: the losing mode's `*-path` output is simply never set, so a downstream upload step sees an empty path. `output-format` is the fallback when none of the three is set, so it is not counted, and a mode flag passed through `extra-args` is invisible to the warning (still_active issues its own, which will be correct).

`bundler-audit` and `alternatives` are **not** output modes — they're independent toggles (a second vulnerability source, on by default; and Ruby Toolbox replacement leads for archived/critical gems, opt-in) that combine with any of the modes above. Leads render in terminal/markdown/json/sarif; they have no effect under `baseline` or `cyclonedx`.

## Cross-ecosystem audits (`sbom`)

still_active ≥ 3.0.0 takes a CycloneDX SBOM instead of a Gemfile and applies the same maintenance lens to `npm`, `pypi`, `cargo`, `go`, `maven`, and `nuget` packages: latest release date, archived repository, advisories, OpenSSF Scorecard, lifecycle status. Generate the SBOM with whatever you already use (Syft, Trivy, `npm sbom`), then point the action at it:

```yaml
- uses: anchore/sbom-action@v0
  with:
    format: cyclonedx-json
    output-file: sbom.cdx.json
- uses: SeanLF/still_active-action@v0
  with:
    github-token: ${{ github.token }}
    sbom: sbom.cdx.json
    fail-if-warning: 'true'
    sarif: still_active.sarif.json
```

Constraints, all enforced by the action before it installs anything, so a mis-wired workflow fails in seconds rather than after a full audit:

- `sbom` is **mutually exclusive** with `gemfile-path` and `gems`. Pick one input source.
- `sbom` does **not** support `baseline`: a maintenance-regression diff needs the native audit's gems/ruby snapshot, which a cross-ecosystem SBOM can't supply. Use `sarif`, or `output-format` `terminal`/`markdown`/`json`.
- `sbom` **does** support `cyclonedx`, and the combination is the interesting one: SBOM in, **enriched SBOM out**. The input is re-emitted with still_active's maintenance signals attached as `still_active:`-namespaced component properties (status, archived, deprecated, scorecard, libyear, last commit, ecosystem, direct-vs-transitive) plus the advisories as CycloneDX `vulnerabilities`, so it can be fed to Dependency-Track rather than only read by a human. Every component keeps the PURL it arrived with, so whatever matched your input matches the output. Unlike the constraints above this one is **not** enforced before install, because it depends on the gem: it needs a still_active newer than `3.0.0.rc6`, and on an older one the run reaches still_active and exits 2. Pin `version:` accordingly.
- Findings are named `ecosystem/name` (e.g. `npm/left-pad`), which is also the identity `ignore` and `.still_active.yml` suppressions key on. SARIF anchors them to the SBOM file, since there is no lockfile to annotate.
- The SBOM is treated as untrusted input: only ecosystem, name, and version are read from it, and repositories are discovered from deps.dev rather than any URL the file supplies. Anything unassessable (an unsupported ecosystem, a missing version, a failed lookup) is listed rather than silently dropped.

## Poison-pill and runtime-ceiling gates

Two opt-in gates, both still_active ≥ 3.0.0, both off by default:

- **`fail-if-poison`** fires when a dormant or archived package caps one of its runtime dependencies below that dependency's latest major, a ceiling no upstream release will lift. `true` gates at `warning` and above (the gem's default tier); pass `note`, `warning`, or `critical` to set the threshold yourself. The strongest case, a dead package pinning a dependency below the version that patches a HIGH advisory, leads the report.
- **`fail-if-language-ceiling`** fires when a pinned dependency's `required_ruby_version` (or `requires_python`, or a NuGet target framework) forbids every supported runtime, stranding you on an end-of-life one. `true` gates the `critical` (EOL-forced) tier only; `note` also gates the "latest doesn't support your runtime yet" FYI tier. `warning` behaves as `critical`, since this signal only ever produces the other two tiers.

```yaml
- uses: SeanLF/still_active-action@v0
  with:
    github-token: ${{ github.token }}
    fail-if-poison: 'true'
    fail-if-language-ceiling: 'true'
```

## Pinning

Use the floating `@v0` tag for convenience, or pin to an immutable SHA for supply-chain strictness (recommended after [tj-actions/changed-files](https://www.cisa.gov/news-events/alerts/2025/03/18/supply-chain-compromise-third-party-tj-actionschanged-files-cve-2025-30066-and-reviewdogaction)):

```yaml
# Convenience (Dependabot keeps it current):
uses: SeanLF/still_active-action@v0

# Immutable (recommended for production):
uses: SeanLF/still_active-action@<full-sha>  # v0.1.0
```

Dependabot's `github-actions` ecosystem updates SHA-pinned actions and adjusts the trailing version comment.

Pinning the action is separate from pinning the gem it installs: `version` (default `latest`) controls the latter. See [Upgrading to still_active 3.0](#upgrading-to-still_active-30).

## Permissions

By default the action only needs `contents: read` (inherited). Add `security-events: write` to the workflow's `permissions:` block if you upload SARIF via `github/codeql-action/upload-sarif`.

## License

MIT. See [`LICENSE`](LICENSE).
