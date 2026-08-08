# Worklog

`worklog` is an evidence-based GitHub skill for reconstructing what a developer worked on during a given period and turning it into a concise daily standup note or an explanatory weekly/range summary.

It is designed for standups, retrospectives, and personal/team work tracking — and explicitly **not** for turning GitHub activity into a productivity score.

The skill reconstructs:

```text
GitHub activity -> evidence -> WorkItem -> state -> outcome -> blocker/follow-up
```

A **WorkItem** is anchored to a ticket or linked issue when possible, falling back to a PR, a standalone issue, or a coherent cluster of unlinked commits. Commits, diffs, changed files, checks, reviews, and comments are supporting evidence — never productivity units.

## What the output looks like

**Daily** (`today`, `yesterday`): terse, grouped repo → WorkItem, readable before standup in under a minute.

**Weekly/range** (`week`, `last 7 days`, `2026-08-01..2026-08-07`, `N days`): explanatory. Each WorkItem gets one or two plain-language sentences on what the deliverable is and does, the why when the PR body states one, and structural facts that change what an outcome means:

```text
RBI-026 — PDF deployment
Completed
- Server-side PDF rendering now survives deployment: Chromium's runtime assets were
  silently dropped from the Vercel bundle, so tracing configuration pins them and a
  regression test fails if they vanish again (PR #123, merged into the integration
  branch — integrated, not yet shipped to main).
```

A merged PR whose base is a long-lived integration branch is reported as *integrated, not shipped*. A feature merged behind a disabled gate is reported as dark. Diff churn the author flags as generated is not presented as substance.

## Design guarantees

- **No scoring.** Never calculates productivity, velocity, effort, or quality; never ranks developers; never uses commit/PR/LOC counts as proxies.
- **Exact attribution.** Events count only with exact actor + timestamp; a merge is attributed to a person only via `mergedBy`; reviews and fixes are credited to whoever actually authored them.
- **Period vs context.** A PR merging inside the period does not mean implementation happened inside the period; older work is reported as context, never as period activity.
- **Preserved uncertainty.** Zero evidence yields "No attributable GitHub activity was found" — never "no work was done". GitHub cannot see meetings, research, debugging without commits, or work in external tools.
- **Privacy.** Events are dated at day granularity with bare dates; time-of-day, late-night, and weekend patterns are deliberately excluded — they describe the person, not the work.
- **Balance.** Failures, reverts, and adverse findings recorded in the evidence survive summarization with the same weight as wins.

## Requirements

Worklog is intentionally a `SKILL.md`-only skill: no Python, Node, `jq`, `yq`, or companion scripts.

You need:

- an Agent Skills-compatible coding agent with shell access;
- [GitHub CLI (`gh`)](https://cli.github.com/) — commands verified against `gh` 2.96;
- authenticated access to the repositories you want inspected.

Verify access:

```bash
gh --version
gh auth status
gh api user --jq '.login'
```

For private organization repositories, the authenticated account must be able to read them. Organizations using SAML SSO may require authorizing the GitHub CLI credential for that organization.

## Installation

### Claude Code — personal

```bash
git clone https://github.com/jaimeberdejo/worklog-skill /tmp/worklog-skill
mkdir -p ~/.claude/skills/worklog
cp /tmp/worklog-skill/SKILL.md ~/.claude/skills/worklog/SKILL.md
rm -rf /tmp/worklog-skill
```

The folder name must be `worklog` so the skill is invokable as `/worklog`.

### Claude Code — project/team

From the target repository root:

```bash
mkdir -p .claude/skills/worklog
cp /path/to/worklog-skill/SKILL.md .claude/skills/worklog/SKILL.md
git add .claude/skills/worklog/SKILL.md
git commit -m "add worklog skill"
```

### Other Agent Skills-compatible tools

Install `SKILL.md` in the tool's skill directory under a folder named `worklog`. The agent must be able to execute `gh`: GitHub data is collected live, never reconstructed from conversation history.

## Usage

```text
/worklog today
/worklog yesterday
/worklog week              # Monday 00:00 -> now
/worklog last 7 days       # rolling window; "3 days", "5 days", "past week" also accepted
/worklog 2026-08-01..2026-08-07

/worklog today --repo frontend        # short names resolve only when unambiguous
/worklog week --repo owner/backend
/worklog week --ticket RBI-023        # exact identifier, never semantic matching
/worklog week --user @developer       # requires read access to their activity
/worklog yesterday --verbose          # append compact evidence/provenance
/worklog today --note "investigated migration options for X"   # manual context, labeled as such
```

`/productivity` is accepted as an alias, but the result is always called a worklog or work summary.

## Implementation notes

Decisions encoded in `SKILL.md` that are easy to get wrong when modifying it:

- **`$GH_USER`, never `$USER`** — the shell predefines `$USER` as the local OS username, which silently queries the wrong person.
- **Search qualifiers as flags, not quoted query strings** — `gh search prs "author:x updated:..."` fails with `Invalid search query`; `--author`/`--updated`/`--reviewed-by` flags work.
- **±1 day widened search window** — GitHub search date qualifiers are UTC; events near local midnight fall on a different UTC date. Candidates are found with the widened window, then filtered by exact timestamps converted to local time.
- **`--limit 1000` with explicit truncation handling** — a busy week can exceed lower limits; a query returning exactly the limit triggers per-repo narrowing or an explicit "partial" label, never silent truncation.
- **Commit search indexes only each repo's default branch** — unmerged branch work surfaces through PR expansion; branch commits with no PR are undiscoverable by search.
- **Orphan commits are checked against `repos/{repo}/commits/{sha}/pulls`** before being counted as separate work.

## Troubleshooting

- **`gh` not authenticated:** `gh auth login`, then re-verify with `gh auth status`.
- **Private repos missing:** check `gh repo view owner/repository`; authorize SSO if the org requires it.
- **Skill not discovered:** verify the path ends in `skills/worklog/SKILL.md` (folder name matters) and restart the session if the skills directory was created after it started.
- **Work outside the requested day appears attributed to it:** that violates the `periodEvidence` / `contextEvidence` boundary, which is a non-negotiable rule in `SKILL.md` — treat it as a skill bug.

## Repository structure

```text
worklog-skill/
├── LICENSE
├── README.md
└── SKILL.md
```

`SKILL.md` is the complete executable workflow. `README.md` is documentation for humans.

## Privacy and intended use

Worklog helps developers reconstruct and communicate their own work. It is not designed for surveillance, employee ranking, or productivity measurement, and its rules actively resist those uses: no scores, no counts-as-effort, no time-of-day or weekend observations, and uncertainty is preserved rather than papered over.

## License

MIT — see [LICENSE](LICENSE).
