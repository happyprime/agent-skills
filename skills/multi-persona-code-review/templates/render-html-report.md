# HTML report renderer

Generates a self-contained, single-file `report.html` from a run directory. Open it in any browser, print it, share it as an artifact. No build step, no server.

## When invoked

Input: a run directory under `${PROJECT_ROOT}/.reviews/{YYYY-MM-DD-HHMM}/`. If the user does not specify one, use the most recent run directory.

Output: `${RUN_DIR}/report.html`.

## What to read

- `${RUN_DIR}/summary.md` — the executive content.
- `${RUN_DIR}/findings/*.md` — every final finding.
- `${RUN_DIR}/decisions-needed.md` — if present, render as its own section.
- `${RUN_DIR}/needs-debate.md` — if present, render as its own section.
- `${RUN_DIR}/nice-to-have.md` — if present, render as a collapsible footer section.

Do not load anything from `raw/` — raw findings are pre-synthesis and not for distribution.

## What to produce

A single self-contained HTML file with:

### Structure

1. **Header.** Project name (from summary), date, run-directory path.
2. **Executive summary block.** Rendered from `summary.md`'s executive overview section.
3. **Counts strip.** Color-coded severity counts as pill badges.
4. **Filter bar.** Sticky on scroll. Checkboxes for each severity (default all on), checkboxes for each category present, checkboxes for each reviewer persona present, and a search input (matches in title, problem, evidence, files).
5. **Findings list.** One collapsible card per finding, sorted by severity then by ID. Card header shows severity badge, ID, title, category, reviewers, effort. Click to expand body sections.
6. **Decisions section.** Always rendered if `decisions-needed.md` is present and non-trivial.
7. **Debate section.** Always rendered if `needs-debate.md` is present.
8. **Nice-to-have footer.** Collapsed by default if `nice-to-have.md` is present.

### Styling

- **Severity colors** (consistent across badges, card borders, and counts):
  - critical: `#b91c1c` (red)
  - high: `#c2410c` (orange)
  - medium: `#b45309` (amber)
  - low: `#1d4ed8` (blue)
  - info: `#4b5563` (grey)
- Light theme by default, with `@media (prefers-color-scheme: dark)` overrides for body backgrounds and text. Keep severity colors unchanged in dark mode for recognition.
- Responsive layout: usable on mobile widths (≥ 360px) with the filter bar collapsing to a "Filters ▾" toggle.
- Print stylesheet: hide the filter bar; expand all collapsible cards; remove background colors except severity badges; ensure code blocks don't overflow the page.

### Behavior

- **Vanilla JS only.** No build step, no framework. One `<script>` block inline.
- **Syntax highlighting** via highlight.js from a CDN. Use the minimal common-languages bundle and a theme that works in both light and dark. Embed the CDN `<link>` and `<script>` tags directly; degrade gracefully if the CDN is unreachable (the code is still readable, just unhighlighted).
- **Filters** combine with AND across categories (severity AND category AND reviewer) and AND with search. State is reflected in `location.hash` so a filtered view is shareable.
- **Search** is a simple case-insensitive substring match over title, problem, evidence, and `files`. Debounce by 200ms.
- **Anchors.** Each finding gets `id="<finding-id>"`. The summary's top-priorities list links to anchors.

### Accessibility

- Real `<button>` elements for collapse toggles; not `<div onclick>`.
- Filter checkboxes have associated `<label>`s.
- Color is not the only signal for severity — also include text and a leading icon character.
- Focus styles visible on every interactive element.
- Card collapse uses `<details>`/`<summary>` for native keyboard support.

## Markdown rendering

The findings are markdown. Render them to HTML at generation time (do not ship a JS markdown parser). Specifically:

- Code fences → `<pre><code class="language-<lang>">`. Apply highlight.js after DOM load.
- Tables in summary.md → real `<table>` markup.
- Lists, headings, paragraphs, inline code — standard mapping.
- Internal references like `CRIT-002` in finding bodies → linkified to `#CRIT-002` if that finding is present.

## Implementation note for the rendering sub-agent

Build the HTML by reading the markdown files and emitting an HTML string. Don't try to load markdown at runtime in the browser — the report must work as a self-contained `file://` open with no network beyond the optional highlight.js CDN.

Save the result to `${RUN_DIR}/report.html` and report the file path back to the user. Include a one-line summary of what's in the report (findings count, decisions count) so the user knows what to expect when opening it.
