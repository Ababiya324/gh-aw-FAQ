---
description: "Continuous multi-channel review of community questions with proposed Adoptium FAQ updates."
on:
  schedule: daily             # fuzzy daily schedule; compiler picks a distributed time
  workflow_dispatch:
    inputs:
      since:
        description: "Start date YYYY-MM-DD (default: 7 days ago)"
        required: false
      until:
        description: "End date YYYY-MM-DD (default: now)"
        required: false
engine:
  id: copilot
  model: gpt-4.1              # free-tier eligible + reliable tool-caller; gpt-4o skips safe-output calls, claude models are not enabled for this account
permissions:
  contents: read
  issues: read
  pull-requests: read
  discussions: read
  copilot-requests: write     # use Actions token-based Copilot inference (no PAT secret)
network:
  allowed:
    - defaults
    - github
    - node                    # npx fetches the Slack MCP server
    - "slack.com"
    - "*.slack.com"
    - "api.stackexchange.com"   # web-fetch fallback for the pre-fetched sources
    - "www.reddit.com"
    - "www.eclipse.org"
# Deterministic pre-fetch: runs outside the firewall sandbox, before the agent.
# Pulls the FAQ + GitHub questions with plain curl and writes them to files, so the
# agent spends inference on judgement (cluster/compare/draft), not on fetching.
steps:
  - name: Pre-fetch FAQ and GitHub sources
    env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      FAQ_SINCE: ${{ github.event.inputs.since }}
      FAQ_UNTIL: ${{ github.event.inputs.until }}
    run: |
      set -euo pipefail
      DIR=/tmp/gh-aw/agent
      mkdir -p "$DIR"

      # --- Sources config (portable across projects). Read from
      # .github/faq-sources.json; fall back to the Adoptium defaults if the file
      # is missing or unparseable, so a run never hard-fails on config. ---
      CFG="$GITHUB_WORKSPACE/.github/faq-sources.json"
      cfg() { jq -r "$1" "$CFG" 2>/dev/null; }
      ISSUE_REPOS=(); SO_TAGS=(); MAILING_LISTS=()
      if [ -f "$CFG" ] && jq -e . "$CFG" >/dev/null 2>&1; then
        ORG=$(cfg '.github.org')
        SUPPORT_REPO=$(cfg '.github.support_repo')
        PMC_LABEL=$(cfg '.github.pmc_agenda_label')
        FAQ_URL=$(cfg '.faq_url')
        REDDIT_QUERY=$(cfg '.reddit_query')
        mapfile -t ISSUE_REPOS < <(cfg '.github.issue_repos[]')
        mapfile -t SO_TAGS < <(cfg '.stackoverflow_tags[]')
        mapfile -t MAILING_LISTS < <(cfg '.mailing_lists[]')
      fi
      : "${ORG:=adoptium}"
      : "${SUPPORT_REPO:=adoptium/adoptium-support}"
      : "${PMC_LABEL:=PMC-agenda}"
      : "${FAQ_URL:=https://raw.githubusercontent.com/adoptium/adoptium.net/main/content/asciidoc-pages/docs/faq/index.adoc}"
      : "${REDDIT_QUERY:=temurin OR adoptium}"
      [ "${#ISSUE_REPOS[@]}" -gt 0 ] || ISSUE_REPOS=(installer containers adoptium.net)
      [ "${#SO_TAGS[@]}" -gt 0 ] || SO_TAGS=(adoptium temurin adoptopenjdk)
      [ "${#MAILING_LISTS[@]}" -gt 0 ] || MAILING_LISTS=(adoptium-pmc temurin-dev)

      # --- Review window. Explicit since/until (workflow_dispatch inputs) mean a
      # one-off backfill; otherwise default to the past 7 days. ---
      if [ -n "${FAQ_SINCE:-}" ] || [ -n "${FAQ_UNTIL:-}" ]; then
        MODE=backfill
        SINCE_DAY="${FAQ_SINCE:-$(date -u -d '7 days ago' +%Y-%m-%d)}"
        UNTIL_DAY="${FAQ_UNTIL:-$(date -u +%Y-%m-%d)}"
      else
        MODE=incremental
        SINCE_DAY=$(date -u -d '7 days ago' +%Y-%m-%d)
        UNTIL_DAY=$(date -u +%Y-%m-%d)
      fi
      SINCE="${SINCE_DAY}T00:00:00Z"
      UNTIL="${UNTIL_DAY}T23:59:59Z"
      SINCE_EPOCH=$(date -u -d "$SINCE_DAY" +%s)
      UNTIL_EPOCH=$(date -u -d "${UNTIL_DAY} 23:59:59" +%s)
      echo "$SINCE" > "$DIR/window-start.txt"
      echo "$UNTIL" > "$DIR/window-end.txt"
      echo "$MODE"  > "$DIR/window-mode.txt"
      echo "Window: $SINCE .. $UNTIL  (mode: $MODE)"

      # Live FAQ
      curl -sSL "$FAQ_URL" -o "$DIR/faq.adoc"
      # Support repo issues (open + closed) updated in the window, authenticated
      # (5000/hr limit instead of 60). Headers as an array so the token survives
      # word splitting.
      GH_API=(-H "Accept: application/vnd.github+json" -H "Authorization: Bearer $GH_TOKEN")
      curl -sSL "${GH_API[@]}" \
        "https://api.github.com/repos/$SUPPORT_REPO/issues?state=open&since=$SINCE&per_page=100" \
        -o "$DIR/support-open.json"
      curl -sSL "${GH_API[@]}" \
        "https://api.github.com/repos/$SUPPORT_REPO/issues?state=closed&since=$SINCE&per_page=100" \
        -o "$DIR/support-closed.json"
      # Org-wide PMC-agenda issues via the search API (reads .items)
      curl -sSL "${GH_API[@]}" \
        "https://api.github.com/search/issues?q=org:$ORG+label:$PMC_LABEL+state:open&per_page=50" \
        -o "$DIR/pmc-agenda.json"
      # Issues from other user-facing repos: install, container, and website
      # questions that never reach the support repo
      for REPO in "${ISSUE_REPOS[@]}"; do
        curl -sSL "${GH_API[@]}" \
          "https://api.github.com/repos/$ORG/$REPO/issues?state=all&since=$SINCE&per_page=100" \
          -o "$DIR/issues-$REPO.json"
      done
      # GitHub Discussions across the org (GraphQL search; best effort)
      gh api graphql -f query="
        query {
          search(query: \"org:$ORG updated:$SINCE_DAY..$UNTIL_DAY\", type: DISCUSSION, first: 50) {
            nodes { ... on Discussion {
              title body url updatedAt repository { nameWithOwner }
              comments(first: 10) { nodes { body } }
            } }
          }
        }" > "$DIR/discussions.json" \
        || echo '{"data":{"search":{"nodes":[]}}}' > "$DIR/discussions.json"
      # Stack Overflow questions per tag (public API; responses are always gzip,
      # hence --compressed; bounded by fromdate/todate; best effort)
      for TAG in "${SO_TAGS[@]}"; do
        curl -sSL --compressed \
          "https://api.stackexchange.com/2.3/questions?order=desc&sort=creation&tagged=$TAG&site=stackoverflow&fromdate=$SINCE_EPOCH&todate=$UNTIL_EPOCH&filter=withbody" \
          -o "$DIR/so-$TAG.json" || echo '{"items":[]}' > "$DIR/so-$TAG.json"
      done
      # Reddit mentions (public JSON; Reddit sometimes blocks CI IPs — best effort)
      REDDIT_Q=$(jq -rn --arg q "$REDDIT_QUERY" '$q|@uri')
      curl -sSL -A "adoptium-faq-review:v1.0 (github-actions)" \
        "https://www.reddit.com/search.json?q=$REDDIT_Q&sort=new&t=week&limit=50&raw_json=1" \
        -o "$DIR/reddit.json" || echo '{"data":{"children":[]}}' > "$DIR/reddit.json"
      # Mailing list archives (actual MHonArc archive pages, not the
      # accounts.eclipse.org subscription page); best effort, never fail the run.
      for ML in "${MAILING_LISTS[@]}"; do
        curl -sSL "https://www.eclipse.org/lists/$ML/" \
          -o "$DIR/mailing-$ML.html" || echo "mailing list unavailable" > "$DIR/mailing-$ML.html"
      done
      # Slim the pre-fetched payloads deterministically: keep only the fields the
      # agent uses and cap long bodies. This runs for free (no inference) and
      # sharply cuts the tokens the agent must read. Field names are preserved, so
      # the documented file formats below still hold. jq failures leave the
      # original file untouched (best effort — never fail the run).
      CAP=2500
      slim() { jq "$2" "$1" > "$1.slim" 2>/dev/null && mv "$1.slim" "$1" || rm -f "$1.slim"; }
      ISSUE_FILTER="[.[]? | {number, title, state, created_at, updated_at, html_url, labels: [.labels[]?.name], body: ((.body // \"\")[0:$CAP])}]"
      for f in support-open support-closed "${ISSUE_REPOS[@]/#/issues-}"; do
        slim "$DIR/$f.json" "$ISSUE_FILTER"
      done
      slim "$DIR/pmc-agenda.json" "{items: [.items[]? | {number, title, state, created_at, html_url, repository_url, body: ((.body // \"\")[0:$CAP])}]}"
      for TAG in "${SO_TAGS[@]}"; do
        slim "$DIR/so-$TAG.json" "{items: [.items[]? | {question_id, title, link, creation_date, tags, body: ((.body // \"\")[0:$CAP])}]}"
      done
      slim "$DIR/discussions.json" "{data: {search: {nodes: [.data.search.nodes[]? | {title, url, updatedAt, repository, body: ((.body // \"\")[0:$CAP]), comments: {nodes: [.comments.nodes[]? | {body: ((.body // \"\")[0:1000])}]}}]}}}"
      slim "$DIR/reddit.json" "{data: {children: [.data.children[]? | {data: {title: .data.title, subreddit: .data.subreddit, permalink: .data.permalink, created_utc: .data.created_utc, selftext: ((.data.selftext // \"\")[0:2000])}}]}}"
      # Mailing list archives: strip HTML to plain text and cap — the raw pages
      # are mostly markup with little content. Keep the same filenames.
      for ML in "${MAILING_LISTS[@]}"; do
        M="mailing-$ML"
        if [ -f "$DIR/$M.html" ]; then
          sed -e 's/<[^>]*>/ /g' "$DIR/$M.html" | tr -s ' \t\n' ' ' | cut -c1-6000 > "$DIR/$M.txt" 2>/dev/null \
            && mv "$DIR/$M.txt" "$DIR/$M.html" || rm -f "$DIR/$M.txt"
        fi
      done
      echo "Pre-fetch complete:"; ls -la "$DIR"
tools:
  web-fetch:                  # fallback fetch if a pre-fetched file is missing
  cache-memory:               # per-channel watermark across runs
mcp-servers:
  slack:
    command: "npx"
    args: ["-y", "@modelcontextprotocol/server-slack"]
    env:
      SLACK_BOT_TOKEN: "${{ secrets.SLACK_BOT_TOKEN }}"
      SLACK_TEAM_ID: "${{ secrets.SLACK_TEAM_ID }}"
    allowed:                  # read-only tools only (enforced at the gateway)
      - slack_list_channels
      - slack_get_channel_history
      - slack_get_thread_replies
safe-outputs:
  create-issue:
    title-prefix: "[faq-review] "
    labels: [documentation, faq]
    expires: false
---

# Adoptium FAQ Review

You review community questions across several channels and propose improvements to
the **Adoptium FAQ**. You do **not** edit the FAQ directly; you produce a single
GitHub issue with ready-to-apply proposals for maintainer review.

## Pre-fetched inputs (read these, do not re-fetch)

A deterministic step has already downloaded the data into `/tmp/gh-aw/agent/`. The
exact set of sources is driven by `.github/faq-sources.json` (org, repos, Stack
Overflow tags, mailing lists, FAQ URL), so filenames follow the patterns below
rather than a fixed list. Read these files directly instead of making web requests:

- `faq.adoc` — the live FAQ. Parse its `==` question headings first, so every
  comparison is against the current entries.
- `support-open.json`, `support-closed.json` — support-repo issues (a JSON array).
  Review each item's `title`, `body`, and comments.
- `pmc-agenda.json` — org-wide PMC-agenda issues. Read the `items` array.
- `issues-<repo>.json` — one per configured issue repo (e.g. `issues-installer.json`,
  `issues-containers.json`, `issues-adoptium.net.json`): issues (open and closed,
  JSON arrays) from the other user-facing repos. Read every `issues-*.json` present.
- `discussions.json` — GitHub Discussions across the org. Read
  `data.search.nodes[]`: `title`, `body`, `url`, `repository.nameWithOwner`, and
  `comments.nodes[].body`.
- `so-<tag>.json` — one per configured Stack Overflow tag (e.g. `so-adoptium.json`,
  `so-temurin.json`, `so-adoptopenjdk.json`), Stack Exchange API format. Read
  `items[]`: `title`, `body`, `link`, `creation_date`. The same question may appear
  under more than one tag — dedupe by `question_id`. Read every `so-*.json` present.
- `mailing-<list>.html` — one per configured mailing list (e.g.
  `mailing-adoptium-pmc.html`, `mailing-temurin-dev.html`): the archive text, or
  "mailing list unavailable" if the fetch failed (then skip it and note so). Read
  every `mailing-*.html` present.
- `window-start.txt`, `window-end.txt` — the ISO 8601 start and end of the review
  window (bound both ends of what you consider).
- `window-mode.txt` — `incremental` (normal scheduled run) or `backfill` (an
  explicit date range was requested). See **Continuity across runs**.

Only fall back to `web-fetch` if one of these files is missing or empty. If a file
exists but is not parseable in its documented format (e.g. an HTML block page or an
API error body instead of JSON), treat that channel as **failed**, quote a short
snippet of what was found, and move on — never invent data for it.

Note: item bodies in these files are truncated (~2500 chars) to conserve context.
That is enough to cluster and classify; if you need the full text of a specific
item to draft a proposal, follow its link rather than re-fetching the whole source.

## FAQ format

AsciiDoc. Each entry is a second-level heading plus prose:

```asciidoc
== How long is Eclipse Temurin supported?

Answer text. Internal links use link:/temurin/releases[label];
external links use https://example.com[label].
```

## Continuity across runs

First read `window-mode.txt`; it decides how you treat the watermark.

**`incremental`** (normal scheduled run): use `cache-memory` with the key
`watermark` (an ISO 8601 timestamp). Read it at the start; process only items newer
than it (and no later than `window-end.txt`); write the newest processed timestamp
back at the end. If the key is missing (first run), use `window-start.txt` as the
cutoff. This keeps the review incremental.

**`backfill`** (an explicit date range was requested): process items within
`window-start.txt`..`window-end.txt`. **Do not** read or write the `watermark` — a
manual backfill is a one-off and must not disturb the scheduled cadence.

## Slack channel

Reading Slack is **mandatory** every run — it is a primary source, not an optional
extra. You MUST call `slack_list_channels`, then `slack_get_channel_history` for
**every channel where the bot is a member** (`is_member: true` in the
`slack_list_channels` response — do not hardcode a channel list; membership is the
source of truth), and `slack_get_thread_replies` for threads, for messages since
the watermark. If `is_member` is absent from the response, attempt the history
call on every listed channel and treat `not_in_channel` as "not a member" rather
than a failure. Do this before drafting any proposals; do not skip it because the
other sources already yielded enough material.

To conserve tokens, request a modest page size (`limit: 50`) — only messages newer
than the watermark matter, so there is no need to pull the maximum history. Save
large tool outputs to a file and filter with `jq`/`grep` rather than reading them
whole into context.

The only acceptable reason to not read a member channel is a genuine tool
failure — the Slack MCP tool returns an error (e.g. `missing_scope`,
`invalid_auth`, a rate limit / `429`, or the tool is absent). Channels the bot is
not a member of are simply listed as "not a member" in Channel Coverage — that is
a skip, not a failure. For genuine failures:
- Attempt the call anyway; do not assume it will fail without trying.
- In **Channel Coverage**, record the exact tool name and the verbatim error string
  returned (e.g. "`slack_get_channel_history` → `missing_scope`"). Never write a
  vague "tooling not invoked" — if you did not invoke it, that is a failure to
  follow instructions, not a valid skip.
- Reading other sources may still continue, but a Slack failure must be surfaced
  loudly in the report so it can be fixed.

## Process

1. Read `window-mode.txt`, `window-start.txt`, and `window-end.txt`. In
   `incremental` mode also read the `watermark` from `cache-memory`; in `backfill`
   mode ignore the watermark entirely.
2. Parse `faq.adoc` and list its existing questions.
3. Read **all** the pre-fetched files (support issues, other-repo issues,
   discussions, Stack Overflow, Reddit, mailing lists), then read the Slack history
   (mandatory — call `slack_list_channels` and `slack_get_channel_history` as
   described above; only a returned tool error excuses a channel). Collect
   questions within the window (in `incremental` mode, newer than the watermark and
   no later than `window-end.txt`). Note which channels had data, which were empty,
   and which failed with what error.
4. Cluster semantically similar questions (across channels — the same question in
   Slack, on Stack Overflow, and in a support issue counts once).
5. Classify each cluster vs. the FAQ: already covered; covered but needs
   update/expansion; or not covered (new-entry candidate).
6. Draft proposals (format below).
7. In `incremental` mode, write the updated `watermark` to `cache-memory`. In
   `backfill` mode, do not touch the watermark.
8. **Terminal action — end with exactly one safe output:**
   - Anything worth reporting → call `create_issue` with the full report.
   - Every reachable channel genuinely empty → call `noop` with a one-line reason.
   - Never finish without a safe-output call.
## Output: issue format

### Summary
Trends this period; volume up or down; common themes.

### Channel Coverage
One line per channel — every pre-fetched source (adoptium-support, installer,
containers, adoptium.net, org Discussions, Stack Overflow, Reddit, each mailing
list) plus Slack — stating read / empty / failed, and on failure why. For Slack
specifically, state whether `slack_list_channels` / `slack_get_channel_history`
were called and, on failure, the verbatim error returned. "Not invoked" is not an
acceptable Slack status — it means the mandatory read was skipped.

### Proposed New FAQ Entries
Per proposal, in order: why it belongs (1–2 sentences); motivating discussions
(links, or Slack channel + approximate time); a paste-ready AsciiDoc block matching
the existing style:

```asciidoc
== <Proposed question>?

<Draft answer in AsciiDoc, using link:/path[label] and https://url[label]>
```

Then: **Apply to:** `content/asciidoc-pages/docs/faq/index.adoc` in `adoptium/adoptium.net`.

### Existing FAQ Improvements
Entries to expand, clarify, update, merge, or remove — name the `== Heading`, the
change, and the reasoning. Give proposed wording as a paste-ready AsciiDoc block.

### Frequently Asked (already covered)
Recurring questions already handled adequately — useful signal, no change needed.

### Documentation Gaps
Where users consistently struggled to find answers.

## Constraints

- Do **not** modify the FAQ directly; propose only.
- Every block must be valid AsciiDoc matching the existing style.
- Prefer official maintainer answers over community speculation.
- Focus on recurring or broadly useful questions, not one-offs.
- Never duplicate an existing entry; compare against `faq.adoc` first.
- Include source links wherever possible.
- Exactly one issue per run, or a `noop`. Never neither.
