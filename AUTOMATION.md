# Account automation library

This repository is the central "brain" for automation across all
`Uncle-Becky` repositories. Workflow logic lives here once; every other
repo calls it with a small stub. Improving a reusable workflow here
upgrades every repository at the same time.

## What's here

| Workflow | Kind | Purpose |
| --- | --- | --- |
| `claude-review.yml` | Reusable (`workflow_call`) | Claude reviews every non-draft PR, grounded in the calling repo's own CLAUDE.md / ARCHITECTURE.md / ROADMAP.md, plus per-repo instructions passed as an input. |
| `claude-agent.yml` | Reusable (`workflow_call`) | On-demand executor: mention `@claude <task>` in any issue or PR comment and it investigates, implements, and pushes. |
| `daily-digest.yml` | Scheduled (runs in this repo) | Sweeps all unread notifications account-wide each morning into a single digest issue here. Delegate any item by commenting `@claude ...` on the digest. |

## One-time setup checklist

1. **Install the Claude GitHub App account-wide** — <https://github.com/apps/claude>,
   choose **All repositories** (covers future repos automatically).
2. **`ANTHROPIC_API_KEY` secret in each repo** that uses review/agent.
   Personal accounts can't share Actions secrets across repos; script it:

   ```bash
   gh repo list Uncle-Becky --limit 200 --json name -q '.[].name' | \
     xargs -I{} gh secret set ANTHROPIC_API_KEY \
       -R Uncle-Becky/{} --body "$ANTHROPIC_API_KEY"
   ```

   Re-run after creating a new repo. (Set a monthly spend limit on the
   key in the Anthropic Console.)
3. **`GH_NOTIFICATIONS_TOKEN` secret in THIS repo only** — a classic PAT
   with the `notifications` scope (read-only), for the daily digest.
   The built-in `GITHUB_TOKEN` cannot read user notifications.

## Adding automation to a repository

Create `<repo>/.github/workflows/claude.yml`:

```yaml
name: Claude

on:
  pull_request:
    types: [opened, ready_for_review, synchronize]
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

jobs:
  review:
    if: github.event_name == 'pull_request' && !github.event.pull_request.draft
    uses: Uncle-Becky/.github/.github/workflows/claude-review.yml@main
    with:
      review_instructions: |
        # Optional: repo-specific review focus, e.g.:
        # - All archive.org I/O must live in the service layer.
    secrets: inherit

  agent:
    if: >-
      github.event_name != 'pull_request' &&
      contains(github.event.comment.body, '@claude')
    uses: Uncle-Becky/.github/.github/workflows/claude-agent.yml@main
    secrets: inherit
```

That stub is the entire per-repo footprint.

## Roadmap (the digital-twin build-out)

- **Phase 1 (this repo, now):** shared review brain, @claude executor,
  daily inbox digest.
- **Phase 2:** custom GitHub App identity ("the twin"), webhook receiver +
  event queue, triage policies with a per-category autonomy ladder
  (observe → propose → act-with-review → act autonomously), append-only
  decision journal.
- **Phase 3:** Claude Agent SDK orchestrator, weekly self-retrospective
  where the twin reviews its journal and proposes policy amendments —
  which only a human merges.
