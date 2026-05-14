---
name: multi-persona-code-review
description: Orchestrates a multi-persona code review of the user's current project by spawning parallel sub-agents — each adopting a different reviewer persona (security, performance, WordPress, WooCommerce, frontend/UX, devops, QA) — then synthesizing their findings into a single prioritized, actionable report with severity ratings, deduplicated issues, and decisions-needed flagged for the user. Use whenever the user asks for a code review, code audit, multi-persona review, panel review, second opinion on a codebase, "review this project", "audit this repo", a security + performance + UX pass, or names specific reviewer personas. Trigger on loose phrasing too ("get a few sets of eyes on this", "review my plugin/theme for production", "what would a senior engineer say about this"). The skill auto-detects which personas to run (always runs security/performance/QA; adds WordPress, WooCommerce, frontend/UX, and devops based on signals in the code), writes raw findings and a synthesized report under `.reviews/{timestamp}/` in the project root, and offers follow-ups to render an HTML report or file GitHub issues. Not for single-file targeted bug hunts; not for greenfield design reviews where there is no code yet.
---

# Multi-Persona Code Review

Orchestrate a panel of specialist reviewer personas against the user's current project, in parallel, then synthesize their findings into one prioritized report.

The skill itself is the conductor. Each persona is a separate sub-agent (spawned via the Task tool) that runs in its own fresh context. Independence is the point — parallel sub-agents give genuinely different perspectives instead of one anchor opinion that drags the rest along.

## Core principles

1. **Parallel before synthesis.** Persona reviews run concurrently in fresh sub-agent contexts. Never feed one persona's findings to another before they have produced theirs.
2. **Evidence-bound.** Every finding cites a file and line range. Uncertain findings are marked `confidence: low`; speculation without evidence is dropped.
3. **One issue = one finding.** Stage 1 may surface the same underlying issue from multiple personas; Stage 3 merges them into a single finding with `reviewers:` listing each contributor.
4. **Severity is cross-domain in the final report.** Specialists rate within their own rubric. The senior-engineer synthesizer re-rates using a unified rubric — the specialist rating is preserved in the finding's notes, not the final severity.
5. **Decisions belong to the user.** When the review surfaces a genuine architectural or product trade-off, it goes in `decisions-needed.md` instead of being silently resolved.

## When NOT to trigger

- User wants a targeted bug hunt in one file or one function (use a direct read instead).
- User wants a greenfield architecture design (this skill reviews existing code).
- User wants only a security review and explicitly says "just security" (run a single persona — this skill's overhead isn't worth it).
- The codebase is small enough (one file, a snippet) that a panel of personas is theater.

If unsure, ask. A full multi-persona run uses a non-trivial amount of tokens; don't fire it speculatively.

## What "the project" means

The project under review is the **current working directory** at the time the skill is invoked — typically a git repository root. The skill's own location (project-level `.claude/skills/`, user-level `~/.claude/skills/`, or installed as a plugin) is irrelevant to where output goes. All internal references in this skill use paths relative to `SKILL.md`.

Output always lands under `${PROJECT_ROOT}/.reviews/{YYYY-MM-DD-HHMM}/`, where `${PROJECT_ROOT}` is the user's current working directory at invocation time. Create `.reviews/` if it doesn't exist and add it to `.gitignore` if the user wants (ask once, then remember within the run).

## Architecture

Three stages. Stage 2 is optional and conditional.

### Stage 1 — Parallel review

For each persona selected for this run, spawn one sub-agent via the Task tool. Each sub-agent receives:

- The persona file (`personas/NN-<slug>.md`) verbatim
- The Stage 1 instructions (`stages/1-parallel-review.md`)
- The finding schema (`templates/finding.md`)
- The output directory for this run
- The project root path
- Any user-supplied scope hints

Sub-agents are spawned **in a single message with multiple Task tool calls** so they actually run concurrently. Each writes its findings to `${RUN_DIR}/raw/{persona-slug}/` — one file per finding plus a `_summary.md` for its own coverage notes.

Do not pass one persona's draft findings to another at this stage.

### Stage 2 — Targeted debate (optional)

After synthesis (Stage 3) flags `needs-debate.md` entries, the user (or a follow-up invocation) can request Stage 2 for a specific finding. A single sub-agent loads the disputed finding plus the contributing personas' source files and plays each role through one structured round (opening → response → synthesis attempt → recommendation). Output is either an updated final finding or an escalation to `decisions-needed.md`. See `stages/2-debate.md`.

Stage 2 does not run automatically — it is a follow-up. The first invocation reports back with what's in `needs-debate.md` and waits.

### Stage 3 — Synthesis

After every Stage 1 sub-agent has returned, spawn one final sub-agent as the senior-engineer persona (`personas/00-senior-engineer.md`) with the Stage 3 instructions (`stages/3-synthesis.md`). It reads everything under `raw/`, dedupes, re-rates severity using the cross-domain rubric, ranks, and produces:

- `findings/{ID}.md` — one file per final finding, IDs prefixed by severity (`CRIT-001`, `HIGH-001`, …)
- `summary.md` — executive overview, counts, top priorities, cross-cutting themes, effort estimate, coverage notes
- `decisions-needed.md` — questions only the user can answer, each with options and a recommendation
- `needs-debate.md` — present only if there are contested findings worth a Stage 2 pass
- `nice-to-have.md` — present only if there is noise worth recording but not promoting

## Workflow when invoked

Walk through these steps in order.

### 1. Confirm scope and select personas

Identify the project root (usually `pwd`, but if the user invoked from inside a sub-directory, walk up to the nearest git root or ask). Then determine which personas apply.

**Always include:**

- `security-engineer`
- `performance-engineer`
- `qa-engineer`

**Conditionally add** based on signals in the codebase. Detect with shell tools (`fd`, `rg`, `grep`, `ls`) — don't rely on intuition.

| Persona | Add when (any signal matches) |
|---|---|
| `wordpress-developer` | `wp-config.php`, `wp-content/`, `wp-load.php`, `functions.php`, or content matches `rg -l 'add_action\(\|add_filter\(\|wp_[a-z_]+\('` |
| `woocommerce-specialist` | `woocommerce.php` anywhere, any path containing `woocommerce`, or content matches `rg -l 'WC\(\)\|wc_get_\|class WC_\|\$order->\|\$cart->'` |
| `frontend-ux-engineer` | Significant template/component files (`*.tsx`, `*.jsx`, `*.vue`, `*.svelte`, template parts in a theme, block sources), or substantial CSS, or `register_block_type` calls |
| `devops-engineer` | `Dockerfile`, `docker-compose*`, `.env*`, `.github/workflows/`, `Makefile`, `Procfile`, `*.tf`, `ansible/`, `k8s/`, `helm/`, or `composer.json` / `package.json` with deploy scripts |

The user can override:

- `all` — run every persona regardless of signals
- Explicit list — e.g. "security and qa only", or "skip wordpress" — honor it and skip detection for excluded ones.

State the chosen persona list back to the user before spawning sub-agents. Don't ask for approval, just state — they'll interject if it's wrong.

### 2. Create the run directory

```
${PROJECT_ROOT}/.reviews/{YYYY-MM-DD-HHMM}/
├── raw/                      # one subdir per persona, created by Stage 1
├── findings/                 # final, deduplicated findings (Stage 3)
├── summary.md                # Stage 3
├── decisions-needed.md       # Stage 3
├── needs-debate.md           # Stage 3, only if contested findings exist
└── nice-to-have.md           # Stage 3, only if there is noise worth recording
```

Use the local timezone for the timestamp. If a run started this minute already exists, append a `-2` suffix rather than overwriting.

### 3. Run Stage 1 in parallel

Spawn one sub-agent per selected persona — **all Task tool calls in a single message** so they execute concurrently. Each prompt should include:

- The full persona file content (read it from `personas/{NN-slug}.md` relative to this SKILL.md)
- The full Stage 1 instructions (`stages/1-parallel-review.md`)
- The finding schema (`templates/finding.md`)
- The project root absolute path
- The output path: `${RUN_DIR}/raw/{persona-slug}/`
- Any scope hints from the user ("focus on the checkout flow", "the new REST controller", etc.)

The sub-agent must be told explicitly: write each finding as a separate markdown file, and write a `_summary.md` with its coverage notes (scope, finding count by severity, areas focused, areas skipped, confidence, open questions).

Wait for all sub-agents to return.

### 4. Run Stage 3 synthesis

Spawn one sub-agent with the senior-engineer persona and the Stage 3 instructions. It reads everything under `${RUN_DIR}/raw/`, produces the deliverables described above, and returns a short summary.

### 5. Report back to the user

A short message with:

- Total findings by severity (Critical/High/Medium/Low/Info)
- Top 3–5 priorities (IDs and one-line titles)
- Whether `decisions-needed.md` has entries (and how many)
- Whether `needs-debate.md` has entries
- Pointers to the run directory and to `summary.md`
- Offer follow-ups: render HTML report, file GitHub issues, run Stage 2 debate on contested findings

Do not paste the full report into chat. The artifacts on disk are the deliverable.

## Follow-up actions

After the initial run, the user may ask for any of these. The skill handles them as follow-ups; the run directory is the input.

- **Render HTML report.** Generate a self-contained `report.html` in the run directory. See `templates/render-html-report.md`.
- **File GitHub issues.** Generate (and optionally execute) a `gh issue create` bash script. See `templates/render-github-issues.md`.
- **Run Stage 2 debate.** Pick a finding from `needs-debate.md` (or one the user names). See `stages/2-debate.md`.

If the user has the `github-issues` skill installed (this repo ships it), the issue-creation step can hand off to it — pass the individual finding markdown files as input.

## Manual-invocation fallback (no Claude Code)

If the user is not running Claude Code (e.g. using the Anthropic API directly, or a different agent harness), they can still use this skill manually:

1. Read each `personas/*.md` file and run it as a separate Claude API conversation against the codebase.
2. Save findings to disk using the schema in `templates/finding.md`.
3. Run `personas/00-senior-engineer.md` + `stages/3-synthesis.md` as the final synthesis pass with the raw findings as input.

The structure is the same; only the orchestration changes.

## Cost note

A full multi-persona run is not cheap. Each Stage 1 sub-agent does a substantive read of the codebase; the synthesis sub-agent reads every raw finding. On a medium-sized WordPress codebase, expect a six-persona run to consume significantly more tokens than a single audit. Recommend single-persona invocations for routine sanity checks; reserve the full panel for pre-launch, post-incident, due diligence, or "we inherited this codebase" situations.
