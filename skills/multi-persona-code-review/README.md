# multi-persona-code-review

A Claude Code skill that orchestrates a panel of specialist reviewer personas against the project you're working in. Each persona runs in its own sub-agent (in parallel), then a senior-engineer persona synthesizes the findings into a single prioritized, actionable report.

## What it does

When invoked, the skill:

1. **Auto-detects which personas apply** based on signals in your codebase. WordPress, WooCommerce, frontend, and DevOps personas activate conditionally; security, performance, and QA always run.
2. **Spawns parallel sub-agents** — one per persona — each adopting a strict reviewer identity in a fresh context. Independence is the point: parallel runs avoid the anchor-bias that you'd get if one persona's findings were fed to the next.
3. **Synthesizes** all raw findings via a senior-engineer sub-agent that dedupes, re-rates severity on a cross-domain rubric, ranks by impact × effort, and surfaces decisions only the user can make.
4. **Writes everything to disk** under `.reviews/{YYYY-MM-DD-HHMM}/` in your project root.
5. **Offers follow-ups** — render an HTML report, file GitHub issues, run a Stage 2 targeted debate on contested findings.

## When to use

- Pre-launch reviews of plugins, themes, or apps.
- Post-incident audits.
- Due diligence on a codebase you've inherited.
- "Get a few sets of eyes on this" — when the developer wants more than a single-lens review.

## When NOT to use

- Targeted bug hunts in a single file (use a direct read).
- Greenfield design reviews where there is no code yet.
- "Just a security review" — run a single security audit instead; the orchestration overhead isn't worth it.
- Tiny codebases (one file, a snippet) — a panel of personas is theater.

## Architecture

Three stages:

| Stage | Purpose | Runs |
|---|---|---|
| **1. Parallel Review** | Each persona reviews the codebase in isolation and files raw findings. | Automatically, in parallel. |
| **2. Targeted Debate** *(optional)* | Resolves contested findings (wide severity disagreement, contradictory fixes) through a structured single-round exchange. | Only on request, after Stage 3 flags `needs-debate.md`. |
| **3. Synthesis** | Senior-engineer persona dedupes, re-rates severity cross-domain, ranks, and writes the final report. | Automatically, after all Stage 1 sub-agents return. |

See `SKILL.md` for the full workflow and `stages/` for the per-stage sub-agent prompts.

## How to invoke

The skill auto-activates on phrases like:

- "Do a code review of this project"
- "Run a multi-persona review"
- "Audit this codebase for security and performance"
- "Get a few sets of eyes on this plugin"
- "What would a senior engineer say about this?"

You can also be explicit about persona selection:

- "Run a code review with all personas"
- "Multi-persona review but skip woocommerce"
- "Run only security and qa"

## Output

A run directory under `{project-root}/.reviews/{YYYY-MM-DD-HHMM}/`:

```
.reviews/2025-01-14-1530/
├── raw/                     # Persona sub-agents' raw findings (pre-synthesis)
│   ├── security-engineer/
│   │   ├── SEC-001.md
│   │   ├── SEC-002.md
│   │   └── _summary.md
│   ├── performance-engineer/
│   ├── wordpress-developer/
│   └── ...
├── findings/                # Final, deduplicated, re-rated findings
│   ├── CRIT-001.md
│   ├── CRIT-002.md
│   ├── HIGH-001.md
│   └── ...
├── summary.md               # Executive overview + counts + top priorities
├── decisions-needed.md      # Choices only the team can make
├── needs-debate.md          # Contested findings (optional)
└── nice-to-have.md          # Filtered noise (optional)
```

## Layout

```
multi-persona-code-review/
├── SKILL.md                 # Frontmatter + workflow + orchestration logic
├── README.md                # This file
├── personas/
│   ├── 00-senior-engineer.md          # Synthesizer
│   ├── 01-security-engineer.md
│   ├── 02-performance-engineer.md
│   ├── 03-wordpress-developer.md
│   ├── 04-woocommerce-specialist.md
│   ├── 05-frontend-ux-engineer.md
│   ├── 06-devops-engineer.md
│   └── 07-qa-engineer.md
├── stages/
│   ├── 1-parallel-review.md           # Sub-agent prompt for each persona
│   ├── 2-debate.md                    # Optional contested-finding debate
│   └── 3-synthesis.md                 # Senior-engineer synthesizer prompt
└── templates/
    ├── finding.md                     # The strict finding schema
    ├── render-html-report.md          # Spec for HTML report follow-up
    └── render-github-issues.md        # Spec for gh-issue-creation follow-up
```

## Adding a new persona

1. Copy an existing persona file in `personas/` as a template. Number it sequentially (e.g., `08-data-engineer.md`).
2. Fill in the required sections: Identity, Lens, Primary categories, Severity rubric, What you ignore, Output format.
3. Add an activation rule in `SKILL.md` under the "Auto-detection" table — either always-on or signal-based.
4. Add the persona's raw ID prefix to `templates/finding.md` (the ID conventions section).
5. If the persona has rendering implications (a new category that doesn't fit existing labels), update `templates/render-github-issues.md`'s label list.

A persona file should stand alone: a fresh sub-agent should be able to adopt the persona and start reviewing with only that file and `templates/finding.md` for reference.

## Cost note

A full multi-persona run is **not cheap**. Each Stage 1 sub-agent does a substantive read of the codebase; the synthesis sub-agent reads every raw finding. On a medium-sized codebase, expect a six-persona run to consume considerably more tokens than a single audit.

Recommendations:

- For routine sanity checks, invoke a single persona ("run a security review on this") rather than the full panel.
- Reserve the full panel for pre-launch, post-incident, due-diligence, or inherited-codebase situations.
- Use the Stage 2 debate selectively — only on findings where the disagreement actually matters.

## Manual-invocation fallback

If you're not running Claude Code (e.g., using the Anthropic API directly, or a different agent harness):

1. For each persona in `personas/`, run it as a separate Claude API conversation against the codebase. Inline the persona file, `stages/1-parallel-review.md`, and `templates/finding.md` into the system prompt.
2. Save the persona's findings to `.reviews/{timestamp}/raw/{persona-slug}/` using the schema in `templates/finding.md`.
3. After all personas are done, run a final conversation with `personas/00-senior-engineer.md` + `stages/3-synthesis.md` as the system prompt, with all raw findings as input. Save its output to `.reviews/{timestamp}/findings/` and the summary files.

The structure is identical to the auto-orchestrated version; only the orchestration changes.
