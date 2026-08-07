# HANDOFF — GitHub Profile README: bug fixes + private metrics + local card generation

## Goal
Fix and improve the GitHub profile README for **BharatSingla12/BharatSingla12** (AI Engineer profile). User chose **"Fix bugs only"** scope, then asked for the lowlighter/metrics cards to **include private repo data**, asked to verify the `METRICS_TOKEN` PAT, and asked to **generate the cards manually/local** because GitHub Actions runs are stuck (runner shortage).

## Constraints & Preferences
- **Fix bugs only** — no content redesign, no new sections (featured projects etc. explicitly deferred).
- **Keep `secrets.METRICS_TOKEN`** — do NOT switch to `GITHUB_TOKEN`; the PAT is needed so private-repo metrics are visible.
- Private data must show in the generated cards.
- Cards must be embedded in the README (already done).

## Progress

### Done
- [x] Reviewed README + `.github/workflows/metrics.yml`; diagnosed 3 bugs: (1) generated SVGs never embedded in README, (2) workflow had no `output_action: commit` so SVGs were discarded (repo copies still placeholder "Loading…"), (3) unnecessary `environment: production` + reliance on `METRICS_TOKEN`.
- [x] Fixed `.github/workflows/metrics.yml`: added `output_action: commit` to **both** steps (token lines untouched).
- [x] Added `repositories_affiliations: owner, collaborator, organization_member` to **both** steps (lowlighter/metrics has no `visibility: private` toggle — private data flows in when the token can see it; this widens which repos are fetched).
- [x] Embedded both cards in `README.md` under "📊 GitHub Analytics": `github-metrics.svg` (Stats & Languages) and `github-metrics-development.svg` (Development Activity), raw.githubusercontent URLs, full-width centered.
- [x] Verified via `gh` (authenticated as BharatSingla12, token scopes: `gist, read:org, repo, user, workflow`): repo secret `METRICS_TOKEN` **exists** (created/updated 2026-08-06T18:41:45Z). Secret *value* is not retrievable via API.
- [x] Diagnosed all 4 workflow runs (31124845490, 31125199963, 31125661132, 31126534214): every one failed/cancelled with **"The job was not acquired by Runner of type hosted even after multiple attempts"** — GitHub Actions runner shortage/outage, not a config/token error. Latest test run (31126534214, triggered via `gh workflow run`) also failed the same way.
- [x] Set up **local metrics generation** (workaround while Actions is down):
  - Cloned `https://github.com/lowlighter/metrics` (v3.34, commit 366f8b9) to `/tmp/metrics`.
  - `npm install --omit=dev` on Node v24.16.0; approved install scripts (`npm install-scripts approve sharp puppeteer canvas libxmljs2`); rebuilt `sharp` OK (libvips downloaded); `libxmljs2` **cannot compile** on Node 24 (NAN/V8 incompatibility, unmaintained pkg).
  - Workarounds: `INPUT_VERIFY=no` (metrics has a real bug — `source/app/metrics/index.mjs:220` runs `libxmljs.parseXml()` even when the import failed, i.e. `if (!libxmljs)` should be `if (libxmljs)`); `PUPPETEER_EXECUTABLE_PATH=/usr/bin/google-chrome` (system Chrome exists; puppeteer needs it for the resize step).
  - Patched action source in the local clone only: `sed -i 's|paths.join("/renders"|paths.join("/tmp/renders"|g' source/app/action/index.mjs` (Docker hardcodes `/renders`).
  - Card 1 (`github-metrics.svg`) now **renders successfully** (data computed, SVG emitted, exit 0) — see In Progress for the remaining blockers.

### In Progress
- [ ] **Card 1 output file not written** — run ends with `Actions to perform: (none)` (the `if (dryrun)` branch, `source/app/action/index.mjs:469`) so no file lands in `/tmp/renders`. `dryrun` unexpectedly truthy → investigate how `dryrun` gets set (metadata default is `no`; possibly enabled by `conf.settings.extras = {default: true}` at `index.mjs:100`, or by GITHUB_ACTIONS/local-env detection). Fix: set `INPUT_DRYRUN=no` explicitly, or patch the action.
- [ ] **18 harmless plugin errors** — unconfigured optional plugins (16personalities, chess, crypto, nightscout, screenshot, splatoon, stock, contributors, licenses, music, pagespeed…) run and error; they only show as inline errors in the image. Cause: `conf.settings.extras = {default: true}` (action line 100). These plugins aren't in the config — need to verify why they're enabled (they should be off without `INPUT_PLUGIN_*`); if unavoidable locally, they don't block the SVG but may add error text to the card.
- [ ] Generate card 2 (`github-metrics-development.svg`) with the dev-activity inputs (habits/followup/lines) once card 1 saves correctly.
- [ ] Copy both SVGs into the profile repo, commit, push (README/workflow changes are currently uncommitted).

### Blocked
- **GitHub Actions runner shortage** — no hosted runner available; all workflow runs fail after ~15–17 min. Local generation is the workaround. Re-try `gh workflow run Metrics` later; the fixed workflow should work once runners return.
- `libxmljs2` won't build on Node 24 (only node v22.21.1 and v24.16.0 installed via nvm; metrics Docker image uses node:20). Worked around via `INPUT_VERIFY=no`.

## Key Decisions
- **Keep `METRICS_TOKEN` PAT (not `GITHUB_TOKEN`)**: user explicitly wants private metrics; `GITHUB_TOKEN` can't see private repos. Both steps still use `${{ secrets.METRICS_TOKEN }}`.
- **`repositories_affiliations: owner, collaborator, organization_member`**: metrics has no private-visibility flag; this maximizes repos fetched. **Caveat:** token must have `repo` scope (classic) or fine-grained access to the private repos — cannot verify the secret's value; user should confirm in GitHub settings.
- **Local generation with the `gh` CLI token**: it has `repo` scope, so cards generated locally will include private data; safer than asking for sudo/Docker.
- **Patch local clone instead of upstream**: `/renders`→`/tmp/renders` is a local-only change; do not commit it.
- **Defer content improvements** (featured projects, stats card, smaller badges) — user chose bug-fix-only scope; offered later.

## Next Steps
1. Fix dryrun so card 1 file is written: try `export INPUT_DRYRUN="no"` in `/tmp/run-card1.sh` and re-run; if still not saved, patch the `if (dryrun)` branch or check how `dryrun` is derived in `source/app/action/index.mjs` (destructuring ~line 130).
2. Confirm `/tmp/renders/github-metrics.svg` exists, is a real card (not "Loading…"), and contains private-repo data (check commit calendar + language percentages are non-trivial).
3. Create `/tmp/run-card2.sh` with dev-activity inputs (`base=""`, `filename=github-metrics-development.svg`, `plugin_habits*`, `plugin_followup*`, `plugin_lines*`) and generate card 2.
4. Copy both SVGs into `/mnt/localdisk/code_base/poc/BharatSingla12/` (replacing the placeholder SVGs), then `git add -A && git commit` the README + workflow + SVGs, and push to `main`.
5. Optionally re-trigger `gh workflow run Metrics` once the runner shortage clears, so cards self-refresh daily.
6. Tell user to double-check `METRICS_TOKEN` scope in GitHub settings (needs `repo` scope for private data).

## Critical Context
- **Profile repo**: `/mnt/localdisk/code_base/poc/BharatSingla12` — remote `https://github.com/BharatSingla12/BharatSingla12`, branch `main`, local == origin at `8fffcbe`; 2 modified files uncommitted (`README.md`, `.github/workflows/metrics.yml`).
- **gh CLI**: authenticated as BharatSingla12; scopes `gist, read:org, repo, user, workflow`. Useful: `gh secret list`, `gh run list`, `gh run view <id>`, `gh workflow run Metrics`.
- **Secret**: `METRICS_TOKEN` exists, updated 2026-08-06T18:41:45Z. Value never retrievable via API.
- **Local metrics setup**: clone at `/tmp/metrics` (v3.34, 366f8b9). Runner script: `/tmp/run-card1.sh` (sets `INPUT_*` env vars from metrics.yml card-1 config + `INPUT_TOKEN="$(gh auth token)"`, `INPUT_OUTPUT_ACTION=none`, `INPUT_VERIFY=no`, `INPUT_DRYRUN` (unset — suspected issue), `PUPPETEER_EXECUTABLE_PATH=/usr/bin/google-chrome`, `GITHUB_REPOSITORY=BharatSingla12/BharatSingla12`). Logs: `/tmp/card1-full.log` (full), `/tmp/card1d.log` (truncated tail).
- **npm gotchas on this box**: npm 11 blocks install scripts — use `npm install-scripts approve <pkg>`; `sharp` needed `npm rebuild sharp --foreground-scripts`; `canvas` fails to compile but is unused by the metrics core (only web statics CSS) — safe to ignore.
- **Upstream metrics bugs/behaviors hit**:
  - `source/app/metrics/index.mjs:220` — `if (!libxmljs) { libxmljs.parseXml(...) }` crashes when libxmljs2 missing → set `INPUT_VERIFY=no`.
  - `source/app/action/index.mjs:100` — `conf.settings.extras = {default: true}` enables extras and probably plugin noise; investigate.
  - `source/app/action/index.mjs:472-473` — writes to hardcoded `/renders` → patched locally to `/tmp/renders`.
  - `output_action: commit` (for the real workflow) commits cards back to the repo each run.
- **Actions runs**: latest = 31126534214 (workflow_dispatch) — failed on runner acquisition; all 4 runs same cause.
- Docs consulted: lowlighter/metrics `source/plugins/core/README.md` (token/committer_token/output_action/retries), `source/plugins/base/README.md` (repositories_affiliations default `owner`).

<read-files>
/mnt/localdisk/code_base/poc/BharatSingla12/README.md
/mnt/localdisk/code_base/poc/BharatSingla12/.github/workflows/metrics.yml
/mnt/localdisk/code_base/poc/BharatSingla12/github-metrics.svg
/mnt/localdisk/code_base/poc/BharatSingla12/github-metrics-development.svg
/tmp/metrics/source/app/action/index.mjs
/tmp/metrics/source/app/metrics/index.mjs
/tmp/metrics/source/plugins/base/README.md
/tmp/metrics/source/plugins/core/README.md
/tmp/card1-full.log
</read-files>

<modified-files>
/mnt/localdisk/code_base/poc/BharatSingla12/.github/workflows/metrics.yml
/mnt/localdisk/code_base/poc/BharatSingla12/README.md
/tmp/metrics/source/app/action/index.mjs
/tmp/run-card1.sh
</modified-files>
