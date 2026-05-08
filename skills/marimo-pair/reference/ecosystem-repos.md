# Ecosystem Repos

Use these repos as part of the normal exploration surface for marimo work.

## How to use this list

- Check the local sibling repo first when it exists in the shared `repos`
  directory.
- If the repo is not present locally, inspect the GitHub repo and clone it into
  the shared `repos` directory if deeper local exploration is useful.
- Treat these repos as sources of implementation ideas, reusable utilities,
  components, notebook patterns, and debugging clues.
- Do not assume a package is installed in the active notebook environment just
  because the source repo exists locally. Verify imports before taking a
  dependency on it.

## Core marimo source

### `marimo`

- Local: `../marimo`
- GitHub: `https://github.com/marimo-team/marimo`
- Use for:
  - source of truth for notebook internals
  - `code_mode` behavior
  - save semantics
  - debugging odd notebook behavior
- Reach for it when:
  - the running notebook behaves in a surprising way
  - an internal API seems unclear
  - you need to confirm how marimo actually implements something

## Local utility and component repos

### `marimo_utils`

- Local: `../marimo_utils`
- GitHub: `https://github.com/drothermel/marimo_utils`
- Use for:
  - reusable helpers
  - notebook utilities
  - project-specific patterns you already own
- Reach for it when:
  - a task sounds like something you've already solved before
  - you want to avoid rebuilding small utilities from scratch

### `dr_widget`

- Local: `../dr_widget`
- GitHub: `https://github.com/drothermel/dr_widget`
- Use for:
  - custom widgets
  - UI components
  - integration patterns for your own widget stack
- Reach for it when:
  - the task needs a richer interactive component
  - stock `mo.ui` is not enough
  - a debugging issue appears widget-related

## External companion libraries

### `mohtml`

- Local: `../mohtml`
- GitHub: `https://github.com/koaning/mohtml`
- Use for:
  - richer HTML composition
  - presentation patterns beyond stock notebook markdown
  - UI building ideas
- Reach for it when:
  - layout or rendering needs are pushing past basic `mo.md` or `mo.ui`
  - you want a cleaner compositional pattern for HTML-heavy output

### `wigglystuff`

- Local: `../wigglystuff`
- GitHub: `https://github.com/koaning/wigglystuff`
- Use for:
  - interactive components
  - visualization patterns
  - custom frontend ideas
- Reach for it when:
  - a notebook needs a more specialized interactive experience
  - you want examples of richer widget behavior

### `moterm`

- Local: typically clone into `../moterm` if needed
- GitHub: `https://github.com/koaning/moterm`
- Use for:
  - terminal-like interfaces
  - alternative notebook interaction patterns
- Reach for it when:
  - a task benefits from terminal-style interaction inside a notebook
  - you want examples of a nonstandard UI pattern

### `smartfunc`

- Local: typically clone into `../smartfunc` if needed
- GitHub: `https://github.com/koaning/smartfunc`
- Use for:
  - helper abstractions
  - reusable function patterns
  - implementation ideas for wrapping logic cleanly
- Reach for it when:
  - the task benefits from a reusable function pattern
  - you want an external example before designing a helper abstraction

## Examples and inspiration

### `gallery-examples`

- Local: typically clone into `../gallery-examples` if needed
- GitHub: `https://github.com/marimo-team/gallery-examples`
- Use for:
  - notebook examples
  - layout and feature inspiration
  - patterns for presenting marimo notebooks cleanly
- Reach for it when:
  - you want to compare approaches for a new notebook
  - you need examples of how a marimo feature is used in practice
