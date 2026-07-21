# periodic-faq: configurable sources + date range

**Date:** 2026-07-22
**Status:** Approved
**Scope:** `periodic-faq` agentic workflow (personal sandbox repo `Ababiya324/gh-aw-FAQ`; to be ported to `adoptium/adoptium` afterwards).

## Motivation

Enhancement requests from the Semesters of Code call (via Shelley Lambert):

1. Read the list of monitored communication channels from a file, so the workflow is portable to other projects.
2. Read the FAQ location and open a draft PR to update it. **Deferred** — the FAQ lives in a different repo (`adoptium/adoptium.net`) than the workflow, so a cross-repo PR needs a PAT/App token decision with the maintainers. This iteration keeps the existing `create-issue` output.
3. Make the review window configurable (cover channels between date X and date Y) instead of a hardcoded past week.

This spec covers requests 1 and 3.

## Feature 1 — Sources config file

New file `.github/faq-sources.json`, read by the deterministic pre-fetch step with `jq` (already present on GitHub-hosted runners):

```json
{
  "github": {
    "org": "adoptium",
    "support_repo": "adoptium/adoptium-support",
    "issue_repos": ["installer", "containers", "adoptium.net"],
    "pmc_agenda_label": "PMC-agenda"
  },
  "stackoverflow_tags": ["adoptium", "temurin", "adoptopenjdk"],
  "reddit_query": "temurin OR adoptium",
  "mailing_lists": ["adoptium-pmc", "temurin-dev"],
  "faq_url": "https://raw.githubusercontent.com/adoptium/adoptium.net/main/content/asciidoc-pages/docs/faq/index.adoc"
}
```

- The pre-fetch reads each value with `jq -r` and loops over arrays.
- Output filenames keep existing patterns: `faq.adoc`, `support-open.json`, `support-closed.json`, `issues-<repo>.json`, `pmc-agenda.json`, `discussions.json`, `so-<tag>.json`, `reddit.json`, `mailing-<list>.html`. Only *which* set exists becomes config-driven.
- Slack is unaffected — coverage is driven by bot channel membership, not config.
- Portability: another project swaps only this file.
- Robustness: if the config file is missing or unparseable, the pre-fetch falls back to the current hardcoded Adoptium values so a run never hard-fails on config.

## Feature 2 — Configurable date range

Add `workflow_dispatch` inputs:

```yaml
workflow_dispatch:
  inputs:
    since:
      description: "Start date YYYY-MM-DD (default: 7 days ago)"
      required: false
    until:
      description: "End date YYYY-MM-DD (default: now)"
      required: false
```

Pre-fetch behavior:

- **Neither input set** (scheduled runs, default dispatch): `since` = 7 days ago, `until` = now — identical to current behavior. Write `window-mode.txt` = `incremental`.
- **Inputs set** (explicit range): `since`/`until` from inputs. Write `window-mode.txt` = `backfill`.
- Write `window-start.txt` (= since) and new `window-end.txt` (= until) for the agent.
- Widen the fetch from `since`; the far end is bounded by the agent's date filtering. Add `todate`/date-range parameters to Stack Overflow (`todate`) and Discussions (`updated:since..until`) where the APIs support it.

Watermark interaction (one-off backfill semantics):

- **incremental:** unchanged — read `watermark` from `cache-memory` (or `window-start.txt` on first run) as the cutoff; process items newer than it; write the newest processed timestamp back at the end.
- **backfill:** use the explicit `window-start.txt`..`window-end.txt` window; **ignore** the watermark and **do not** write it back. A manual backfill does not disturb the scheduled cadence.

## Prompt changes

- Note the source set and window are produced by the pre-fetch (config-driven; window may be custom).
- Add the backfill rule: read `window-mode.txt`; when `backfill`, use `window-start.txt`..`window-end.txt`, skip the watermark read, and skip writing the watermark.
- Reference `window-end.txt` as the far bound.

## Unchanged

Deterministic-prefetch architecture, `create-issue` safe output, the token-slimming pass, Slack handling, weekly schedule, `copilot-requests: write` auth.

## Testing

On `Ababiya324/gh-aw-FAQ`:
1. Default run (no inputs) — confirm config-driven fetch reproduces current sources and produces an issue/no-op.
2. Dispatch with explicit `since`/`until` — confirm backfill window is used and the watermark is not advanced.
