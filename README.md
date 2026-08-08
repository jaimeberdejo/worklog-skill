# Worklog

A `SKILL.md`-only skill for Agent Skills-compatible coding agents (Claude Code and others) that reconstructs what a developer worked on during a period, from live GitHub evidence via `gh`.

Daily output is a terse standup note. Weekly/range output explains each piece of work in plain language — what it is, why it happened, and what its state means (a PR merged into an integration branch is *integrated, not shipped*; a gated feature is *dark*).

It never scores productivity: no rankings, no commit/PR/LOC counts as effort, no time-of-day or weekend observations, exact attribution only, and failures or reverts recorded in the evidence are reported with the same weight as wins. If GitHub shows nothing, the answer is "no attributable GitHub activity" — not "no work was done".

## Usage

```text
/worklog today
/worklog yesterday
/worklog week                          # Monday 00:00 -> now
/worklog last 7 days                   # rolling; "3 days", "past week" also work
/worklog 2026-08-01..2026-08-07

/worklog week --repo owner/backend
/worklog week --ticket RBI-023         # exact ID, never semantic matching
/worklog week --user @developer
/worklog week --lang en                # output language; default is Spanish (es)
/worklog yesterday --verbose           # append evidence/provenance
/worklog today --note "..."            # manual context, labeled as such
```

The default output language is Spanish; override per run with `--lang` or per repo with a `language:` entry in `.worklog.yml`. IDs, repo names, and quoted technical text stay verbatim.

## Installation

Requires the [GitHub CLI](https://cli.github.com/), authenticated (`gh auth status`) with read access to the repos you want inspected.

**Personal (all your Claude Code projects):**

```bash
git clone https://github.com/jaimeberdejo/worklog-skill /tmp/worklog-skill
mkdir -p ~/.claude/skills/worklog
cp /tmp/worklog-skill/SKILL.md ~/.claude/skills/worklog/SKILL.md
rm -rf /tmp/worklog-skill
```

**Project (shared with the repo's contributors):** copy `SKILL.md` to `.claude/skills/worklog/SKILL.md` in the target repo and commit it.

**Other agents:** install `SKILL.md` in the tool's skill directory under a folder named `worklog`.

The folder name must be `worklog` — it makes the skill invokable as `/worklog`. If the skill isn't discovered, restart the agent session.

## License

MIT — see [LICENSE](LICENSE).
