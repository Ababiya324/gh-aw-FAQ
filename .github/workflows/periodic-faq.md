---
description: "Continuous multi-channel review of community questions with proposed Adoptium FAQ updates."
on:
  schedule: daily             # fuzzy daily schedule; compiler picks a distributed time
  workflow_dispatch:
engine:
  id: copilot
  model: gpt-4.1              # free-tier eligible + reliable tool-caller; gpt-4o skips safe-output calls, claude models are not enabled for this account
permissions:
  contents: read
  issues: read
  pull-requests: read
  discussions: read
network:
  allowed:
    - defaults
    - github
    - node                    # npx fetches the Slack MCP server
    - "slack.com"
    - "*.slack.com"
# Deterministic pre-fetch: runs outside the firewall sandbox, before the agent.
# Pulls the FAQ + GitHub questions with plain curl and writes them to files, so the
# agent spends inference on judgement (cluster/compare/draft), not on fetching.
steps:
  - name: Pre-fetch FAQ and GitHub sources
    env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    run: |
      set -euo pipefail
      DIR=/tmp/gh-aw/agent
      mkdir -p "$DIR"
      SINCE=$(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%SZ)
      echo "$SINCE" > "$DIR/window-start.txt"
      # Live FAQ
      curl -sSL \
        "https://raw.githubusercontent.com/adoptium/adoptium.net/main/content/asciidoc-pages/docs/faq/index.adoc" \
        -o "$DIR/faq.adoc"
      # adoptium-support issues (open + closed) updated in the window, authenticated
      # (5000/hr limit instead of 60). Headers as an array so the token survives
      # word splitting.
      GH_API=(-H "Accept: application/vnd.github+json" -H "Authorization: Bearer $GH_TOKEN")
      curl -sSL "${GH_API[@]}" \
        "https://api.github.com/repos/adoptium/adoptium-support/issues?state=open&since=$SINCE&per_page=100" \
        -o "$DIR/support-open.json"
      curl -sSL "${GH_API[@]}" \
        "https://api.github.com/repos/adoptium/adoptium-support/issues?state=closed&since=$SINCE&per_page=100" \
        -o "$DIR/support-closed.json"
      # Org-wide PMC-agenda issues via the search API (reads .items)
      curl -sSL "${GH_API[@]}" \
        "https://api.github.com/search/issues?q=org:adoptium+label:PMC-agenda+state:open&per_page=50" \
        -o "$DIR/pmc-agenda.json"
      # PMC mailing list archive (best effort; never fail the run)
      curl -sSL "https://accounts.eclipse.org/mailing-list/adoptium-pmc" \
        -o "$DIR/mailing-list.html" || echo "mailing list unavailable" > "$DIR/mailing-list.html"
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

A deterministic step has already downloaded the data into `/tmp/gh-aw/agent/`. Read
these files directly instead of making web requests:

- `faq.adoc` — the live FAQ. Parse its `==` question headings first, so every
  comparison is against the current entries.
- `support-open.json`, `support-closed.json` — `adoptium/adoptium-support` issues
  (a JSON array). Review each item's `title`, `body`, and comments.
- `pmc-agenda.json` — org-wide `PMC-agenda` issues. Read the `items` array.
- `mailing-list.html` — PMC mailing list archive, or the text "mailing list
  unavailable" if the fetch failed (then skip it and note so).
- `window-start.txt` — the ISO 8601 start of the 7-day pre-fetch window.

Only fall back to `web-fetch` if one of these files is missing or empty.

## FAQ format

AsciiDoc. Each entry is a second-level heading plus prose:

```asciidoc
== How long is Eclipse Temurin supported?

Answer text. Internal links use link:/temurin/releases[label];
external links use https://example.com[label].
```

## Continuity across runs

Use `cache-memory` with the key `watermark` (an ISO 8601 timestamp). Read it at the
start; process only items newer than it; write the newest processed timestamp back
at the end. If the key is missing (first run), use `window-start.txt` as the cutoff.
This keeps the review incremental.

## Slack channel

Read `#support`, `#community`, `#general` via the Slack MCP tools: call
`slack_list_channels` to resolve IDs, then `slack_get_channel_history` per channel
(and `slack_get_thread_replies` for threads) for messages since the watermark. If
Slack tools are unavailable, skip and note it in Channel Coverage; do not abort.

## Process

1. Read the `watermark` from `cache-memory`.
2. Parse `faq.adoc` and list its existing questions.
3. Read the pre-fetched JSON files and the Slack history; collect questions newer
   than the watermark. Note which channels had data and which were empty/skipped.
4. Cluster semantically similar questions (across channels — the same question in
   Slack and in a support issue counts once).
5. Classify each cluster vs. the FAQ: already covered; covered but needs
   update/expansion; or not covered (new-entry candidate).
6. Draft proposals (format below).
7. Write the updated `watermark` to `cache-memory`.
8. **Terminal action — end with exactly one safe output:**
   - Anything worth reporting → call `create_issue` with the full report.
   - Every reachable channel genuinely empty → call `noop` with a one-line reason.
   - Never finish without a safe-output call.
## Output: issue format

### Summary
Trends this period; volume up or down; common themes.

### Channel Coverage
Which channels were read, and which were skipped and why.

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
