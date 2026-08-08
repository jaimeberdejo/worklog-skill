---
name: worklog
description: Reconstruct evidence-based developer work from GitHub for dailies, weekly summaries, retrospectives, and repo/ticket/user lookups. Use when asked what someone worked on during a period or when /worklog or /productivity is invoked. Uses GitHub through gh only; no companion scripts. Never score or rank developer productivity.
---

# Worklog

Reconstruct work from GitHub as:

`activity -> evidence -> WorkItem -> state -> outcome -> blocker/follow-up`

The primary unit is a **WorkItem**: ticket/linked issue when possible, otherwise PR, issue, or coherent unlinked work. Commits, diffs, LOC, files, checks, reviews, and comments are supporting evidence, not productivity units.

## Non-negotiable rules

- Never calculate productivity, velocity, effort, quality, or developer scores.
- Never rank developers.
- Never use commit count, PR count, LOC, or changed-file count as proxies for productivity.
- Never invent ticket links, blockers, next steps, outcomes, or completion.
- Never attribute another person's implementation/review/comment/merge to the selected user.
- Never claim implementation happened inside a period just because the PR merged there.
- Prefer structured GitHub relationships over text inference.
- Preserve uncertainty when evidence is incomplete.
- If nothing attributable is found, say **"No attributable GitHub activity was found"**, not "no work was done".
- Require only `gh`; do not depend on Python, Node, jq, yq, or companion scripts.

---

# Interface

Support:

```text
/worklog today
/worklog yesterday
/worklog week
/worklog since monday
/worklog last 7 days
/worklog 2026-08-01..2026-08-07

/worklog today --repo frontend
/worklog week --repo owner/backend
/worklog today --ticket RBI-023
/worklog week --user @developer
/worklog yesterday --verbose
/worklog today --note "investigated migration options for X"
```

`/productivity` may be accepted as an alias, but call the result a **worklog** or **work summary**, never a productivity score.

Keep flags limited to:

```text
--repo <repo>
--ticket <ticket>
--user <login|@login|me>
--verbose
--note "..."
```

Defaults:

- user = authenticated GitHub user;
- repo = all relevant accessible repos;
- day range = concise daily output;
- week/custom range = explanatory output (see section 17);
- `--verbose` = append compact evidence/provenance.

---

# 1. Preflight

Verify GitHub CLI and authentication before doing anything else:

```bash
gh --version
gh auth status
gh api user --jq '.login'
```

If unavailable or unauthenticated, stop. Do not construct a worklog from memory.

Preflight already verified earlier in the same session need not be re-run.

Resolve selected user:

- omitted / `me` -> authenticated login;
- `@alice` -> `alice`;
- `alice` -> `alice`.

Do not resolve people by display name.

---

# 2. Resolve the period

Use the user's/local environment timezone for calendar dates. Inspect it when needed:

```bash
date '+%Y-%m-%d %H:%M:%S %z'
```

Resolve internally:

```text
START_DATE = YYYY-MM-DD
END_DATE   = YYYY-MM-DD
START_AT   = exact local timestamp
END_AT     = exact local timestamp
```

Definitions:

- `today`: today 00:00 -> now.
- `yesterday`: previous calendar day 00:00 -> 23:59:59.
- `week` / `since monday`: Monday 00:00 -> now.
- `last 7 days`: six calendar days before today 00:00 -> now.
- `A..B`: A 00:00 -> B 23:59:59, inclusive.

GitHub search date qualifiers are interpreted in UTC and filter by calendar date, not exact timestamp. Events near local midnight can fall on a different UTC date and would be missed by an exact-date search, so widen the candidate window:

```text
SEARCH_START_DATE = START_DATE minus 1 day
SEARCH_END_DATE   = END_DATE plus 1 day
```

Use the widened dates to find candidates, then exact event timestamps to decide what belongs to the period.

API timestamps are UTC (`Z` suffix). Convert to the local timezone before naming the day an event happened; a raw UTC date prefix misdates events within a few hours of midnight.

`updatedAt` alone never proves the selected user acted at that time.

---

# 3. Optional repo conventions

The skill must work with zero configuration.

If `.worklog.yml` / `.worklog.yaml` or an established project config contains worklog settings, read it directly and use it as hints, for example:

```yaml
worklog:
  ticket_patterns:
    - '\\bRBI-\\d+\\b'
    - '\\bWP-\\d+\\b'
  ignore:
    bots: [dependabot[bot], github-actions[bot], renovate[bot]]
    paths: ['**/generated/**', '**/*.lock']
  tests: ['tests/**', '**/*.test.*', '**/*.spec.*']
  docs: ['docs/**', 'README*']
```

Without config, conservatively recognize explicit ticket IDs like `RBI-023`, `WP-0106`, `ABC-123` (conceptually `\b[A-Z][A-Z0-9]+-\d+\b`). Never infer a ticket from semantic similarity.

---

# 4. Evidence model

For every WorkItem separate:

### `periodEvidence`
Events inside the exact requested interval attributable to the selected user, plus material state changes to that user's active work that affect the summary.

Examples: authored commit, submitted review, authored comment, PR opened, authored PR merged, requested changes received, material CI state change.

### `contextEvidence`
Information used to understand the work but not attributable as work performed in the interval: PR title/body, earlier commits, linked issue, complete file list, current state, earlier discussion, etc.

Example: implementation Tuesday, merge Friday. Friday's log may say **"PR merged Friday; implementation was primarily earlier in the week"**. It must not claim Friday implementation without Friday implementation evidence.

A merge/review/check by someone else is a status/outcome event, not an action by the selected user.

---

# 5. Collect candidates

Use **candidate -> expand -> exact-filter -> group**. Do not crawl every accessible repo.

Set `$GH_USER`, `$SEARCH_START_DATE`, `$SEARCH_END_DATE` conceptually from the invocation. Use `$GH_USER`, never `$USER`: the shell already defines `$USER` as the local OS username, which silently substitutes the wrong login.

Pass search qualifiers as flags, not as a quoted query string: `gh search` treats a quoted positional argument as one literal search term, so `gh search prs "author:x updated:..."` fails with `Invalid search query`.

## Authored commits

```bash
gh search commits \
  --author "$GH_USER" \
  --author-date "$SEARCH_START_DATE..$SEARCH_END_DATE" \
  --limit 1000 \
  --json sha,commit,repository,url
```

With repo scope, add:

```text
--repo owner/repo
```

Prefer commit author over committer for attribution.

Commit search only indexes each repository's default branch. Commits on unmerged feature branches surface through PR expansion (`gh pr view --json commits`), so treat PR expansion as the source for in-progress commits; branch commits with no PR are undiscoverable by search.

## Authored PR candidates

```bash
gh search prs \
  --author "$GH_USER" \
  --updated "$SEARCH_START_DATE..$SEARCH_END_DATE" \
  --limit 1000 \
  --json number,title,state,isDraft,createdAt,updatedAt,closedAt,repository,url
```

Candidate only: another actor may have caused `updatedAt`.

## Reviewed PR candidates

```bash
gh search prs \
  --reviewed-by "$GH_USER" \
  --updated "$SEARCH_START_DATE..$SEARCH_END_DATE" \
  --limit 1000 \
  --json number,title,state,isDraft,createdAt,updatedAt,closedAt,repository,url
```

Later keep only reviews authored by `$GH_USER` with `submittedAt` inside the exact period.

## PRs/issues commented on by user

```bash
gh search issues \
  --include-prs \
  --commenter "$GH_USER" \
  --updated "$SEARCH_START_DATE..$SEARCH_END_DATE" \
  --limit 1000 \
  --json number,title,state,isPullRequest,createdAt,updatedAt,closedAt,repository,url
```

Later filter comments by exact author and timestamp.

## Issue candidates

When issues are first-class work objects:

```bash
gh search issues \
  --author "$GH_USER" \
  --updated "$SEARCH_START_DATE..$SEARCH_END_DATE" \
  --limit 1000 \
  --json number,title,state,createdAt,updatedAt,closedAt,repository,url
```

If a query returns exactly the limit and truncation is plausible, narrow by repository or warn that the result may be incomplete. Never silently present a truncated search as exhaustive.

---

# 6. Apply filters early

## `--repo`

Prefer exact `owner/repo`.

For a short name like `backend`, use it only when uniquely resolvable from the current repo/workspace or discovered candidate repos. Do not assume the authenticated user's namespace. If still ambiguous, report the ambiguity.

## `--ticket`

Treat it as an exact identifier. Search likely evidence directly:

```bash
gh search prs '"RBI-023"' --limit 1000 \
  --json number,title,state,repository,url,updatedAt

gh search issues '"RBI-023"' --include-prs --limit 1000 \
  --json number,title,state,isPullRequest,repository,url,updatedAt

gh search commits '"RBI-023"' --author "$GH_USER" --limit 1000 \
  --json sha,commit,repository,url
```

Add `--repo owner/repo` when supplied. After discovery, require an exact occurrence or structured GitHub link before attaching evidence to that ticket.

---

# 7. Expand relevant PRs

PRs are normally the richest context object.

For daily output, expand every relevant PR. For wide weekly/range candidate sets, expand every PR that anchors a WorkItem — its body is the explanation source for the output, not just classification input — and skip expansion only for PRs that end up as secondary references. Merged state plus `closedAt` from search results is sufficient evidence for a merge outcome, but attribute the merge to a person only after fetching `mergedBy`.

For each PR to expand:

```bash
gh pr view "$PR" --repo "$REPO" \
  --json number,title,body,state,isDraft,author,createdAt,updatedAt,closedAt,mergedAt,mergedBy,headRefName,baseRefName,commits,files,changedFiles,additions,deletions,closingIssuesReferences,reviews,latestReviews,reviewDecision,reviewRequests,comments,statusCheckRollup,mergeStateStatus,mergeable,url
```

Use it to establish:

- intent/title/body;
- commits belonging to PR;
- linked/closing issues;
- branch;
- changed files;
- reviews and current review decision;
- checks;
- merge/current state.

### Full diff

Do not fetch every full diff. First use PR metadata and changed paths. Fetch:

```bash
gh pr diff "$PR" --repo "$REPO"
```

only when it materially improves interpretation: ambiguous intent, classification, important tests/refactor/bugfix details, requested verbose evidence, or code-level blocker analysis.

---

# 8. Expand issues and orphan commits only as needed

For an issue anchoring a WorkItem:

```bash
gh issue view "$ISSUE" --repo "$REPO" \
  --json number,title,body,state,author,assignees,labels,createdAt,updatedAt,closedAt,comments,url
```

A canonical linked issue that remains open prevents ticket-level `completed`, even when one related PR merged.

For each independently discovered commit, check whether it belongs to a PR before creating separate work:

```bash
gh api -H 'Accept: application/vnd.github+json' \
  "repos/$REPO/commits/$SHA/pulls"
```

If associated with a relevant PR, attach it there and do not count it separately.

For a true orphan commit that needs more context:

```bash
gh api "repos/$REPO/commits/$SHA"
```

---

# 9. Build WorkItems

Use this conceptual structure; no JSON output is required:

```text
WorkItem
- repo
- anchor: ticket | issue | pr | unlinked
- id / title
- tickets[] / issues[] / prs[] / commits[] / branches[]
- reviews[] / comments[] / checks[] / changedFiles[]
- periodEvidence[]
- contextEvidence[]
- kind
- lifecycle
- confidence
- outcomes[]
- blockers[]
- followUps[]
- firstActivityAt / lastActivityAt
```

Anchor/group in this priority:

1. structured linked/closing GitHub issue;
2. exact ticket in PR title;
3. exact ticket in PR body;
4. exact ticket in PR branch;
5. exact ticket in commit message;
6. PR;
7. standalone issue;
8. coherent unlinked cluster.

Never use semantic similarity to invent ticket membership.

### Grouping

- many commits in one PR -> one PR/ticket WorkItem;
- many PRs explicitly linked to one ticket -> one ticket WorkItem with sub-deliveries;
- one PR with several tickets -> do not duplicate the whole PR under every ticket; split only where evidence supports a clean split;
- orphan commits -> cluster only when same repo, close in time, and clearly one technical change.

---

# 10. Deduplicate

Canonical identities:

```text
commit = repo + SHA
PR     = repo + PR number
issue  = repo + issue number
```

Reviews/comments use GitHub IDs when available.

Required behavior:

- commit found by search and inside PR appears once;
- squash merge commit is an outcome, not a second implementation unit;
- rebase/force-push must not create extra accomplishments;
- PR discovered through author/review/comment searches remains one PR;
- ticket referenced in several places remains one ticket anchor;
- identifiable cherry-pick/backport is propagation/delivery, not a second original implementation.

Never sum per-commit diffs to estimate total work; the PR net diff serves as context only.

---

# 11. Exact attribution

After expansion, keep events in `periodEvidence` only when exact actor/timestamp support it.

- **commit**: selected user is author and timestamp lies in period;
- **review**: `review.author.login == GH_USER` and `submittedAt` lies in period;
- **comment**: selected user authored it and creation time lies in period;
- **PR merge**: authored PR merged in period -> relevant outcome; say user merged it only if `mergedBy.login == GH_USER`;
- **external changes requested**: relevant status change on selected user's PR, but attribute feedback to reviewer;
- **checks**: state evidence, never human activity. Mention only when validation/blocking materially matters.

Drop candidates with neither attributable activity nor a material status transition relevant to the user's active work.

An open PR should not reappear every day merely because it remains open.

---

# 12. Classify work and state

Work kind, when useful:

```text
feature | bugfix | refactor | tests | docs | investigation | ci | review | maintenance | unknown
```

`investigation` and `review` are work kinds, not lifecycle states. `follow-up required` belongs under follow-ups.

Lifecycle:

```text
completed | in progress | in review | blocked | unknown
```

Confidence:

```text
confirmed | strong | inferred
```

## State rules

### Ticket-anchored

- canonical linked issue closed -> `completed · confirmed`;
- ticket open + active non-draft PR with review activity/request -> `in review · strong`;
- ticket open + attributable implementation/draft PR -> `in progress · strong`;
- one merged PR does **not** complete an open multi-PR ticket.

### PR-anchored

- `mergedAt != null` -> `completed · confirmed`;
- open draft -> `in progress · strong`;
- open non-draft + review requested/review activity -> `in review · strong`;
- open non-draft + fresh implementation only -> `in progress · strong`;
- closed without merge -> do not call completed; report `closed without merge` / `unknown`.

### Commit-only

- attributable commits without completion evidence -> `in progress`;
- commits observed on the default branch may be reported as `landed on default branch` — delivery is observable — without claiming the ticket or workstream is complete;
- words like `done`, `final`, or `fixed` in a commit message do not prove completion.

### Blocked

Use only with direct evidence: explicit dependency/access/decision wait, unresolved required changes, required failing checks preventing progression, or equivalent structured blocking state. Do not infer blockage from inactivity or one optional failed check.

---

# 13. Interpret changed files conservatively

Useful hints:

```text
tests/**, **/*.test.*, **/*.spec.*, __tests__/** -> tests
docs/**, README*, CHANGELOG*, *.md                -> docs
.github/workflows/**, Dockerfile*, vercel.json    -> CI/deployment/config
```

Use paths plus PR/diff context before stating a technical outcome.

Generated/vendor/lock files may matter, but their volume does not indicate effort. Examples: lockfiles, `vendor/**`, `dist/**`, `build/**`, `generated/**`.

Poor commit messages (`fix`, `wip`, `oops`, `final`) are low-priority evidence. Prefer linked issue -> PR title/body -> diff/files -> reviews/comments/checks -> commit messages.

---

# 14. Blockers, follow-ups, likely next

A blocker must explain what is currently preventing progress and must have evidence.

A follow-up/likely-next item must be supported by one of:

- explicit TODO/checklist;
- linked follow-up issue;
- unresolved requested changes;
- explicit remaining work;
- explicit next action written by the selected user.

Do not manufacture a `Today` plan from open tickets. Prefer `Likely next` when evidence suggests the next action but does not prove a committed plan. Omit the section if unsupported.

---

# 15. Reviews are first-class work

A valid worklog may contain zero commits and substantial review work.

Good:

```text
PR #147 — Review
Completed review
- Reviewed deployment changes.
- Requested changes around runtime asset tracing.
```

Do not rewrite this as "fixed runtime asset tracing" unless the selected user actually authored that fix.

Fetch inline review comments only when their detail materially improves the summary; do not expand all review threads by default.

---

# 16. Manual and invisible work

`--note` is self-reported context, not GitHub evidence:

```text
Additional context
- Investigated migration options for the report schema. (manual note)
```

GitHub may miss meetings, design, debugging without commits, research, coordination, pairing, architecture decisions, and work in Jira/Linear/Slack/Docs/Figma/Vercel/cloud consoles/etc.

Therefore a zero-evidence result means only:

```text
No attributable GitHub activity was found for the selected period.
```

---

# 17. Output

Rules for every range:

- Date events at day granularity using bare dates (`Aug 6`), not weekday names — weekday labels resurface weekend patterns; exact clock times exist to decide period membership, not for output.
- Time-of-day, late-night, and weekend patterns describe the person, not the work — leave them out.
- A ticket spanning several repos appears once, anchored to the ticket and naming the repos inside it, not once per repo.
- Unfavorable outcomes the evidence records — failures, reverts, adverse findings, unresolved counts — survive summarization with the same weight as wins.

## Daily (`today`, `yesterday`)

Optimize for reading before standup in under one minute. Group repo -> WorkItem. Prefer 1-3 outcome bullets.

```text
## Yesterday

### report-builder-integration

RBI-026 — PDF deployment
Completed
- Fixed Chromium packaging for Vercel.
- Added deployment regression coverage.
- PR #123 merged.

RBI-027 — Translation tables
In review
- Standardized repeated heading terminology.
- PR #126 is open for review.

### Reviews
- Reviewed PR #147 and requested changes around runtime tracing.

### Blockers
- RBI-028: production verification is waiting on Vercel access. (inferred from PR discussion)

### Likely next
- Address requested changes on RBI-027.
```

Omit empty `Reviews`, `Blockers`, `Follow-ups`, and `Likely next` sections.

## Weekly/range

Group by repo and WorkItem; emphasize progression/outcomes, not chronology or counts.

Explain, don't enumerate. For each WorkItem, drawing only on collected evidence:

- one or two plain-language sentences on what the deliverable is and does, from the PR body or linked issue — a reader outside the project should understand the work without opening a link;
- the why, when the body states one: the defect, measurement, or decision that drove the change;
- structural facts that change what an outcome means: a base branch that is not the default (merged = integrated, not shipped), feature gates or dark launches, diff noise the author flags as generated churn;
- ticket IDs and PR numbers as anchors inside the sentence, never as the whole sentence.

```text
## Week · Aug 3–7

### report-builder-integration

RBI-026 — PDF deployment
Completed
- Server-side PDF rendering now survives deployment: Chromium's runtime assets were
  silently dropped from the Vercel bundle, so tracing configuration pins them and a
  regression test fails if they vanish again (PR #123, merged into the integration
  branch — integrated, not yet shipped to main).

RBI-027 — Translation tables
In progress
- Standardized the shared heading terminology that report tables repeat across
  languages, and updated the tables to consume it; Scope 3 terminology remains open.

### Reviews / collaboration
- Reviewed PR #147 and identified a deployment configuration problem.

### Open follow-ups
- RBI-031 — Chromium packaging hardening.
```

## `--verbose`

Append compact provenance, not a dashboard:

```text
Evidence
- PR #123 · merged Aug 6
- Commits: abc123, def456
- Principal files: src/pdf.ts, vercel.json, tests/pdf.test.ts
- Tests: tests/pdf.test.ts
- Checks: build passed; deployment preview passed
- Linked issue: RBI-026
- Diff context: 8 files · +143 / -61
```

LOC/counts may appear only as context here.

---

# 18. Edge cases

- **Squash merge:** PR is canonical; squash commit is merge outcome.
- **Rebase/force-push:** rewritten SHAs are not additional work units.
- **PR open multiple days:** include only with period activity or meaningful status transition.
- **Commits outside period + merge inside:** report merge; earlier implementation is context.
- **Several branches/PRs one ticket:** group when explicit linkage exists.
- **Several tickets one PR:** avoid duplicated accomplishment; use shared PR/ticket item if split is uncertain.
- **Bots:** exclude bot-authored activity by default; bot/check output may be contextual evidence.
- **Generated files:** ignore volume, not necessarily semantic relevance.
- **Cherry-pick/backport:** describe propagation/delivery when identifiable.
- **Revert:** report both delivery and reversal; do not call it unqualified success.
- **Multiple repos:** group by repository; never duplicate the same event.
- **No ticket:** PR title is a valid WorkItem; if no PR, use a conservative unlinked cluster.

---

# 19. Execution algorithm

Follow this order:

1. Resolve `GH_USER`, timezone, period (including widened search dates), repo/ticket filters, verbosity, note.
2. Run preflight/auth checks.
3. Discover candidate commits, authored PRs, reviews, comments, and issues.
4. Narrow repository set from candidates or explicit filters.
5. Expand relevant PRs; expand issues/orphan commits only as needed.
6. Use exact actor + timestamp to split `periodEvidence` from `contextEvidence`.
7. Associate commits -> PRs and PRs -> linked issues/tickets.
8. Deduplicate canonical entities.
9. Build WorkItems using `linked issue/ticket > PR > issue > unlinked cluster`.
10. Determine lifecycle/confidence from structured state before text interpretation.
11. Extract only evidence-supported outcomes, blockers, and follow-ups.
12. If needed, inspect a focused diff to improve technical wording.
13. Generate concise daily/weekly output from WorkItems, not raw events.
14. Run the quality checklist below before returning.

---

# 20. Quality checklist

Before answering, verify:

- correct GitHub login and time interval;
- exact timestamp filtering after broad search;
- no unrelated repos/bots;
- no commit/PR/ticket double counting;
- no semantic ticket guesses;
- open ticket not marked completed merely because one PR merged;
- implementation outside period not attributed to period;
- reviews and implementation attributed to correct people;
- blockers and likely-next items have evidence;
- no LOC/commit/PR productivity interpretation;
- no ranking/score;
- no time-of-day or work-rhythm observations;
- evidence-recorded failures, reverts, and adverse findings not dropped;
- claims are traceable to collected GitHub evidence;
- normal daily output is readable in under one minute.

If evidence conflicts, prefer structured current GitHub state over stale prose and mention the conflict when material.

If permissions, rate limits, ambiguous repo resolution, or search truncation prevent completeness, return the useful evidence collected but explicitly label the result **partial**.

---

# Core principle

**Measure nothing about the person. Reconstruct the work the evidence can support, group it into meaningful units, and summarize only what matters.**
