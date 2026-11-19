# Session learnings — 2026-11-19

Project memory for the upstream sync, the performance-inventory sampling
decision, and the move to the `localgovdrupal` organisation. Not a changelog of
every commit.

## Upstream sync

- vital-core `main` at `bb3f430` (26 non-merge commits, `f1cc4ff..bb3f430`,
  41 files) was merged into this fleet's `main` in one merge commit.
- Only `config/targets.yml` was changed on both sides, and the overlap was not
  at line level: upstream added a `sampling.performance-resources` line while
  this fleet's targets and defaults sit elsewhere in the file. Git auto-merged
  it, so the merge had no manual conflict resolution.
- **Merge, never squash.** The sync went through PR #32 and is merged with a
  merge commit so `upstream/main` stays an ancestor of `main`. Squashing would
  replay all 26 commits against already-applied files on the next
  `git merge upstream/main`.
- After a sync, `git diff upstream/main..main --stat` should list only
  fleet-only files. On 2026-11-19 that was `config/targets.yml`,
  `config/profiles/lgd.yml`, `.github/workflows/report.yml`,
  `docs-internal/UPSTREAM-SUGGESTIONS.md`, `WALSALL-FEEDBACK.md`, plus this
  kind of session note.
- The local clone now has an `upstream` remote
  (`https://github.com/mgifford/vital-core`). The documented update path is
  `git fetch upstream main && git merge upstream/main`.

## Performance inventory and sampling

- `src/engines/performance-resources.js` stores up to 200 Resource Timing
  entries per page. Measured on `mgifford/vital-core-data` at 2026-W47: 44–51
  resources per page produced 14–17 KB of JSON, 60–78% of the page record
  (~4 KB gzipped).
- This fleet scans ~2,600 pages/week and had been writing ~60 MB/week of page
  detail (W44–W46), so upstream's `sampling.performance-resources: 100` would
  add roughly 25–35 MB/week.
- The fleet therefore runs it at **25** (matching `third-party`): about 650
  pages/week, still well above the 3-page minimum for shared-resource
  candidates. Filed upstream as suggestion #34 because the per-page storage
  cost is undocumented.

## Lighthouse crash exposed by the cohort

- Stable-cohort pages now always run Lighthouse, so iframe-heavy pages are
  audited more often. On 2026-11-19 `go.walsall.gov.uk` crashed two
  consecutive scan runs with an uncaught `PROTOCOL_TIMEOUT`
  (`Network.enable`) from Lighthouse's target listener. The data still
  committed (the commit and state steps run with `if: always()`), but the
  job exited 1 and the run went red.
- Fixed upstream in [vital-core#383](https://github.com/mgifford/vital-core/pull/383)
  and merged here from the same branch. The next Walsall scan is the live
  check.
- A red `scan` job is not automatically lost data: read the failed step and
  the data repo log before rescanning.

## Data repo

- No migration, backfill, new file or `.gitignore` change was needed. New
  fields (`performanceResources`, `sustainability.measurement`,
  `thirdParty.measurement`, run-log `blockedStop`/`performanceCohort`, summary
  `performanceCohort`/`performanceInvestigations`) are additive.
- Old records are handled explicitly: the report section is omitted when there
  is no inventory, candidates are empty, and coverage is `null` rather than
  `0`. Missing evidence is not turned into a pass.
- The next aggregate rewrites each retained week's `summary.json` once to add
  the two summary keys; the values are deterministic, so later runs do not
  churn. A pruned week (no page directory) keeps its stored summary as-is.
- Cooldown and cohort state live in crawl state (`state/<key>/crawl.json`,
  kept on the `scan-state` release), not in the data repo.
- `www.lbhf.gov.uk` now stops after repeated WAF blocks and gets an eight-week
  cooldown, so expect fewer new lbhf weeks rather than failures.

## Transferring to localgovdrupal

GitHub's documented transfer behaviour (checked 2026-11-19): issues, pull
requests, wiki, stars, watchers, releases and settings transfer; **webhooks,
secrets and deploy keys remain associated**. Old repository URLs redirect for
clone, fetch and push. **GitHub Pages URLs are not redirected.** The old owner
stays a collaborator, and creating a new repository at the old name
permanently deletes the redirects.

Checklist:

1. Merge all work first (this sync is PR #32).
2. Transfer the code repo to `localgovdrupal` (Settings → General → Transfer,
   or `gh api -X POST repos/mgifford/vital-core-lgd/transfer -f
   new_owner=localgovdrupal`). Decide on the data repo: transferring
   `vital-core-lgd-data` keeps ownership consistent; leaving it personal works
   because the deploy key lives on the data repo and is preserved, but the
   org-owned repo would then depend on a personal repo.
3. Repoint the local clone:
   `git remote set-url origin git@github.com:localgovdrupal/vital-core-lgd.git`
   and keep `upstream` as `mgifford/vital-core`.
4. In the new repo update `config/targets.yml`:
   - `report_base_url: https://localgovdrupal.github.io/vital-core-lgd`
   - `data_repo: localgovdrupal/vital-core-lgd-data` (only if it moved)
   - the `user_agent` contact URL
   Commit these **after** the transfer; before it they would point at a site
   that has not moved yet.
5. Verify the settings the org can override even though the transfer carries
   them: Actions enabled and allowed to run (the only third-party action is
   `astral-sh/setup-uv`, which an org allowlist could block), Pages source set
   to GitHub Actions, `VITAL_DATA_DEPLOY_KEY` present, the write deploy key on
   the data repo, branch protection/rulesets, and the AccessLint app installed
   on the org (otherwise that PR check disappears).
6. Run Actions → scan and report (`workflow_dispatch`) and check the data push,
   the new Pages URL, the issue comment, and one CSV/`llms.txt` link. Watch the
   Walsall job: it is the known-flaky Lighthouse path until #383 is upstream. The issue
   comment derives its base URL from the repo owner and name, so it follows
   automatically; CSV, `llms.txt` and the API catalog use `report_base_url`
   and do not.
7. Update human-facing links: README, `WALSALL-FEEDBACK.md`, and any
   docs-internal report URLs.
8. Old `mgifford.github.io/vital-core-lgd/...` links will 404. If they matter,
   put a custom domain in front (for example `vital.localgovdrupal.org`) so
   the published URL is independent of the owner.
9. Do not create a repository at `mgifford/vital-core-lgd`; it may be retired,
   and a new repo there kills the redirects.
