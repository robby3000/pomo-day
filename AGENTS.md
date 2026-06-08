# Core Constraints

- Stack: Single-file index.html only.
- No external files (.css, .js).
- CSS inside head <style>. JS inside body bottom <script>.
- Tech: Pure vanilla HTML5/CSS3/JS. Zero external frameworks, CDNs, or bundlers.
- State: Single global `appState` object. Persist via localStorage. Do not break existing schema.
- Edits: Use precise, incremental line changes. Never rewrite whole file.
- Style: Direct, non-patronizing, zero pleasantries.