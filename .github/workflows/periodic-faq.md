---
description: Weekly review of community questions and proposed FAQ updates.
on:
  schedule:
    - cron: "0 10 * * 1"
  workflow_dispatch:
engine:
  id: copilot
  model: gpt-4o
permissions:
  contents: read
  issues: read
  pull-requests: read
  discussions: read
  copilot-requests: write
network:
  allowed:
    - defaults
    - github.com
    - accounts.eclipse.org
safe-outputs:
  create-issue:
    title-prefix: "[faq-review] "
    labels: [documentation, faq]
    close-older-issues: false
---

# Adoptium FAQ Review

Review recent community questions and identify opportunities to improve the Adoptium FAQ. The goal is to proactively identify recurring questions, documentation gaps, and outdated FAQ entries, then produce a single report containing proposed additions and improvements for maintainer review.

## Scope

Review questions and discussions from the previous 7 days across the following sources:

- Slack channels:
  - `#support`
  - `#community`
  - `#general`
- Issues in the `adoptium/adoptium-support` repository:
  - https://github.com/adoptium/adoptium-support
- Issues across the Adoptium GitHub organization labelled `PMC-agenda`.
- The Adoptium PMC mailing list archives:
  - https://accounts.eclipse.org/mailing-list/adoptium-pmc

Compare the questions found against the current FAQ before proposing any additions.

## What to include

Produce a single report containing:

### Summary

- Overall trends observed this week.
- Whether question volume has increased or decreased.
- Common themes that emerged.

### Proposed New FAQ Entries

For each proposed FAQ entry include:

- Proposed question/title.
- A concise draft answer.
- Why this should become an FAQ.
- Links to the discussions that motivated it.

### Existing FAQ Improvements

Identify existing FAQ entries that should be:

- expanded
- clarified
- updated
- merged with another entry
- removed if obsolete

Explain the reasoning for each recommendation.

### Frequently Asked Questions

List questions that appeared multiple times during the review period, even if they are already adequately covered by the FAQ.

### Documentation Gaps

Highlight areas where users consistently struggled to find answers or where existing documentation appears insufficient.

## Process

1. Review discussions from all configured sources since the previous run.
2. Extract user questions and maintainer responses.
3. Group semantically similar questions together.
4. Compare each topic against the existing FAQ.
5. Determine whether each topic:
   - is already covered,
   - requires an update to an existing FAQ entry,
   - or warrants a new FAQ entry.
6. Draft proposed FAQ additions or improvements.
7. Create a single GitHub issue containing the complete review.

## Constraints

- Focus on recurring or broadly useful questions rather than one-off support requests.
- Avoid proposing duplicate FAQ entries.
- Prefer official maintainer responses over community speculation when drafting answers.
- Include links to the original discussions where possible.
- Keep the report concise and actionable.
- Do not modify the FAQ directly.
- Create exactly one GitHub issue containing the review and recommendations.
