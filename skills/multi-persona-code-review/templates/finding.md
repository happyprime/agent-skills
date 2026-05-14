# Finding schema

Every finding produced by this skill — raw (Stage 1) or final (Stage 3) — uses this exact structure. The schema is intentionally strict so that downstream rendering (HTML report, GitHub issues) is mechanical.

A finding is a single markdown file with YAML frontmatter followed by structured body sections.

## Frontmatter

### Required fields

```yaml
---
id: <STRING>            # See "ID conventions" below
title: <STRING>         # Short title, < 80 chars, no trailing period
severity: <ENUM>        # critical | high | medium | low | info
category: <ENUM>        # security | performance | bug | ux | accessibility | code-quality | architecture | operations | testability
confidence: <ENUM>      # high | medium | low
effort: <ENUM>          # trivial | small | medium | large | very-large
files:                  # Array of "path:line" or "path:start-end" strings, relative to project root
  - "path/to/file.php:42"
  - "path/to/other.php:120-145"
reviewers:              # Array of persona slugs that surfaced this finding
  - <persona-slug>
status: open            # Always "open" at creation; lifecycle managed by GitHub or follow-up
---
```

### Optional fields

```yaml
subcategory: <STRING>           # Free-form, e.g. "csrf", "n-plus-one", "idempotency"
original_ids:                   # Pre-synthesis raw IDs that merged into this finding (Stage 3 only)
  - <RAW-ID>
agreement: <ENUM>               # unanimous | majority | contested  (Stage 3 only)
labels:                         # Labels to apply when filed as a GitHub issue
  - <STRING>
references:                     # Links to docs, CVEs, advisories
  - <URL>
assignee_hint: <STRING>         # GitHub login or team name; advisory only
blocks_release: <BOOL>          # true if this should block the next deploy
```

### ID conventions

- **Raw findings (Stage 1).** Persona-prefixed and zero-padded sequential within the persona's output directory.
  - `SEC-001` … security-engineer
  - `PERF-001` … performance-engineer
  - `WP-001` … wordpress-developer
  - `WOO-001` … woocommerce-specialist
  - `UX-001` … frontend-ux-engineer
  - `OPS-001` … devops-engineer
  - `QA-001` … qa-engineer

- **Final findings (Stage 3).** Severity-prefixed and zero-padded sequential within the severity tier.
  - `CRIT-001`, `CRIT-002`, …
  - `HIGH-001`, `HIGH-002`, …
  - `MED-001`, `MED-002`, …
  - `LOW-001`, `LOW-002`, …
  - `INFO-001`, `INFO-002`, …

### Effort scale

- **trivial:** < 1 hour
- **small:** < 1 day
- **medium:** 1–3 days
- **large:** 3 days – 2 weeks
- **very-large:** > 2 weeks

Estimates assume an engineer familiar with the codebase. If a fix is hard to estimate, lean larger and mark `confidence: low`.

## Body

Sections appear in this order. Required sections are required. Optional sections are omitted entirely (no empty headers) when they have no content.

### Required: `## Problem`

One to three paragraphs in plain language. State what's wrong as if explaining to the engineering lead. Avoid jargon when a plain word will do. Do not restate the title.

### Required: `## Evidence`

A fenced code block containing the offending code, 3–10 lines with enough context to be readable. Always include the language tag (`php`, `js`, `tsx`, etc.). Cite line numbers in a comment at the top of the snippet:

````markdown
```php
// path/to/file.php:42-50
public function delete_item( $request ) {
    $post_id = (int) $request['id'];
    wp_delete_post( $post_id, true ); // No capability check
    return rest_ensure_response( array( 'deleted' => true ) );
}
```
````

For findings that span multiple files, include one snippet per file with separate fenced blocks.

### Required: `## Impact`

Concrete. State **who** is affected, **what** they experience or what's at risk, **when** the issue triggers, and **how bad** it is. Examples:

- "Any unauthenticated visitor can delete arbitrary posts by sending a DELETE request to `/wp-json/acme/v1/items/{id}`. Loss is irreversible without backup restore."
- "On the orders list page, response time grows linearly with the number of orders; at 10k orders we observe ~30s page loads."
- "Screen reader users cannot complete checkout; the payment field has no associated label, and the submit error is not announced."

Avoid "could possibly under some circumstances." If the impact is conditional, name the condition.

### Required: `## Recommended Fix`

A specific, actionable change with a code example if possible. Not "consider X" — either specify the change or state precisely what context is missing.

````markdown
Add a `permission_callback` to the route registration:

```php
register_rest_route( 'acme/v1', '/items/(?P<id>\d+)', array(
    'methods'  => WP_REST_Server::DELETABLE,
    'callback' => array( $this, 'delete_item' ),
    'permission_callback' => function ( $request ) {
        return current_user_can( 'delete_post', (int) $request['id'] );
    },
) );
```
````

If the fix requires a user choice (e.g., "either rate-limit this endpoint or remove it entirely"), this is a sign the finding belongs in `decisions-needed.md` instead.

### Required: `## Verification`

How to confirm the fix worked. Concrete steps, commands, or test cases. Examples:

- "Make an unauthenticated DELETE request to the endpoint and verify it returns 401."
- "Run `EXPLAIN ANALYZE` on the orders list query and verify the new index is used."
- "Manually navigate the checkout with a screen reader and verify the payment field is announced."

### Optional: `## Synthesis notes`

Added by the senior-engineer synthesizer in Stage 3 only. Explains severity decisions, debate outcomes, and any reframing during synthesis.

### Optional: `## Open questions`

Things you'd ask the developer if you could. The synthesizer may relay these into `decisions-needed.md`.

### Optional: `## Related`

Other finding IDs that touch the same code or pattern. Use plain references: `Related: CRIT-002, MED-014`.

## Discipline rules

- **Never invent file paths or line numbers.** If you cite a line, you have read that line. If you're not sure, set `confidence: low` and either reduce the cite or omit specific line numbers.
- **Never leave Recommended Fix as "consider doing X" or "could be improved."** Specify the change. If you can't, state what context is missing (e.g., "Confirm the gateway supports webhook signature verification; if so, switch to signed verification — see provider docs at <URL>").
- **One finding = one issue.** If you find a missing nonce in three different places, that's one finding with three `files:` entries — unless the impact differs materially.
- **Title is descriptive, not generic.** "Missing capability check" is bad. "Missing capability check on REST endpoint that deletes any post by ID" is good.
- **Severity follows your persona's rubric in Stage 1.** Synthesis re-rates in Stage 3 using the cross-domain rubric.

## Worked example

A complete finding file looks like this end to end:

```markdown
---
id: CRIT-001
title: Unauthenticated REST endpoint allows deleting any post
severity: critical
category: security
confidence: high
effort: small
files:
  - "includes/rest/class-items-controller.php:78-92"
reviewers:
  - security-engineer
  - wordpress-developer
status: open
subcategory: authorization
original_ids:
  - SEC-001
  - WP-014
agreement: unanimous
labels:
  - security
  - bug
references:
  - "https://developer.wordpress.org/rest-api/extending-the-rest-api/adding-custom-endpoints/#permissions-callback"
blocks_release: true
---

## Problem

The `delete_item` route is registered with `permission_callback => '__return_true'`. Any unauthenticated visitor can issue a `DELETE` request to `/wp-json/acme/v1/items/{id}` and the handler will execute `wp_delete_post( $id, true )` — a permanent delete that bypasses the trash.

## Evidence

```php
// includes/rest/class-items-controller.php:78-92
register_rest_route( 'acme/v1', '/items/(?P<id>\d+)', array(
    'methods'  => WP_REST_Server::DELETABLE,
    'callback' => array( $this, 'delete_item' ),
    'permission_callback' => '__return_true',
) );

public function delete_item( $request ) {
    $post_id = (int) $request['id'];
    wp_delete_post( $post_id, true );
    return rest_ensure_response( array( 'deleted' => true ) );
}
```

## Impact

Any anonymous visitor can permanently delete any post (including pages, products, custom post types) by sending a single HTTP request. Deletions bypass the trash, so recovery requires a database restore. Discoverable by anyone walking the REST namespace.

## Recommended Fix

Add a `permission_callback` that requires the appropriate capability for the targeted post:

```php
register_rest_route( 'acme/v1', '/items/(?P<id>\d+)', array(
    'methods'             => WP_REST_Server::DELETABLE,
    'callback'            => array( $this, 'delete_item' ),
    'permission_callback' => function ( $request ) {
        return current_user_can( 'delete_post', (int) $request['id'] );
    },
) );
```

Also: consider whether `force => true` (hard delete) is the right behavior for this API. Soft-delete via trash is usually safer and matches `wp-admin` conventions.

## Verification

1. Issue an unauthenticated DELETE request: `curl -X DELETE https://site.test/wp-json/acme/v1/items/1`. Expect HTTP 401.
2. Issue the request as a user without `delete_post` capability on the target. Expect HTTP 403.
3. Issue the request as an authorized user. Expect HTTP 200 and the post deleted.

## Synthesis notes

Both security-engineer (SEC-001) and wordpress-developer (WP-014) surfaced this independently. SEC framed it as missing authorization on a destructive REST route; WP framed it as `permission_callback => '__return_true'` anti-pattern. Same root cause; merged. Severity unanimous (`critical`) — kept at cross-domain `critical`.
```
