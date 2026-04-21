# Readability & maintainability checklist

Code that's hard to read is hard to audit, hard to change, and hard to secure. These checks focus on maintainability, not style wars. The goal is a codebase the next developer can pick up without a briefing.

## Table of contents

1. File and directory structure
2. Naming and prefixing
3. Separation of concerns
4. Functions and complexity
5. PHPDoc and comments
6. WordPress Coding Standards (WPCS)
7. Modern PHP usage
8. Dead code
9. Configuration and constants
10. Error handling
11. Testing surface

---

## 1. File and directory structure

**For a mu-plugin:**

- A single entry file directly in `mu-plugins/` that loads the rest (WordPress does not recurse into subdirectories). Without it, nothing inside a subfolder runs.
- Related code grouped into `includes/`, `src/`, or similar — not all in one giant file.
- Plugin header (`Plugin Name`, `Version`, `Author`, `Text Domain`, etc.) in the entry file.

**For a theme:**

- Standard structure: `style.css` with the theme header, `functions.php`, `index.php`, template files at the root (or inside `template-parts/` for partials).
- Custom logic split out of `functions.php` into `inc/` or `includes/` — a 2000-line `functions.php` is a flag in itself.
- Template parts used via `get_template_part()` rather than `include` / `require`.

**Flag if:**

- `functions.php` is a catch-all for logic that belongs in a plugin (CPTs, custom APIs, business logic). Themes should handle presentation; plugins should handle behavior. Mixing them means switching themes breaks features.
- A mu-plugin has its code buried in a subfolder with no top-level loader file — it won't run.
- Files are named inconsistently (`my-class.php` next to `MyClass.php` next to `my_class.php`).
- A single file exceeds ~500 lines without clear sectioning (this is a soft guideline, not a rule).

---

## 2. Naming and prefixing

**Principle:** Anything in the global namespace (functions, constants, hook names, option keys, CSS classes shared with other code) must be prefixed to avoid collisions.

Pick a short, unique prefix per project (e.g., `acme_`, `ACME_`). Use it consistently.

**Flag if:**

- Functions declared in the global namespace without a prefix: `function setup() { … }`, `function get_items() { … }`. These will collide with WordPress core, other plugins, or each other.
- Constants without a prefix: `define( 'DEBUG', true )`.
- Option keys without a prefix: `update_option( 'api_key', $value )`.
- Inconsistent prefix variants within one project: `acme_foo()` next to `acme_bar()` next to `ac_baz()`.
- Class names lacking a namespace and also lacking a prefix: `class Settings_Page {}` instead of `Acme\Admin\Settings_Page` or `Acme_Admin_Settings_Page`.
- Hook names generic enough to collide: `do_action( 'before_save' )` instead of `do_action( 'acme_before_save_order' )`.

**Namespaces** (PHP `namespace`) are better than prefixes for any project using autoloading. Flag a missing namespace as a medium-severity maintainability issue, not a bug.

---

## 3. Separation of concerns

**Flag if:**

- SQL, business logic, and HTML rendering are interleaved in a single function.
- A template file does significant data fetching and manipulation (should happen in a controller/hook, then pass data to the template).
- A class titled `MyPluginHelper` or `Utils` accumulates unrelated methods (a sign the architecture has no real boundaries).
- The same logic is duplicated in multiple places (copy-pasted validation, repeated query patterns).

Strong signal that refactoring would pay off: finding the same three-line escape+sanitize dance in six different places.

---

## 4. Functions and complexity

**Flag if:**

- A single function exceeds ~50 lines without a clear reason.
- A function has more than 4–5 parameters (usually means it wants to be a class or a config array).
- Cyclomatic complexity is high — nested conditionals 3+ levels deep, long `switch` statements with logic in each case, multiple early-return branches combined with late-return branches.
- A function does multiple unrelated things and its name reflects that (`save_and_email_and_log`).

---

## 5. PHPDoc and comments

**PHPDoc is required for:**

- Every public function and method — at minimum a one-line description and `@param` / `@return`.
- Every class — brief description of the class's purpose.
- Every hook the project defines via `do_action()` / `apply_filters()` — describe parameters and when it fires, so other developers can integrate with it.

**Flag if:**

- Public APIs (functions, methods, registered hooks) have no docblocks.
- Docblocks exist but are empty or generate placeholders (`@param mixed $foo`).
- Comments describe *what* the code does rather than *why* — "increment $i" is noise; "off-by-one workaround for IE11" is signal.
- Comments are stale — contradicted by the code they describe.

Don't flag missing docblocks on trivial private methods. Judgment matters.

---

## 6. WordPress Coding Standards (WPCS)

If the project has a `phpcs.xml` or `phpcs.xml.dist`, run `phpcs` against it and summarize the output in the report — don't reproduce every warning. Note the total count at each severity and call out recurring issues.

If there's no phpcs config, suggest adding one with the WordPress-Extra and WordPress-Docs rulesets.

**Common WPCS patterns worth calling out even without running the tool:**

- Yoda conditions (`42 === $value`) — WPCS expects them; note if the project is inconsistent.
- Space after opening parenthesis and before closing (WPCS style, unusual elsewhere).
- Array long-form vs short-form: modern WPCS permits `[]`, but mixed usage within one file reads sloppy.
- Improper spacing around operators.

WPCS compliance is a low-severity finding in isolation. Flag it when it's noisy enough to obscure real issues during review.

---

## 7. Modern PHP usage

Check `composer.json` or plugin header `Requires PHP` for the minimum supported PHP version. WordPress itself currently requires PHP 7.2+, many plugins target 7.4 or 8.0+.

**Flag if:**

- The project claims modern PHP support but code is still using array() instead of [], create_function(), or pre-type-hint syntax everywhere.
- Type hints are missing on method signatures despite the PHP version supporting them. Type hints catch bugs and document intent.
- Return types are missing on functions where they'd clarify contract.
- Nullability is unclear — a function returns `string|false|null` without declaring it.
- `strict_types` is inconsistent — some files declare it, others don't.

---

## 8. Dead code

**Flag if:**

- Commented-out blocks remain in files (version control is for that).
- Functions or methods are defined but never called anywhere in the codebase.
- Hook registrations point to callbacks that don't exist or have been renamed.
- Files exist in the repo but aren't loaded or referenced.
- Feature flags default to off with no clear path to either enabling or removing the feature.

A quick `grep` for each defined function against the rest of the codebase surfaces most dead code.

---

## 9. Configuration and constants

**Flag if:**

- Configuration values (API endpoints, feature flags, email addresses) are scattered as literals throughout the code rather than centralized.
- Magic numbers appear without naming (`if ( $count > 42 )` — what is 42?).
- Environment-specific values are hardcoded rather than read from `wp_get_environment_type()` or constants.
- `define()` is used for values that should be options (user-configurable) and vice-versa.

---

## 10. Error handling

**Flag if:**

- Errors are silently swallowed (`@` operator, empty catch blocks).
- Exceptions are caught and ignored without logging.
- `error_log()` is used in production code paths without gating (fills the log with noise).
- `wp_die()` is called with a raw error message instead of an escaped translated string.
- `WP_Error` is returned from some code paths but `false` from others for the same function — inconsistent error signals confuse callers.
- Errors meant for developers are shown to end users verbatim.

---

## 11. Testing surface

**Not a blocker, but worth noting:**

- Is there a `tests/` directory? Any PHPUnit or Pest configuration?
- Are there CI hooks (`.github/workflows/`, `bitbucket-pipelines.yml`)?
- Is the code structured to be testable (dependency injection, avoiding `static::` everywhere)?

A project with zero tests isn't a finding per se, but it's worth noting in the report — especially paired with recommendations around refactoring or any critical-path logic. A plugin that handles payments or user data without any test coverage is a risk worth surfacing to the business.
