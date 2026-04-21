# Performance checklist

WordPress-specific performance checks. Focus on problems that will bite in production — slow queries, unbounded loops, and the perennial "works fine with 10 posts, collapses with 10,000."

## Table of contents

1. Database queries
2. WP_Query patterns
3. Meta queries
4. Autoloaded options
5. Object caching and transients
6. Asset loading (scripts and styles)
7. Remote requests
8. Hooks and filter overhead
9. File operations
10. Image handling
11. Cron and scheduled tasks

---

## 1. Database queries

**The core rules:**

- No queries inside loops. Each query has overhead; running one per iteration produces the classic N+1 problem.
- Fetch only the columns you need. `SELECT *` on wide tables wastes memory.
- Index any custom table column used in `WHERE`, `ORDER BY`, or `JOIN`.
- Limit result sets. Unbounded `get_results()` on a table that could grow unboundedly is a latent incident.

**Flag if:**

- A `foreach` loop contains a `$wpdb` call or a `get_post_meta()` / `get_user_meta()` call that could be batched.
- `query_posts()` is used (globally destructive; should be `WP_Query` or `pre_get_posts`).
- A function like `get_all_users()` or `get_all_products()` returns every row with no pagination, and is called from a page-rendering path.
- Custom tables have no indexes beyond the primary key despite being queried by other columns.
- `SELECT *` is used where a narrow column list would do.
- `$wpdb->get_results()` is called on an unbounded table in a request-time path.

**Batch-fetch pattern to suggest:**

```php
// Instead of calling get_post_meta() inside a loop:
$post_ids = wp_list_pluck( $posts, 'ID' );
update_meta_cache( 'post', $post_ids ); // primes cache in one query
foreach ( $posts as $post ) {
    $value = get_post_meta( $post->ID, 'my_key', true ); // cached, no DB hit
}
```

---

## 2. WP_Query patterns

**Check `WP_Query` calls for:**

- `posts_per_page => -1` on a path that could return thousands of posts. Public-facing code should paginate; background jobs should batch.
- `no_found_rows => true` when pagination isn't needed (skips `SQL_CALC_FOUND_ROWS`).
- `update_post_meta_cache` / `update_post_term_cache` set to `false` when meta/terms aren't used (saves queries).
- `fields => 'ids'` when only IDs are needed (avoids hydrating full post objects).
- `suppress_filters => true` when running queries in a context where filters would bloat them unnecessarily.
- `cache_results => false` only when intentional — it's a footgun more often than an optimization.

**Flag if:**

- A frontend template runs `new WP_Query( … )` with `posts_per_page => -1` and no `fields => 'ids'`.
- `get_posts()` is used where `WP_Query` (with the flags above) would be measurably faster.
- A query is run on every page load without caching that could easily be transient-cached.

---

## 3. Meta queries

Meta queries are a common performance trap. `wp_postmeta` is a key/value table not designed for complex filtering.

**Flag if:**

- A `meta_query` joins `wp_postmeta` multiple times (each clause is a JOIN — 3+ clauses on a large site will crawl).
- A `meta_query` uses `compare => 'LIKE'` or `'NOT EXISTS'` on a large dataset.
- A frequently-accessed attribute is stored as post meta when it should be a custom taxonomy or a custom table column.
- `orderby => 'meta_value'` is used without also limiting via indexed columns.

When you see a meta-heavy architecture, the fix is often a custom table. Note this in the report as a direction for refactoring, not necessarily a blocker.

---

## 4. Autoloaded options

**Background:** WordPress loads all options with `autoload = 'yes'` into memory on every request via `wp_load_alloptions()`. A large autoloaded option is a tax on every single page view.

**Check:**

- Options added via `add_option()` default to `autoload = 'yes'`. Is that what the code intends?
- Options storing large data (serialized arrays, long strings) — should they autoload?

**Flag if:**

- `add_option()` or `update_option()` stores a large value (> a few KB) without explicit `autoload = 'no'`.
- Transients are stored via `update_option()` directly (bypasses the transient API and autoloads them).

Correct pattern:

```php
// For large data that doesn't need to be loaded every request:
add_option( 'my_plugin_big_data', $data, '', 'no' );
// Or update:
update_option( 'my_plugin_big_data', $data, false );
```

---

## 5. Object caching and transients

**Principle:** Expensive computations should be cached. WordPress provides two layers:

- **Object cache** (`wp_cache_get()` / `wp_cache_set()`): In-memory, per-request by default; persistent if the site has Memcached/Redis.
- **Transients** (`get_transient()` / `set_transient()`): Persistent, stored in the options table (or object cache if available), with expiration.

**When to use which:**

- Per-request memoization of expensive computation → object cache with `wp_cache_get/set`.
- Cross-request caching of remote API responses or expensive queries → transient with a sensible TTL.
- Very large values or those that change frequently → consider whether caching adds value at all.

**Flag if:**

- An expensive operation (external API call, complex aggregation query) runs on every request without caching.
- A transient has no expiration (`0`) where a TTL would be safer.
- A transient is used for data that must be real-time (cache lies).
- A function computes the same value multiple times within a single request (should memoize).
- `delete_transient()` is never called where data invalidation matters — stale cache is its own incident category.

---

## 6. Asset loading (scripts and styles)

**Rules:**

- Enqueue via `wp_enqueue_script()` / `wp_enqueue_style()` — never hardcode `<script>` / `<link>` in templates.
- Enqueue on the right hook: `wp_enqueue_scripts` for frontend, `admin_enqueue_scripts` for admin, `enqueue_block_editor_assets` for the block editor.
- Version assets with `filemtime()` or a build hash so cache busts work.
- Load assets conditionally — don't load checkout JS on every page.

**Flag if:**

- Scripts/styles are output via raw HTML in a template or inline via `wp_head` without enqueue.
- All assets load on every page regardless of template (`wp_enqueue_scripts` with no conditional).
- Assets lack a version number (third argument of enqueue), making cache-busting impossible.
- jQuery is re-enqueued from a CDN, overriding the WP-bundled copy.
- Large vendor libraries are loaded for a feature that appears on one page.
- `wp_localize_script()` is used for data that's not actually translation-related (modern choice: `wp_add_inline_script()` with `wp_json_encode()`).
- Scripts load in `<head>` without `defer` / `async` when they could load in the footer.

---

## 7. Remote requests

**Rules:**

- Every `wp_remote_*` call must have a timeout. The default is 5 seconds; anything more than 30 is suspicious outside of background jobs.
- Cache remote responses via transients when appropriate.
- Handle `WP_Error` gracefully — a failed remote call should not crash the page.

**Flag if:**

- A frontend or admin page makes a synchronous remote request on every load without caching.
- Timeout is set high (30s+) in a request-blocking path.
- Errors from `wp_remote_get()` aren't checked — truthy responses can still be error objects.
- Background-job-like work runs on frontend page loads (should be WP-Cron or Action Scheduler).

---

## 8. Hooks and filter overhead

**Flag if:**

- Heavy computations run on high-frequency hooks like `init`, `wp`, `the_content`, `template_redirect`, or `admin_init` unconditionally.
- A function is attached to `the_content` and runs expensive work even when the main query isn't a singular post.
- `add_filter` runs with a priority that fights with core or other plugins (very low or very high) without comment explaining why.
- The same callback is registered on multiple hooks, each running the full body.
- `pre_get_posts` modifies queries without checking `is_main_query()` and/or `is_admin()` first, affecting unrelated queries site-wide.

---

## 9. File operations

**Flag if:**

- File I/O runs on every page load (reading config files, parsing CSV/JSON) without caching.
- A directory is scanned (`scandir`, `glob`, `DirectoryIterator`) on a request-time path.
- `require` / `include` is called in a loop.
- Temporary files are created but not cleaned up.

WordPress's autoloader and class system should do most file-loading work. Explicit includes in hot paths deserve a look.

---

## 10. Image handling

**Flag if:**

- Full-size images are served where a smaller registered size (`thumbnail`, `medium`, `large`) would suffice.
- Images lack `width` / `height` attributes (triggers layout shift, hurts CLS).
- Images aren't served with `loading="lazy"` (WP does this by default for post content; custom templates may not).
- Custom image sizes are registered but the upload pipeline never regenerates them (leaves stale or missing variants).
- SVGs are uploaded without sanitization (this is a security issue too).
- A resize happens on the fly on every request instead of via `wp_get_attachment_image_src()` which uses pre-generated sizes.

---

## 11. Cron and scheduled tasks

**Principle:** WP-Cron runs on page loads, not on a system cron. Long-running scheduled tasks can slow a page load to a crawl.

**Flag if:**

- A scheduled event does heavy work (batch processing, bulk API calls) synchronously in the cron callback with no batching.
- The project uses `wp_schedule_event()` for work that should use Action Scheduler (batching, retry, admin visibility).
- Events are scheduled but never unscheduled on deactivation/uninstall.
- `wp_schedule_single_event()` fires on every request because the check for existing events is missing.
- DISABLE_WP_CRON is set but no system cron is configured to hit `wp-cron.php`.
