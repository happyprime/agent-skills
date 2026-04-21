# Security checklist

WordPress-specific security checks, organized by category. For each, the goal is to find concrete instances in the code — not theoretical risks.

## Table of contents

1. Direct file access
2. Input sanitization
3. Output escaping
4. Nonces (CSRF)
5. Capability checks (authorization)
6. SQL injection
7. REST API authorization
8. AJAX authorization
9. File uploads and file operations
10. Redirects
11. Remote requests and SSRF
12. Secrets and credentials
13. Dangerous functions
14. Serialization
15. Passwords and cryptography
16. Third-party dependencies

---

## 1. Direct file access

**Check:** Every PHP file that isn't a front-controller or explicitly loaded entry point should prevent direct web access.

Look for this near the top of the file:

```php
defined( 'ABSPATH' ) || exit;
```

Files missing this can sometimes be hit directly via the web server, which has leaked fatal errors, stack traces, and even allowed logic to execute out of context.

**Flag if:** A PHP file in `mu-plugins/` or the theme contains code (functions, classes, executed statements) but has no ABSPATH guard. Template parts loaded via `get_template_part()` are the main exception, though a guard still doesn't hurt.

---

## 2. Input sanitization

**Principle:** Sanitize on input, escape on output. Use the narrowest sanitizer that fits the data type.

| Data type | Correct sanitizer |
|---|---|
| Plain text | `sanitize_text_field()` |
| Textarea (multi-line text) | `sanitize_textarea_field()` |
| Integer | `absint()` or `(int)` |
| Email | `sanitize_email()` |
| URL | `esc_url_raw()` (for storage) |
| Slug/key | `sanitize_key()` or `sanitize_title()` |
| File name | `sanitize_file_name()` |
| HTML (limited) | `wp_kses( $html, $allowed_tags )` |
| HTML (post content) | `wp_kses_post()` |
| Hex color | `sanitize_hex_color()` |
| Array of values | Iterate and sanitize each element |

**Flag if:**

- Raw `$_GET` / `$_POST` / `$_REQUEST` / `$_COOKIE` / `$_SERVER` values are used without any sanitizer.
- Data goes into the database or a shell/file operation without validation.
- `stripslashes()` is used without a corresponding sanitizer (it doesn't sanitize, just unescapes).
- A too-permissive sanitizer is used (e.g., `wp_kses_post()` on data meant to be a single-line plain-text field).

---

## 3. Output escaping

**Principle:** Escape at the point of output, matching the context.

| Context | Correct escaper |
|---|---|
| HTML body text | `esc_html()` |
| HTML attribute | `esc_attr()` |
| URL in href/src | `esc_url()` |
| Inside a `<script>` variable | `wp_json_encode()` or `esc_js()` for strings in inline JS |
| Inline style attribute value | `esc_attr()` (and validate the value beforehand) |
| Translated string (HTML body) | `esc_html__()` / `esc_html_e()` |
| Translated string (attr) | `esc_attr__()` / `esc_attr_e()` |
| Rich HTML content | `wp_kses_post()` or custom `wp_kses()` |
| Already-safe HTML from trusted filter | Escape anyway unless definitively proven safe |

**Flag if:**

- A value is echoed inside HTML without escaping.
- `$_GET` / `$_POST` / user meta / post meta / option values are printed directly.
- Translator-facing strings use `__()` or `_e()` in HTML contexts (should be `esc_html__()` / `esc_html_e()`).
- Data is interpolated into an attribute without `esc_attr()`.
- A URL is rendered without `esc_url()`.
- An inline `<script>` embeds PHP values without `wp_json_encode()`.

Escaping is cheap. The rule of thumb: if in doubt, escape. Double-escaping an already-safe string produces ugly output but no security hole; missing an escape on untrusted data does.

---

## 4. Nonces (CSRF protection)

**Principle:** Any state-changing request initiated from a form, link, or AJAX call in the admin (or frontend, if it affects server state) must be protected by a nonce.

**Form pattern:**

```php
wp_nonce_field( 'my_action_slug', 'my_action_nonce' );
// On submit:
if ( ! isset( $_POST['my_action_nonce'] )
    || ! wp_verify_nonce( sanitize_key( wp_unslash( $_POST['my_action_nonce'] ) ), 'my_action_slug' ) ) {
    wp_die( esc_html__( 'Security check failed.', 'textdomain' ) );
}
```

**URL pattern:**

```php
$url = wp_nonce_url( $base_url, 'my_action_slug', 'my_action_nonce' );
// On the receiving end:
check_admin_referer( 'my_action_slug', 'my_action_nonce' );
```

**AJAX pattern:**

```php
check_ajax_referer( 'my_action_slug', 'nonce' );
```

**REST pattern:** REST handles nonces automatically for cookie-authenticated requests via the `X-WP-Nonce` header; the `permission_callback` is what you're verifying.

**Flag if:**

- A form submits and the handler reads `$_POST` without any nonce verification.
- An admin-post or AJAX handler mutates data without `check_admin_referer()` / `check_ajax_referer()`.
- A nonce exists but isn't actually verified (generated on the form, ignored on the receiver).
- Nonces are used in place of capability checks — they're not authorization, they're anti-CSRF.

---

## 5. Capability checks (authorization)

**Principle:** Every privileged action needs a capability check. A nonce proves the request came from the user's browser; a capability check proves the user is allowed to do the thing.

Common patterns:

```php
if ( ! current_user_can( 'manage_options' ) ) {
    wp_die( esc_html__( 'You are not allowed to do this.', 'textdomain' ) );
}

// For post-specific actions:
if ( ! current_user_can( 'edit_post', $post_id ) ) { ... }
```

**Flag if:**

- An admin page renders or a settings form saves without `current_user_can()`.
- A REST endpoint uses `permission_callback => '__return_true'` for anything other than genuinely public read-only data.
- A capability is checked but the wrong one is used (e.g., `read` instead of `manage_options` for an admin action, or `edit_posts` for deleting someone else's post).
- `is_admin()` is used as authorization. It is not — `is_admin()` only checks whether the request targets an admin URL, not whether the user is authenticated or authorized.
- `is_user_logged_in()` is the sole check for an action that should be role-restricted.

---

## 6. SQL injection

**Principle:** Every SQL query that embeds any dynamic value must use `$wpdb->prepare()`. No exceptions.

**Correct:**

```php
$results = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT * FROM {$wpdb->prefix}custom_table WHERE user_id = %d AND status = %s",
        $user_id,
        $status
    )
);
```

**Wrong:**

```php
// String concatenation — SQL injection
$wpdb->get_results( "SELECT * FROM ... WHERE user_id = " . $user_id );
// String interpolation — also SQL injection
$wpdb->get_results( "SELECT * FROM ... WHERE user_id = $user_id" );
// sprintf without prepare — does NOT escape, just formats
$wpdb->get_results( sprintf( "SELECT * FROM ... WHERE id = %d", $id ) );
```

**Placeholders:**

- `%d` — integer
- `%f` — float
- `%s` — string (quoted)
- `%i` — identifier (table/column name, WP 6.2+)

Identifiers (table and column names) historically could not be parameterized. For pre-6.2 support or in general, whitelist them against a known list rather than accepting them from user input.

**Flag if:**

- Any `$wpdb->query()`, `get_results()`, `get_var()`, `get_row()`, `get_col()` call builds its SQL via concatenation or interpolation of dynamic values.
- `$wpdb->prepare()` is called but the result isn't actually passed to the query method (a common mistake).
- Dynamic `ORDER BY` or `LIMIT` values come from user input without whitelisting.
- `LIKE` queries interpolate user input without `$wpdb->esc_like()` in addition to `prepare()`.

---

## 7. REST API authorization

**Principle:** Every `register_rest_route()` must declare a `permission_callback`. Missing this is a deprecation warning in modern WordPress and an outright vulnerability in older versions.

**Correct:**

```php
register_rest_route( 'myplugin/v1', '/items', array(
    'methods'             => 'POST',
    'callback'            => array( $this, 'create_item' ),
    'permission_callback' => function () {
        return current_user_can( 'edit_posts' );
    },
    'args'                => array(
        'title' => array(
            'required'          => true,
            'type'              => 'string',
            'sanitize_callback' => 'sanitize_text_field',
        ),
    ),
) );
```

**Flag if:**

- `permission_callback` is missing entirely.
- `permission_callback => '__return_true'` is used for a non-public endpoint.
- A destructive method (POST/PUT/PATCH/DELETE) has only `is_user_logged_in()` as its check — any logged-in user shouldn't necessarily be able to do everything.
- `args` schema is missing or loose on a public endpoint (no `sanitize_callback`, no `validate_callback`, no `type`).
- Validation is done inside the handler instead of via `sanitize_callback` / `validate_callback`, making the schema misleading.

---

## 8. AJAX authorization

**Principle:** `wp_ajax_<action>` is for logged-in users; `wp_ajax_nopriv_<action>` is for logged-out users. A handler registered under both is effectively public.

**Flag if:**

- A `wp_ajax_nopriv_` handler does something stateful (writes to the database, sends email, triggers an API call).
- A handler is registered for both and performs state changes without internal capability differentiation.
- A handler has no `check_ajax_referer()` call.
- The handler sanitizes output but never sanitizes input.

---

## 9. File uploads and file operations

**Upload rules:**

- Use `wp_handle_upload()` or `media_handle_upload()` — never move uploaded files yourself.
- Validate mime type against a whitelist, not just the extension.
- Never trust `$_FILES['…']['type']` — it's client-supplied.
- Restrict where uploaded files are stored and how they're served.

**File path rules:**

- Never concatenate user input into file paths without `basename()` and path normalization.
- Validate resolved paths with `realpath()` and check that they start with the expected base directory — this prevents `../` path traversal.
- Use WP_Filesystem for filesystem operations when possible.

**Flag if:**

- `move_uploaded_file()` is used directly instead of `wp_handle_upload()`.
- Filename from user input is used in a path without `basename()`.
- `include` / `require` / `file_get_contents` / `fopen` is called with any part of the path coming from user input and no path-traversal check.
- Arbitrary file extensions are accepted for upload.
- Uploaded files land in a web-accessible directory without execution restrictions (`.htaccess` or equivalent).

---

## 10. Redirects

**Principle:** Use `wp_safe_redirect()` instead of `wp_redirect()` when the destination is influenced by user input. `wp_safe_redirect()` restricts the destination to the site's own host, preventing open-redirect attacks used in phishing.

**Flag if:**

- `wp_redirect()` receives a value derived from `$_GET['redirect_to']` or similar, without domain whitelisting.
- A redirect is issued without `exit;` immediately after (allows code to continue executing).
- PHP's `header('Location: …')` is used directly instead of WP's redirect helpers.

---

## 11. Remote requests and SSRF

**Principle:** Use `wp_remote_*` for outbound HTTP requests. They add filters, caching integration, and proxy support.

**Check:**

- Every `wp_remote_get()` / `wp_remote_post()` has a `timeout` argument. The WP default (5s) is acceptable; missing it implies relying on a silent default.
- Responses are checked with `is_wp_error()` before `wp_remote_retrieve_body()`.
- URLs derived from user input are validated — at minimum, checking the scheme is `http`/`https` and the host is not an internal address (127.0.0.1, 10.x, 192.168.x, etc.) if SSRF matters for the use case.

**Flag if:**

- `file_get_contents()` is used with a URL (bypasses WP's filter stack and allows_url_fopen issues).
- `curl_*` functions are called directly rather than `wp_remote_*`.
- A remote URL is fetched with any part coming from user input and no SSRF protection.
- Response body is used without checking for `WP_Error`.

---

## 12. Secrets and credentials

**Flag if:**

- API keys, tokens, database passwords, or signing secrets appear as literal strings in the code.
- Secrets are stored in an option without documentation that they should be set elsewhere (e.g., `wp-config.php` via `define()`).
- A `.env` file or credentials file is committed to the repo (check the directory listing).
- Stripe/Mailgun/AWS/OAuth keys appear anywhere other than a `wp-config.php` constant or a secrets manager.

Correct pattern:

```php
$api_key = defined( 'MY_SERVICE_API_KEY' ) ? MY_SERVICE_API_KEY : '';
```

---

## 13. Dangerous functions

Search the codebase for these and investigate each occurrence:

- `eval(` — rarely legitimate
- `assert(` with a string argument — same as eval on older PHP
- `shell_exec`, `exec`, `system`, `passthru`, `popen`, `proc_open`, backticks — command execution
- `preg_replace` with `/e` modifier — deprecated, executes as PHP
- `unserialize(` — see Serialization below
- `create_function(` — removed in PHP 8, same risks as eval

Each of these, when present, needs a justification comment or a finding.

---

## 14. Serialization

**Principle:** `unserialize()` on untrusted input can lead to PHP object injection and, with a vulnerable class in scope, to remote code execution.

**Flag if:**

- `unserialize()` is called on any value that originated from user input, cookies, or unauthenticated requests.
- `maybe_unserialize()` is called on untrusted data (it's the same risk).

For data that needs structured storage from untrusted sources, use `wp_json_encode()` / `json_decode()`. If `unserialize()` is required for a specific legacy reason, use `unserialize($data, ['allowed_classes' => false])`.

---

## 15. Passwords and cryptography

**Flag if:**

- `md5()` or `sha1()` is used for password hashing (use `wp_hash_password()` / `wp_check_password()`).
- Custom encryption using `mcrypt_*` or homemade XOR schemes.
- Random values generated with `rand()` / `mt_rand()` for security-sensitive purposes (use `wp_generate_password()`, `wp_generate_uuid4()`, or `random_bytes()`).
- Timing-sensitive comparisons use `==` instead of `hash_equals()`.

---

## 16. Third-party dependencies

**Check:**

- `composer.json` dependencies — are any pinned to versions with known CVEs?
- Bundled JS libraries — what's the version, is it current?
- Vendored PHP libraries in the repo — when were they last updated?

A quick `composer audit` or `npm audit` (if there's a package.json) is often the fastest way to surface known-vulnerable dependencies. Note the findings in the report even if you can't remediate.
