# Pomo/Day — Design Phase Guidelines

## Project Structure

- **PWA structure**: `index.html`, `manifest.json`, `sw.js`, `icons/`
- **CSS location**: All styles inside `<head><style>` block in `index.html`
- **JS location**: All JavaScript inside `<body><script>`
- **Tech stack**: Pure vanilla HTML5/CSS3/JS. No frameworks or bundlers.
- **State**: Global `state` object. Persist via localStorage (`pomoday_state`).

## Hard Constraints (Do Not Break)

- **Inline CSS/JS**: All CSS and JS stay inside `index.html`. No external `.css` or `.js` files for styling or functionality.
- **State schema**: Do not rename or remove existing state fields. Add new fields to the `state` object only.
- **localStorage**: Use key `'pomoday_state'` for persistence.
- **Incremental edits**: Use precise `edit` calls. Never rewrite entire `index.html`.
- **Preserve functionality**: Existing features (timer, tasks, PWA install) must continue working.
- **Dexie.js exception**: Dexie is loaded via a CDN `<script>` in `<head>` because the no-bundler constraint makes a CDN the only viable way to include the IndexedDB wrapper.

## Working with CSS

### Where to Make Changes

All CSS is in the `<style>` block in `<head>`. Structure is:

1. **Reset and base** — `:root` variables, `html, body` base styles
2. **Header** — `header`, `.logo`, `.header-stats`, `.btn-new-day`
3. **Navigation** — `.tab-nav`, `.tab-nav-btn`
4. **Pages** — `.page`, `.timer-page`, `.tasks-page`, `.stats-page`, `.log-page`
5. **Timer components** — `.mode-tabs`, `.clock-container`, `.controls`
6. **Task components** — `.tasks-list`, `.task-item`, `.add-task`
7. **Stats/Log** — `.stats-grid`, `.daily-bar`, `.log-list`
8. **Responsive** — `@media (min-width: 700px)` at the end

### CSS Variables (Dynamic Values)

The app uses CSS custom properties for dynamic theming. These are updated by JS:

- `--mode-color`: Changes based on timer mode (work/short/long). Used for accents, active states.

If changing the color system, update both the `:root` defaults AND the JS that modifies these variables (search for `document.documentElement.style.setProperty`).

## Design Change Guidelines

1. **Read before editing** — Understand the current structure and which elements use which classes
2. **Check responsive behavior** — Test both mobile (default) and desktop (`@media (min-width: 700px)`)
3. **PWA considerations** — Changes to theme colors may need sync with `manifest.json` or `theme-color` meta tag
4. **Contrast** — Dark theme background is `--bg: #0d0d0d`. Ensure text remains readable.
5. **Safe areas** — Mobile UI should account for notched devices (see `env(safe-area-inset-*)` usage)

## What to Preserve

- **State management logic** — The `state` object, localStorage functions, event dispatch patterns
- **Timer functionality** — Clock SVG, progress calculations, mode switching
- **Task CRUD** — Add, toggle, delete task operations
- **PWA essentials** — Manifest link, service worker registration, meta tags

## What Can Be Freely Changed

- Colors, fonts, spacing, borders, shadows
- Layout and sizing
- Animation timing and easing
- Button and card styling
- Typography scale and weights
