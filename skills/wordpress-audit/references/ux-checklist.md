# UX checklist

UX checks for a WordPress project split into four areas: accessibility, internationalization, admin experience, and frontend quality. A11y and i18n are often treated as polish; for many projects (especially ones serving a global audience or subject to legal accessibility requirements), they're table stakes.

## Table of contents

1. Accessibility (a11y)
2. Internationalization (i18n) and localization (l10n)
3. Admin experience
4. Frontend UX
5. Block editor compatibility
6. Forms
7. Error and empty states

---

## 1. Accessibility (a11y)

WordPress's own accessibility coding standards require WCAG 2.1 AA compliance for core and default themes. Custom themes should aim for the same.

**Semantic HTML:**

- Headings form a logical outline: one `<h1>` per page, no heading-level skips (`<h2>` straight to `<h4>`).
- Landmarks are used: `<header>`, `<nav>`, `<main>`, `<footer>`, `<aside>`.
- Lists use `<ul>` / `<ol>` / `<li>` — not divs with bullet CSS.
- Buttons that do things use `<button>`; links that navigate use `<a href>`. A `<div onclick>` is an accessibility flag.

**Keyboard:**

- Every interactive control is reachable via Tab.
- Focus order follows the visual order.
- Focus is visible — no `outline: none` without a replacement focus style.
- Custom widgets (menus, modals, tabs) handle keyboard interaction (Esc to close, arrow keys to navigate where appropriate).

**Screen readers:**

- Images have `alt` attributes. Decorative images use `alt=""`.
- Icon-only buttons have `aria-label` or a visually-hidden text label.
- Form controls have associated `<label>` elements (not just placeholder text).
- Dynamic content updates use `aria-live` regions where appropriate.
- `aria-*` attributes are used correctly — misuse can be worse than none.

**Color and contrast:**

- Body text has a contrast ratio of at least 4.5:1 against its background.
- Large text (18pt+ or 14pt bold+) has at least 3:1.
- UI components (buttons, form borders, focus rings) have at least 3:1 against adjacent colors.
- Color is never the sole means of conveying information (e.g., required fields aren't marked only by red).

**Motion:**

- Animations respect `prefers-reduced-motion`.
- Auto-advancing carousels can be paused.

**Flag if:**

- `<button>`s are styled as plain text links without indication of interactivity.
- Modal/dialog code doesn't trap focus or manage Esc.
- Skip-to-content link is absent or broken.
- Form validation shows errors only by color change.
- Custom dropdowns/menus don't have keyboard support.

**Useful quick check:** run the project through [axe DevTools](https://www.deque.com/axe/) or Lighthouse accessibility audit as a sanity check. Automated tools find maybe a third of a11y issues, but finding zero issues from them is a good signal.

---

## 2. Internationalization (i18n) and localization (l10n)

**Principle:** Every user-facing string should be translatable via a text domain.

**Text domain rules:**

- The plugin or theme declares `Text Domain:` in its header.
- Every translation function passes the text domain: `__( 'Save', 'acme' )`.
- The text domain string matches the plugin slug.
- `load_plugin_textdomain()` / `load_theme_textdomain()` is called on the appropriate hook (typically `init` or `after_setup_theme`).

**Translation functions:**

- `__()` — return a translated string.
- `_e()` — echo a translated string.
- `esc_html__()` / `esc_html_e()` — translated and escaped for HTML body (preferred in templates).
- `esc_attr__()` / `esc_attr_e()` — translated and escaped for attributes.
- `_n()` — plural forms ("1 item" / "%d items").
- `_x()` — with context (for strings that look identical but translate differently).
- `sprintf( __( '...%s...', 'td' ), $value )` — inject dynamic values after translation.

**Flag if:**

- Hardcoded English strings appear in user-facing output (admin pages, frontend templates, error messages).
- Translation functions are called with no text domain or the wrong one.
- Translation functions use variables as the string argument — `__( $label, 'td' )` — which breaks extraction.
- Strings are concatenated before translation: `__( 'Hello ', 'td' ) . $name` instead of `sprintf( __( 'Hello %s', 'td' ), $name )`.
- Dates, numbers, or currencies are formatted with fixed formats that don't respect locale (use `wp_date()`, `number_format_i18n()`).
- JavaScript strings are hardcoded instead of passed through `wp_set_script_translations()` or `wp_localize_script()`.
- Plural forms are handled with `if ( $n === 1 )` instead of `_n()`.

---

## 3. Admin experience

**Settings pages:**

- Use the Settings API (`register_setting()`, `add_settings_section()`, `add_settings_field()`) or, for more complex UIs, the modern block-editor / React approach.
- Section/field descriptions explain what each setting does.
- Validation errors are shown inline with the field that triggered them, not just as a generic "something went wrong."
- `settings_errors()` is called to display Settings API errors.

**Admin notices:**

- Registered via the `admin_notices` hook (or `network_admin_notices` for multisite).
- Dismissible notices use the `is-dismissible` class and have a way to persist the dismissal per-user.
- Notices are targeted — don't show a plugin's notice on every admin screen.

**Menus:**

- Top-level menu items are reserved for major features. Use submenus for settings pages of existing plugins.
- Menu labels are translated.
- Icons (dashicons or custom SVG) are consistent with admin style.

**Meta boxes and custom columns:**

- Appear in the right context (the relevant post type, not everywhere).
- Save handlers verify nonces and capabilities (covered in security).
- Columns are sortable where it makes sense.

**Flag if:**

- Admin notices appear globally on every admin page for non-global concerns.
- A settings page renders raw HTML form inputs instead of using the Settings API.
- Post types, taxonomies, or meta boxes appear with default/placeholder labels ("Custom Post Type", "Meta Box Title").
- Privileged actions lack confirmation prompts where irreversible (bulk delete, etc.).
- Helpful elements (tooltips, contextual help tabs) are missing for non-obvious features.

---

## 4. Frontend UX

**Responsive design:**

- Layouts work from ~320px viewport width and up.
- Images are responsive (not fixed pixel widths).
- Touch targets are at least 44x44px.

**Performance-adjacent UX:**

- Above-the-fold content doesn't shift as assets load (layout stability).
- Loading states exist for content that takes time — spinners, skeleton screens.
- Buttons and forms show a pending state after submission so users don't double-submit.

**Navigation:**

- Current page is indicated in the nav.
- Breadcrumbs (where used) are accurate.
- 404 pages guide the user somewhere useful rather than dead-ending.

**Flag if:**

- Horizontal scrolling is introduced at common mobile widths.
- Touch targets are too small or too close together.
- Forms submit without feedback, leaving users wondering if anything happened.
- Error pages offer no navigation or search.

---

## 5. Block editor compatibility

If the theme supports the block editor:

- `add_theme_support( 'editor-styles' )` and `add_editor_style()` are used so the editor matches the frontend.
- `add_theme_support( 'wp-block-styles' )` loads default block styles.
- `theme.json` is present and declares typography, colors, spacing tokens.
- Custom blocks are registered via `register_block_type()` with a `block.json` and are consistent with core block patterns.

**Flag if:**

- Frontend styles render block output unrecognizably differently from the editor view.
- The theme ships custom editor styles but they're out of sync with frontend CSS.
- Color palette / font size presets appear in the block editor but don't match the theme's actual design tokens.

---

## 6. Forms

**Flag if:**

- Required fields aren't visually marked or marked only with color.
- Error messages are generic ("Invalid input") rather than specific ("Email must include an @ sign").
- Error messages appear far from the field they reference.
- Form submission doesn't disable the submit button or show a pending state.
- Successful submission doesn't give clear feedback (no confirmation message, no navigation change).
- Form labels use `placeholder` instead of `<label>` — placeholder text disappears on focus and isn't a label substitute.
- Auto-complete is broken by mis-use of `autocomplete` attributes or custom inputs that browsers can't recognize.

---

## 7. Error and empty states

**Flag if:**

- An empty list or search-with-no-results shows nothing — no message, no suggestion.
- Error messages leak technical details (stack traces, SQL errors, file paths) to end users.
- 404s, 403s, 500s fall through to the browser's default page instead of a themed template.
- "No items yet" states don't explain how to create the first item.

Empty states and error states are the moments users feel the care that went into a product. They're worth flagging even when they're low severity — they affect perception disproportionately to the effort required to fix them.
