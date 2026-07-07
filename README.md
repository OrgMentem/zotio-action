<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/OrgMentem/zotio/main/docs/assets/logo-wordmark-dark.svg">
    <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/OrgMentem/zotio/main/docs/assets/logo-wordmark.svg">
    <img alt="zotio" src="https://raw.githubusercontent.com/OrgMentem/zotio/main/docs/assets/logo-wordmark.svg" width="180">
  </picture>
</p>

# Bibliography health for Zotero

CI for your bibliography, powered by [zotio](https://github.com/OrgMentem/zotio). This composite action installs `zotio`, syncs your Zotero library from the Zotero Web API, and runs `zotio library health` as a quality gate for a paper, thesis, dissertation, or review repository.

```yaml
- uses: OrgMentem/zotio-action@v1
  with:
    api-key: ${{ secrets.ZOTERO_API_KEY }}
    for: citation
    fail-on: high
```

## What you get

- **A real quality gate, not a report.** Deterministic exit codes (`0` pass · `11` findings at your threshold · `12` stale data · `9` setup) — the job fails when your bibliography isn't fit to cite.
- **Retraction checking.** `check-retractions: "true"` probes Crossref's Retraction Watch data and fails the build the day a cited paper is retracted.
- **A badge that never lies.** `badge-path` writes shields.io endpoint JSON *before* the gate exits — a failing gate still publishes a truthful red badge, never a stale green one.
- **Readable failures.** Every run writes a verdict table to the job's step summary; `exit-code`/`message`/`color` outputs let downstream steps post PR comments or publish the badge.
- **Presets that match how researchers work.** `--for citation` (submission-ready), `systematic-review` (screening integrity), `quick`, `all`; `extra-args` passes anything else (`--require-fresh 24h`, `--limit 20`).
- **No Docker, no toolchain.** Composite action; installs a signed release binary on Linux, macOS, or Windows runners in seconds. Key verified up front and masked in logs — a read-only Zotero key is all it needs.

## Run it on a schedule — your library drifts outside git

A normal CI gate runs when the *repo* changes. But your Zotero library changes when you import a paper on a Tuesday night — no push, no PR, no trigger. Add a cron so drift is caught within a day, not at submission time:

```yaml
on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: "0 6 * * *"   # daily: catch library drift and fresh retractions
```

This also re-checks retractions daily — papers get retracted on their own schedule, not yours.

## Fail the build on a retracted citation

<p align="center">
  <img src="https://raw.githubusercontent.com/OrgMentem/zotio/main/docs/assets/demos/retract-check.gif" alt="zotio catching a genuinely retracted paper via Crossref's Retraction Watch data" width="700">
</p>

With `check-retractions: "true"`, the health gate probes every DOI-bearing item against Crossref's Retraction Watch metadata (retractions, expressions of concern, corrections) as a critical finding — the exact failure mode that ends careers when a reviewer finds it first.

## CI authentication

GitHub-hosted runners do **not** have Zotero Desktop, the local Zotero API at `localhost:23119`, or an already-synced `zotio` SQLite store — so for CI, `api-key` is effectively required. The action fails fast with exit `9` when it is missing. When present, the action:

1. Masks and exports the key as `ZOTERO_API_KEY` for both `zotio sync` and `zotio library health`.
2. Calls `https://api.zotero.org/keys/current` to verify the key and resolve your numeric Zotero user ID.
3. Exports `ZOTERO_BASE_URL=https://api.zotero.org/users/<userID>` so `zotio sync` reads from the Web API.

Create a key at <https://www.zotero.org/settings/keys> — **read access is sufficient**; the action never writes to your library. For group libraries, set `ZOTERO_GROUP` in the workflow environment to the numeric group ID (the key must have read access to that group). Fork PRs cannot read your secret: GitHub withholds secrets from fork-triggered runs by default.

## Complete thesis workflow

```yaml
name: Bibliography health

on:
  pull_request:
  push:
    branches: [main]
  schedule:
    - cron: "0 6 * * *"   # your library drifts outside git — check daily

permissions:
  contents: read

jobs:
  bibliography:
    name: Gate Zotero bibliography
    runs-on: ubuntu-latest
    env:
      # Optional: uncomment for a group library. The key must be allowed to read it.
      # ZOTERO_GROUP: "1234567"
    steps:
      - name: Check out thesis
        uses: actions/checkout@v4

      - name: Check Zotero library health
        id: health
        uses: OrgMentem/zotio-action@v1
        with:
          api-key: ${{ secrets.ZOTERO_API_KEY }}
          for: citation
          fail-on: high
          check-retractions: "true"
          badge-path: bibliography-badge.json
          extra-args: --require-fresh 24h

      - name: Upload badge JSON artifact
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: bibliography-badge
          path: bibliography-badge.json

      - name: React to the verdict (optional)
        if: always()
        run: echo "gate=${{ steps.health.outputs.exit-code }} verdict='${{ steps.health.outputs.message }}' color=${{ steps.health.outputs.color }}"
```

The checkout step is not required by `zotio` itself, but it keeps this workflow ready for thesis repositories where later jobs build the manuscript.

## Shields endpoint badge

Set `badge-path` to make the health run write a [shields.io endpoint](https://shields.io/badges/endpoint-badge) JSON file before the quality gate exits. Publish that JSON somewhere public, such as GitHub Pages, a gist, or another static artifact host, then add an endpoint badge to your README or manuscript dashboard.

Example endpoint URL after publishing `bibliography-badge.json` to GitHub Pages:

```markdown
![bibliography](https://img.shields.io/endpoint?url=https://<owner>.github.io/<repo>/bibliography-badge.json)
```

The badge is written **before** the gate exits, so a failing run still publishes a truthful red badge — never a stale green one. (`zotio library health --badge` already emits JSON, so the action never combines badge mode with `--json`.)

## Inputs

| Input | Default | Description |
|---|---:|---|
| `version` | `latest` | zotio release tag to install. `latest` resolves `https://api.github.com/repos/OrgMentem/zotio/releases/latest`. Non-latest values must match a release tag, for example `v1.2.3`. |
| `for` | `citation` | Health preset passed to `zotio library health --for`: `quick`, `citation`, `systematic-review`, or `all`. |
| `fail-on` | `high` | Gate threshold passed to `zotio library health --fail-on`: `critical`, `high`, or `any`. |
| `badge-path` | empty | Path where a shields.io endpoint badge JSON should be written. Empty disables badge output. |
| `check-retractions` | `false` | When `true`, passes `--check-retractions` to the health command. |
| `extra-args` | empty | Extra shell-style arguments appended to `zotio library health`, such as `--require-fresh 24h --limit 20`. |
| `api-key` | empty | Zotero Web API key. Required on CI because there is no Zotero Desktop/local API or synced store on the runner. Exported as `ZOTERO_API_KEY` for sync and health. |

## Outputs

| Output | Description |
|---|---|
| `exit-code` | Health gate exit code (`0`, `9`, `11`, `12`). |
| `message` | Badge message, e.g. `healthy` or `2 critical, 5 high` (empty unless `badge-path` is set). |
| `color` | Badge color keyword, e.g. `brightgreen`, `yellow`, `red` (empty unless `badge-path` is set). |

Every run also writes a verdict table to the job's **step summary**, so failures are readable at a glance in the Actions UI.

## Exit codes

The action preserves the `zotio library health` gate exit code.

| Code | Meaning |
|---:|---|
| `0` | Sync and health completed, and the configured quality gate passed. |
| `9` | Setup/precondition failure, including missing `api-key` in CI or a health check whose required setup is unavailable. |
| `11` | Quality gate failed: findings met or exceeded `fail-on`. This fails the job by design. |
| `12` | Freshness gate failed, for example `extra-args: --require-fresh 24h` found the local mirror too stale after sync. |

Setup outside the health gate can also fail before `zotio` runs, for example when the GitHub releases API or Zotero Web API cannot be reached or rejects the key.

## Release artifact assumptions

This action downloads zotio from `https://github.com/OrgMentem/zotio/releases` using the current GoReleaser archive naming scheme:

- Project name: `zotio`
- Binary used by the action: `zotio`
- Archive template: `zotio_<version>_<os>_<arch>`
- Unix archives: `.tar.gz`
- Windows archives: `.zip`
- Supported OS/arch pairs: `darwin_amd64`, `darwin_arm64`, `linux_amd64`, `linux_arm64`, `windows_amd64`, `windows_arm64`

The release archives also include `zotio-mcp`, but the action only invokes the `zotio` binary.

## License

MIT. See [LICENSE](LICENSE).

---

Zotero is a registered trademark of the [Corporation for Digital Scholarship](https://digitalscholar.org/). zotio and this action are independent projects and are not affiliated with or endorsed by Zotero or the Corporation for Digital Scholarship.
