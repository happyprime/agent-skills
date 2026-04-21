---
name: wordpress-audit
description: Performs a systematic, evidence-based code audit of WordPress projects — specifically custom must-use (mu) plugins and custom themes — covering security, performance, readability, and UX. Use this skill whenever the user asks for a code audit, code review, security review, performance review, pre-launch review, or general quality pass on a WordPress codebase, or when they mention reviewing/auditing a WordPress plugin, theme, or mu-plugin. Trigger even when the phrasing is loose ("can you look over this theme", "give this plugin a once-over", "is this production-ready") as long as WordPress is in the picture. Default deliverable is a written report — do not modify project files unless the user explicitly asks for fixes.
---

# WordPress Code Audit

Systematic audit of a WordPress codebase focused on a custom mu-plugin and a custom theme. Produces a severity-ranked report with file:line evidence and recommended fixes.

## Core principles

1. **Evidence over intuition.** Every finding cites a specific file and line. If the issue can't be located in the code, it doesn't go in the report.
2. **Severity before volume.** A short report of real issues beats an exhaustive list of nitpicks. Group findings by severity and lead with what can hurt users, data, or revenue.
3. **Report, don't refactor.** The default mode is read-only analysis. Suggest fixes in prose or short code snippets inside the report. Only edit project files when the user explicitly asks for remediation.
4. **WordPress-idiomatic.** Recommendations should use WordPress APIs and conventions (hooks, `$wpdb->prepare`, `wp_enqueue_*`, capability checks, text domains) rather than generic PHP advice.

## When NOT to trigger

- User wants to build a new plugin/theme from scratch (that's a development task, not an audit)
- User wants to debug a single specific bug (that's targeted troubleshooting)
- User wants a summary of what the code does without any quality judgment

If a user asks for a "quick look" or "opinion" rather than a full audit, offer a lighter-weight walkthrough and confirm before running the whole methodology.

## Workflow

The audit runs in three phases: **Recon → Analysis → Report**. Do them in order. Don't skip recon to jump straight to findings — without a map of the codebase, the analysis will miss whole categories of issues.

### Phase 1 — Recon

Goal: build a mental model of the codebase and inventory the attack surface and hot spots. The analysis phase is only as good as the recon.

Locate the project root. WordPress projects usually have `wp-content/mu-plugins/` and `wp-content/themes/<theme-name>/`. If the user points at a subfolder, confirm which directories are in scope before proceeding.

Walk the codebase and catalog:

- **Entry points.** Main plugin file(s), `functions.php`, any bootstrap/loader files in `mu-plugins/` (remember: WordPress does not auto-load mu-plugins from subdirectories — a top-level loader file is required).
- **Plugin/theme headers.** Name, version, requires-at-least, tested-up-to, text domain, license.
- **Registered hooks.** Every `add_action()` and `add_filter()` call — note the hook name, callback, priority, and accepted args.
- **AJAX handlers.** Every `wp_ajax_*` and `wp_ajax_nopriv_*` action. `nopriv` handlers are public; flag them for extra scrutiny.
- **REST routes.** Every `register_rest_route()` call. Note the `permission_callback` — missing or `__return_true` is a serious finding.
- **Shortcodes and blocks.** `add_shortcode()`, `register_block_type()`.
- **Custom post types, taxonomies, roles, capabilities.**
- **Database interactions.** Every `$wpdb` usage. Raw SQL. `dbDelta()` calls. Custom table creation.
- **User input surfaces.** Forms, `$_GET` / `$_POST` / `$_REQUEST` / `$_COOKIE` / `$_FILES` / `php://input`, settings pages, Customizer controls.
- **Output surfaces.** `echo`, `print`, template partials, returned HTML strings.
- **External I/O.** `wp_remote_*`, `file_get_contents` on URLs, file reads/writes, `shell_exec`/`exec`/`system` (any of these last three warrant immediate flagging).
- **Third-party dependencies.** `composer.json`, vendored libraries, bundled JS packages.

Keep this inventory in a scratchpad. It becomes the input for Phase 2 and the appendix of the report.

### Phase 2 — Analysis

Goal: run each finding category against the inventory from Phase 1. Work through all four checklists — do not stop after security just because something looks bad there.

Read the reference files as you go. They are the authoritative checks:

- `references/security-checklist.md` — nonces, capability checks, sanitization, escaping, SQL injection, CSRF, file upload, auth on REST/AJAX, secrets, redirects, serialization, SSRF
- `references/performance-checklist.md` — N+1 queries, autoloaded options, caching, transients, asset loading, meta queries, remote request timeouts
- `references/readability-checklist.md` — WPCS, PHPDoc, naming, file structure, separation of concerns, dead code
- `references/ux-checklist.md` — accessibility, i18n/l10n, admin notices, mobile responsiveness, error states

For each potential issue, before recording it, verify three things:

1. **Is it actually present in the code?** Don't cite a theoretical problem.
2. **What's the concrete impact?** "Could allow an unauthenticated user to delete any post" beats "lacks authorization."
3. **What's the fix?** If you can't describe the remediation in one or two sentences, either dig deeper or don't include the finding.

#### Severity framework

Assign each finding a severity. Be strict — inflation makes the report less useful.

| Severity | Meaning | Examples |
|---|---|---|
| **Critical** | Exploitable vulnerability, data loss risk, or site-breaking bug. Fix before next deploy. | Unauthenticated privilege escalation, SQL injection, RCE, unescaped user input rendered to admins, `permission_callback` missing on destructive REST route |
| **High** | Serious bug or vulnerability requiring authentication/specific conditions, or a performance issue that will degrade production under normal load. | Authenticated XSS, missing nonce on state-changing action, N+1 query on a public page, non-autoloaded large option accessed on every request becoming autoloaded |
| **Medium** | Real problem with a clear impact but limited blast radius. | Missing capability check where a nonce still gates access, missing transient on an expensive external call, inaccessible form control, hardcoded English strings |
| **Low** | Quality and maintainability issues, minor UX gaps, style inconsistencies. | WPCS violations, missing PHPDoc, inconsistent naming, dead code, minor contrast issues |
| **Info** | Observations worth mentioning but not defects. | Deprecated-but-still-working API, opportunity for refactor, version mismatch |

When a finding sits between two severities, bump it up, not down.

### Phase 3 — Report

Use the exact structure in `references/report-template.md`. Key rules:

- Lead with an **Executive summary** — 3–6 sentences a non-engineer can understand. State the overall health, count of critical/high issues, and top three things to fix first.
- Sort findings **by severity, then by file**.
- Every finding needs: title, severity, location (`file.php:L123`), impact, evidence (short code excerpt, 3–10 lines), recommendation.
- Include an **appendix** with the Phase 1 inventory — it gives the reader a map of what was reviewed.

Write the report to a single markdown file in the project root (or wherever the user specifies) as `AUDIT-<YYYY-MM-DD>.md`. Don't scatter findings into separate files; a single document is easier to triage.

## Tips that pay off

- **Grep is your friend.** Searches like `rg -n "\\\$_(GET|POST|REQUEST|COOKIE)" --type php`, `rg -n "wp_ajax_" --type php`, `rg -n "register_rest_route" --type php`, `rg -n "\\\$wpdb->" --type php`, and `rg -n "\beval\b|shell_exec|passthru|system\(" --type php` get you 80% of the attack surface in under a minute.
- **Read the plugin's main file top-to-bottom first.** It reveals the author's conventions, which informs how to read everything else.
- **Check the JS too.** A theme or plugin's frontend JS often makes AJAX calls whose handlers are the real target of security analysis. Trace both sides.
- **Don't assume WPCS means secure.** A plugin can pass WPCS checks and still be wildly insecure. WPCS is about style and some safety patterns, not a replacement for a security review.
- **When uncertain, say so.** If a function's behavior depends on runtime data you can't see, note the assumption in the finding rather than overstating confidence.

## If the project is large

For a codebase too big to hold in working memory at once:

1. Do recon on the whole thing — don't skip it.
2. Tackle the mu-plugin first (usually higher-privilege code), then the theme.
3. Within each, prioritize files referenced from entry points and hook registrations before utility/helper files.
4. If a particular area (say, a REST controller class) is dense, audit it in isolation and write up findings before moving on.

## Confirming before running

Before kicking off a full audit, confirm with the user:

- Which directories are in scope (mu-plugin path, theme path, anything else)?
- Report destination (default: `AUDIT-<date>.md` in the project root)?
- Should findings include suggested code patches, or just descriptions of the fix?
- Any areas of special concern (e.g., "we just added a REST API, pay extra attention there")?

Then proceed through the three phases.
