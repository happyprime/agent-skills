---
name: github-issues
description: Turns one or more raw items (bugs, feature ideas, TODOs, call notes, bullet lists) into well-structured GitHub issues using the `gh` CLI — grounding descriptions in actual repo code where possible, classifying each as bug vs feature, and applying labels that exist on the target repo. Files issues directly without per-item approval; the user reviews in GitHub afterward. Use this skill whenever the user asks to "create issues", "file tickets", "open GitHub issues", "turn these into issues", or hands over a list of things that clearly belong in a tracker. Trigger even when the phrasing is loose ("can you write these up as issues", "file these on the repo", "log this as a bug") as long as GitHub issues are the destination.
---

# GitHub Issues

Turn raw items into well-written GitHub issues via `gh`, filing them directly. The user reviews in GitHub afterward, not before.

## Core principles

1. **Best-effort comprehensive, not cautious.** The user expects to hand over a list and get issues filed. Don't stall for per-item approval. If something is ambiguous, make a reasonable call and note the ambiguity in the issue body — the user will refine in GitHub.
2. **Ground descriptions in code when possible.** An issue that says "sanitize input in `includes/form.php:42`" is far more actionable than "fix sanitization somewhere". Cite file paths and line numbers when you can find them. If you can't locate the code, say so in the body (e.g., "Could not locate in current codebase — author to confirm") rather than fabricating references.
3. **Match the repo's existing conventions.** Before drafting, glance at a few recent issues to learn the house style (title casing, checklist usage, how labels are applied). A filed issue should feel like a regular contributor wrote it.
4. **One item = one issue.** If the user hands over five bullets, file five issues. Group only when items are unmistakably the same problem. When in doubt, split — merging later in GitHub is trivial; splitting is annoying.

## When NOT to trigger

- User asks to *read*, *comment on*, or *close* existing issues — straight `gh` calls, no skill needed.
- User wants a PR filed (`gh pr create`).
- User is explicitly brainstorming ("what would an issue for this look like?") and hasn't said to file yet.

## Workflow

Three phases: **Scope → Recon → Draft & File.**

### Phase 1 — Scope

Before touching the repo, confirm two things:

1. **Which repo?** If the user named one (e.g., "on happyprime/skills"), use it. If not, ask — don't assume from the current working directory, because users often want issues filed on a *different* repo than the one they're in. Once you have an `owner/repo`, verify access and that issues are enabled:

   ```bash
   gh repo view <owner/repo> --json name,hasIssuesEnabled
   ```

   If issues are disabled, stop and report.

2. **Which items?** Parse the user's input into a numbered list and echo it back briefly:

   ```
   I read 4 items to file on happyprime/skills:
   1. Login form throws 500 when email contains "+"
   2. Add dark mode toggle to dashboard
   3. Refactor the issue-creator retry logic
   4. Docs: document the new /status endpoint

   Proceeding.
   ```

   This is a quick sanity check, not an approval gate — proceed without waiting unless the user interjects.

### Phase 2 — Recon

Gather what's needed to write specific, actionable issues.

**Repo metadata (once, cached across items):**

```bash
# Available labels — needed to assign sensibly
gh label list --repo <owner/repo> --limit 200 --json name,description,color

# Recent issues — useful for matching house style and detecting dupes
gh issue list --repo <owner/repo> --limit 10 --state all --json title,labels,body

# Issue templates, if any
gh api "repos/<owner/repo>/contents/.github/ISSUE_TEMPLATE" 2>/dev/null
```

**Per-item code grounding (best effort):**

If the item references a file, function, behavior, or UI element, try to locate it. Use `Grep` and `Glob` on the local repo if it's the current working directory (faster and more precise than GitHub search). If it's a different repo:
- For quick lookups: `gh search code --repo <owner/repo> "<query>"`
- If deeper reading is needed: skip the grounding rather than cloning — the user said grounding is best-effort, not required.

For each item, capture what you find:
- File path(s) and line number(s) of relevant code
- A short code snippet (2–5 lines) if citing it clarifies the issue
- Related existing issues via `gh issue list --repo <owner/repo> --search "<keywords>"` — if there's an obvious duplicate, note it in the new issue's body ("Possibly related to #N") but still file; the user will merge in GitHub if needed.

If the item is abstract or you can't find anything, move on. Don't invent references.

### Phase 3 — Draft & File

For each item, classify, draft, and file in one pass. No batch approval step.

**Classify as bug, feature, or task:**
- **Bug**: broken, incorrect, or unexpected behavior of something that exists. Verbs like "breaks", "fails", "throws", "returns wrong".
- **Feature**: something that doesn't exist yet, or an enhancement. Verbs like "add", "support", "allow".
- **Task / chore**: maintenance, refactoring, docs, CI, infra. Treat as feature for template purposes unless the repo has a dedicated `task`/`chore` label.

Ambiguous items: pick the better-fitting type and note the call in the issue body if it's a real toss-up.

**Pick labels** from the repo's actual label list (from recon). Match on semantics, not exact string:
- Bug-ish items → `bug`, `defect`, or repo's equivalent
- Feature-ish items → `enhancement`, `feature`, `improvement`
- Docs items → `documentation`, `docs`
- Area labels (e.g., `area: auth`, `frontend`, `ci`) → match based on file paths from recon

Never invent labels. If nothing fits, file with no labels — better empty than wrong.

**Draft the body** using the template for the item's type.

#### Bug template

```markdown
## Summary
<One-sentence description of what's broken.>

## Current behavior
<What happens today. Include error messages or observed output if the user provided them.>

## Expected behavior
<What should happen instead.>

## Where in the code
<File path and line range, with a short snippet if useful. Omit this section entirely if the code isn't known.>

```<language>
// path/to/file.ext:42-48
<snippet>
```

## Steps to reproduce
<If the user gave steps, list them. If not, write "To be confirmed by reporter." — don't invent.>

## Notes
<Optional: related issues ("Possibly related to #N"), hypotheses, or context from the user's original wording that didn't fit above. Include any classification calls you made on ambiguous items here.>
```

#### Feature / task template

```markdown
## Summary
<One-sentence description of the desired capability or change.>

## Motivation
<Why this matters. Pull from the user's original wording — don't manufacture justifications.>

## Proposed behavior
<Concrete enough that someone could scope an implementation.>

## Relevant code
<If there's existing code that would be modified or extended, cite it. Omit if new-from-scratch or not located.>

```<language>
// path/to/file.ext:42-48
<snippet>
```

## Acceptance criteria
- [ ] <Verifiable outcome>
- [ ] <Another one>

## Notes
<Optional: alternatives, open questions, related issues.>
```

**Title guidelines:**
- Bugs: lead with the symptom ("Login form throws 500 when email contains `+`")
- Features/tasks: lead with a verb ("Add dark mode toggle to dashboard")
- Under 80 characters
- Match existing issues' casing style (from recon)
- No trailing period

**File directly:**

```bash
gh issue create \
  --repo <owner/repo> \
  --title "<title>" \
  --body "<body>" \
  --label "<label1>,<label2>"
```

If the body is long, write it to a temp file and use `--body-file` — avoids shell-escaping headaches with backticks and code blocks.

Capture each resulting URL.

### Phase 4 — Report

After filing, report the URLs as a numbered list matching the input order:

```
Filed 4 issues on happyprime/skills:
1. https://github.com/happyprime/skills/issues/123 — Login form throws 500 when email contains "+" [bug]
2. https://github.com/happyprime/skills/issues/124 — Add dark mode toggle to dashboard [enhancement]
3. https://github.com/happyprime/skills/issues/125 — Refactor the issue-creator retry logic [enhancement]
4. https://github.com/happyprime/skills/issues/126 — Document the new /status endpoint [documentation]
```

Include the labels you applied so the user can spot classification errors at a glance when reviewing in GitHub.

## Edge cases

- **Repo uses issue templates** (`.github/ISSUE_TEMPLATE/*.md` or `*.yml`): conform to the template structure rather than the bug/feature templates above. `gh issue create --template <name>` can also be used when the template file has a clear `name:` field.
- **Huge list (20+ items)**: file sequentially; don't parallelize (GitHub rate-limits and out-of-order filing makes the final report list confusing).
- **One item clearly duplicates another in the same batch**: file both but cross-link them in each body's Notes section.
- **Private / enterprise repo behind SSO**: if `gh` returns auth errors, stop and tell the user to run `gh auth login` or `gh auth refresh` themselves.
- **Assignees / milestones / projects**: only set these if the user mentioned them in the original request. Don't guess.
- **A single filing fails mid-batch**: report what succeeded, what failed, and the error. Don't silently retry or skip.
