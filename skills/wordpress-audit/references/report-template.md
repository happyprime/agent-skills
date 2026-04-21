# Report template

The audit report is always a single markdown file named `AUDIT-<YYYY-MM-DD>.md` in the project root (or wherever the user specifies).

Use exactly this structure. Omit sections that have no content rather than leaving empty headers.

---

# WordPress Code Audit — [Project Name]

**Date:** YYYY-MM-DD
**Reviewer:** Claude (Claude Code)
**Scope:**
- mu-plugin: `wp-content/mu-plugins/<name>/` (version X.Y.Z)
- Theme: `wp-content/themes/<name>/` (version X.Y.Z)

## Executive summary

Three to six sentences in plain language. Cover:

- Overall state of the code ("generally solid", "serviceable but needs work", "significant concerns").
- Count of findings by severity.
- Top 2–3 things to fix immediately.
- Any systemic patterns worth the team's attention.

Example opening: "The theme and mu-plugin are workable but have three critical security issues that should be resolved before the next deploy. All three stem from the same pattern — REST endpoints registered without a `permission_callback` — suggesting a template worth applying consistently across the codebase."

## Findings by severity

Group all findings under these headings in this order. Skip any that are empty.

### Critical

### High

### Medium

### Low

### Informational

---

## Finding format

Every finding uses this structure:

#### [NNN] Short title

**Severity:** Critical | High | Medium | Low | Info
**Category:** Security | Performance | Readability | UX
**Location:** `relative/path/to/file.php:L123`

**Impact:**
One or two sentences describing what can happen in practice. Be concrete — "unauthenticated users can delete any post via this endpoint" beats "authorization is missing."

**Evidence:**
```php
// 3–10 lines of the actual offending code, with surrounding context.
// Include the line numbers where helpful.
public function delete_item( $request ) {
    $post_id = (int) $request['id'];
    wp_delete_post( $post_id, true ); // No capability check
    return rest_ensure_response( array( 'deleted' => true ) );
}
```

**Recommendation:**
How to fix it, in one or two sentences plus optionally a short code snippet. Reference WordPress APIs and conventions.

```php
// Example patch:
'permission_callback' => function () {
    return current_user_can( 'delete_posts' );
},
```

---

## Numbering and linking

Number findings sequentially across the whole report (001, 002, 003…). Use them as anchor references in the executive summary ("See [003] and [007]").

## Appendix A — Recon inventory

The inventory gathered in Phase 1. Keep this concise but complete — it shows the reader what was actually reviewed.

### Entry points

- `wp-content/mu-plugins/acme-loader.php` — loads `acme/plugin.php`
- `wp-content/mu-plugins/acme/plugin.php` — main plugin bootstrap
- `wp-content/themes/acme-theme/functions.php` — theme bootstrap

### Registered hooks

Brief list: hook name → callback → file:line. Truncate if very long, but indicate count.

### AJAX handlers

- `wp_ajax_acme_save_item` → `Acme\Ajax\Items::save()` → `mu-plugins/acme/src/Ajax/Items.php:24`
- `wp_ajax_nopriv_acme_track` → ...

### REST routes

- `POST /acme/v1/items` → `Acme\Rest\Items::create()` — permission_callback: `__return_true` ⚠️
- `GET /acme/v1/items/(?P<id>\d+)` → `Acme\Rest\Items::get()` — permission_callback: `current_user_can('read')`

### Custom post types and taxonomies

- CPT `acme_item` (public, with archive)
- Taxonomy `acme_category` (hierarchical)

### Custom database tables

- `{prefix}acme_logs` (created in `mu-plugins/acme/src/Install.php`)

### External services called

- `api.stripe.com` — payments (via `Acme\Services\Stripe`)
- `api.mailgun.net` — transactional email

### Third-party dependencies

List `composer.json` entries and notable JS packages. Note any version concerns here; detailed CVE findings go in the Security section.

## Appendix B — Files reviewed

A tree or list of every PHP file inspected. This protects against "but did you check X?" questions and gives the next reviewer a baseline.

## Appendix C — What was not reviewed

If any areas were explicitly out of scope (other plugins, WordPress core, third-party libraries beyond version-level checks), list them here. Transparency about scope limits is more useful than a pretense of completeness.

---

## Tone and style notes for the reviewer

- Be direct, not harsh. "This query is vulnerable to SQL injection" is direct. "This developer clearly doesn't know what they're doing" is harsh and unhelpful.
- Assume competence. Findings should explain the issue and the fix, not imply the author is ignorant.
- Prefer "this" to "you" when describing code. "This loop queries the database on each iteration" reads better than "you are querying the database in a loop."
- Skip filler. Every finding should be scannable. Avoid "I noticed that possibly there might be..." — just state what's there.
- Don't invent severity to pad the list. A five-item critical list is more useful than a sixty-item mixed list where nothing feels urgent.
