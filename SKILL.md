---
name: marimo-pair
description: >-
  Work inside a running marimo notebook's kernel — execute code, create cells,
  and build a notebook as an artifact. Use when the user wants to start a
  marimo notebook or work in an active marimo session.
allowed-tools: Bash(bash **/scripts/discover-servers.sh *), Bash(bash **/scripts/execute-code.sh *), Read
---

# marimo Pair Programming Protocol

This skill gives you full access to a running marimo notebook. You can read
cell code, create and edit cells, install packages, run cells, and inspect
the reactive graph — all programmatically. The user sees results live in their
browser while you work through bundled scripts or MCP.

## Philosophy

marimo notebooks are a dataflow graph — cells are the fundamental unit of
computation, connected by the variables they define and reference. When a cell
runs, marimo automatically re-executes downstream cells. You have full access
to the running notebook.

- **Cells are your main lever.** Use them to break up work and choose how and
  when to bring the human into the loop. Not every cell needs rich output —
  sometimes the object itself is enough, sometimes a summary is better.
  Match the presentation to the intent.
- **Understand intent first.** When clear, act. When ambiguous, clarify.
- **Follow existing signal.** Check imports, `pyproject.toml`, existing cells,
  and `dir(ctx)` before reaching for external tools.
- **Stay focused.** Build first, polish later — cell names, layout, and styling
  can wait.

## Column Discipline

For users who care about notebook presentation, column structure is a primary
constraint. Treat it as part of the notebook's architecture, not as incidental
styling.

- Before editing, inspect the notebook's current column layout and preserve it
  unless the user explicitly wants a redesign.
- When adding cells, decide whether the new content belongs in an existing
  column or starts a new one.
- When working in a notebook that already uses columns, do not casually insert
  cells in ways that collapse or scramble the visual story.
- When building a notebook from scratch, prefer an intentional multicolumn
  structure when the notebook mixes narrative, controls, summaries, and plots.
- Assume spacer cells, hidden markdown, and summary blocks may be deliberate
  layout anchors.

### Technical model you must follow

marimo's live layout is reconstructed from flat cell order plus sparse
`config.column` anchors:

- `marimo.App(width="columns")` enables true multicolumn rendering.
- The first cell of a visual column typically carries explicit `column=N`.
- Cells with `column=None` inherit the previous cell's column.
- Reordering cells can therefore change layout even if column metadata is
  unchanged.
- Saved notebook configs are typically sparse: only column-boundary cells need
  explicit markers.

When mutating a live notebook, reason about both:

- cell order
- column anchors

Do not treat one without the other.

## Quickstart for New Notebooks

When setting up a new notebook, default to a deliberate left-to-right structure
instead of building one long column and cleaning it up later.

1. Create a setup cell at the top of the first column. Put `import marimo as mo`
   there along with the notebook's other imports, configuration, and shared
   constants.
2. Put data loading directly below the setup cell in the first column.
3. Put helper functions, classes, and other reusable definitions in their own
   cells below the data-loading cells, still in the first column.
4. Reserve the last column as empty breathing room. In a new three-column
   notebook, add a markdown cell in the third column whose content is exactly
   `leave space`.
5. Do not put anything else in the last column unless the user explicitly wants
   a different layout.
6. Put displays, exploratory outputs, and heavier or more situational
   computations in the center columns only: the second through second-to-last
   columns.
7. Keep each center column bounded. A column should contain at most a few
   screens' worth of cells before you start a new column or simplify the
   presentation.

This structure keeps the notebook readable:

- first column = durable foundations
- center columns = analysis and presentation
- last column = visual margin

Use this as the default notebook design process unless the existing notebook or
the user's request clearly calls for something else.

## Prerequisites

### How to invoke marimo

Only servers started with `--no-token` register in the local server registry
and are auto-discoverable — starting without a token makes discovery easier.
If a server has a token, set the `MARIMO_TOKEN` environment variable before
calling the execute script (avoids leaking the token in process listings). The
right way to invoke marimo depends on context (project
tooling, global install, sandbox mode). See
[finding-marimo.md](reference/finding-marimo.md) for the full decision tree.

**Do NOT use `--headless` unless the user asks for it.** Omitting it lets
marimo auto-open the browser, which is the expected pairing experience. If the
user explicitly requests headless, offer to open it with
`open http://localhost:<port>`.

## Troubleshooting

### `SyntaxError` or `ImportError` from `execute-code.sh`

Code runs **inside the running marimo kernel** — `execute-code.sh` POSTs it
over HTTP and never invokes a local Python. So errors here are not caused by
the local Python version, missing venv, or `uv` vs `pip` — they're problems
with the code being sent. Fix the code (use a heredoc for anything
multiline; don't try to one-line compound statements with `;`).

### User keeps getting prompted to allow Bash commands

The skill declares `allowed-tools` in its frontmatter, but Claude Code may
still prompt for each Bash call. To fix this, the user should add the absolute
paths to the scripts to their `.claude/settings.json` (project-level) or
`~/.claude/settings.json` (global):

```json
{
  "permissions": {
    "allow": [
      "Bash(bash /absolute/path/to/skills/marimo-pair/scripts/discover-servers.sh *)",
      "Bash(bash /absolute/path/to/skills/marimo-pair/scripts/execute-code.sh *)"
    ]
  }
}
```

## How to Discover Servers and Execute Code

Two operations: **discover servers** and **execute code**.

| Operation | Script | MCP |
|-----------|--------|-----|
| Discover servers | `bash scripts/discover-servers.sh` | `list_sessions()` tool |
| Execute code | `bash scripts/execute-code.sh -c "code"` | `execute_code(code=..., session_id=...)` tool |
| Execute code (multiline) | `bash scripts/execute-code.sh <<'EOF'` | same |
| Execute code (by URL) | `bash scripts/execute-code.sh --url http://localhost:2718 -c "code"` | same (with `url` param) |

Scripts auto-discover sessions from the local server registry. Use
`--port` to target a specific server when multiple are running,
`--session` to target a specific session when multiple notebooks are
open on the same server, or `--url` to skip discovery and connect to a
server by URL (e.g. `--url http://localhost:2718`). Set the
`MARIMO_TOKEN` env var to authenticate when the server has token auth
enabled (`--token` flag also works but exposes the token in process
listings). If the server was started with `--mcp`, you'll have MCP tools
available as an alternative.

### Discovery finds nothing but the user has a server running?

Only `--no-token` servers are in the registry. If discovery comes up empty,
the server likely has token auth — ask the user for the token and set it as
the `MARIMO_TOKEN` environment variable.

### No servers running?

**Always discover before starting.** Background task "completed" notifications
do not mean the server died — check the output or run discover first.

If no servers are found, read the user's intent — if they want a notebook,
start one. **Always start marimo as a background task** (using
`run_in_background` on the Bash tool) so the server automatically gets cleaned
up when the session ends and doesn't block the conversation. See
[finding-marimo.md](reference/finding-marimo.md).

If there's no `.py` file yet, pick a descriptive filename based on context
(e.g., `exploration.py`, `analysis.py`, `dashboard.py`). Don't ask — just
pick something reasonable.

**Avoid shell escaping issues.** `-c` works for simple one-liners, but for
multiline code or code with quotes/backticks/`${}`, use a heredoc or a file:

```bash
# heredoc (single-quoted delimiter prevents shell interpolation)
bash scripts/execute-code.sh <<'EOF'
import marimo._code_mode as cm

async with cm.get_context() as ctx:
    ctx.create_cell("x = 1")
EOF

# file
bash scripts/execute-code.sh /tmp/code.py

# target a specific port (skips auto-selection when multiple servers run)
bash scripts/execute-code.sh --port 2718 -c "1 + 1"
```

## Executing Code

Every execute-code call runs inside the notebook's kernel. All cell variables
are in scope — `print(df.head())` just works. Nothing you define persists
between calls (variables, imports, side-effects all reset), but you can freely
introspect the notebook: inspect variables, test code snippets, check types
and shapes. Use this to explore, prototype, and validate before committing
anything to the notebook — then create cells to persist state and make results
visible to the user.

To mutate the notebook's dataflow graph — create, edit, and delete cells,
install packages, and run cells — use `marimo._code_mode`:

```python
import marimo._code_mode as cm

async with cm.get_context() as ctx:
    cid = ctx.create_cell("x = 1")
    ctx.install_packages("pandas")
    ctx.run_cell(cid)
```

You **must** use `async with` — without it, operations silently do nothing.
All `ctx.*` methods are **synchronous** — they queue operations and the
context manager flushes them on exit. Do **not** `await` them.

The kernel supports top-level `await`, so use `async with` directly. Do
**not** wrap calls in `async def main(): ...` + `asyncio.run(main())` — it's
unnecessary and easy to get wrong (compound statements like `async with`
can't follow `def name():` on the same line, so cramming it into a `-c`
one-liner produces a `SyntaxError`).

**Cells are not auto-executed.** `create_cell` and `edit_cell` are structural
changes only — use `run_cell` to queue execution.

`code_mode` is a tested, safe API for notebook mutations — prefer it for all
structural changes. You also have access to marimo internals from the kernel,
but treat that as a last resort and only with high confidence after exploration.

### Saving Notebook Changes

`ctx.create_cell(...)` and `ctx.edit_cell(...)` update the **live notebook
session**, but they do **not** necessarily persist those edits back to the
`.py` file immediately. If the task requires the notebook file on disk to
change, perform an explicit save step after your edits.

The most reliable save path from inside the kernel is to build a
`SaveNotebookRequest` from the live `ctx.cells` and hand it to
`AppFileManager`:

```python
import marimo._code_mode as cm
from marimo._server.models.models import SaveNotebookRequest
from marimo._session.notebook.file_manager import AppFileManager

async with cm.get_context() as ctx:
    manager = AppFileManager(ctx.globals["__file__"])
    request = SaveNotebookRequest(
        cell_ids=[cell.id for cell in ctx.cells],
        codes=[cell.code for cell in ctx.cells],
        names=[cell.name for cell in ctx.cells],
        configs=[cell.config for cell in ctx.cells],
        filename=ctx.globals["__file__"],
        layout=None,
        persist=True,
    )
    manager.save(request)
```

Important details:

- Use the code from `ctx.cells` when saving; that is the authoritative live
  notebook state.
- `cell.code` is the raw notebook cell body. Do **not** include generated
  wrapper code or notebook-file `return` statements when calling `edit_cell`.
- A direct HTTP call to `/api/files/save` may fail with `401` unless you also
  reproduce the frontend's authenticated session. Prefer the in-kernel
  `AppFileManager(...).save(...)` path.
- When column layout matters, preserve the live `cell.config` values from
  `ctx.cells` when saving. Do not rebuild configs loosely or you may strip the
  notebook's intended column anchors.

**UI state lives outside the reactive graph.** Anywidget traitlets can be read
or set directly (e.g., `slider.value = 5`). For `mo.ui.*` elements, use
`ctx.set_ui_value(element, new_value)` inside `code_mode`.

### First Step: Explore the API

The `code_mode` API can change between marimo versions — and each running
server could be a different version. Inspect what's available at the start of
each session, especially when switching between servers.

```python
import marimo._code_mode as cm

async with cm.get_context() as ctx:
    ctx  # inspect me — dir(), help(), .cells, ...
```

When layout matters, explicitly inspect column structure early. For example:

```python
import marimo._code_mode as cm

async with cm.get_context() as ctx:
    [
        {
            "id": cell.id,
            "name": cell.name,
            "column": getattr(cell.config, "column", None),
        }
        for cell in ctx.cells
    ]
```

Use this to identify:

- explicit column anchors
- runs of inherited cells
- empty or hidden cells that are acting as layout separators

If you are about to insert or reorder cells, inspect this first.

## Guard Rails

Skip these and the UI breaks:

- **Install packages via `ctx.install_packages()`, not `uv add` or `pip`.**
  The code API handles kernel restarts and dependency resolution correctly.
  Only fall back to external CLIs if the API is unavailable or fails.
- **Custom widget = anywidget.** For bespoke visual components, use anywidget
  with HTML/CSS/JS. Composed `mo.ui` is fine for simple forms and controls.
  See [rich-representations.md](reference/rich-representations.md).
- **NEVER write to the `.py` file directly while a session is running — the kernel owns it.**
- **No temp-file deps in cells.** `pathlib.Path("/tmp/...")` in cell code is a bug.
- **Avoid empty cells.** Prefer `edit_cell` into existing empty cells rather
  than creating new ones. Clean up any cells that end up empty after edits.
- **Don't worry about cell names.** Most cells don't need explicit names —
  see [notebook-improvements.md](reference/notebook-improvements.md#cell-names).
- **Do not destroy column anchors accidentally.** If a cell is the first cell in
  a visual column, moving or deleting it can regroup many downstream cells.
- **Do not assign `column=` mechanically to every new cell.** In most cases,
  inheriting the surrounding column is the correct behavior.
- **Do not create a new column with a markdown cell anchor.** Use a Python cell
  as the first cell in that column.
- **Do not reorder cells without checking inherited column effects.** Flat order
  is part of the layout model.

## Editing Workflow When Columns Matter

Use this workflow when the notebook already has columns or should gain them.

1. Inspect `ctx.cells` and record each cell's `id`, `name`, and `config.column`.
2. Identify which cells are explicit column anchors and which cells inherit.
3. Decide whether your new content belongs:
   in an existing column,
   as a new anchor starting a new column,
   or as a replacement for an existing anchor/spacer cell.
4. Prefer editing an existing empty or placeholder cell inside the target
   column before creating a new cell.
5. If you create a new cell inside an existing column, usually leave its column
   unset so it inherits naturally.
6. If you create a new visual column, make the first cell a Python cell with
   explicit `column=N`.
7. After mutations, re-inspect the live cell list and confirm the anchors still
   imply the intended layout.
8. Save using the live `ctx.cells` configs.

When the user specifically values layout, this verification is required, not
optional.

## Widgets and Reactivity

Anywidget state (traitlets) lives outside marimo's reactive graph. To hook a
widget trait into the graph, pick one strategy per widget — never mix them:

- **`mo.state` + `.observe()`** — you pick specific traits to bridge. Default choice.
- **`mo.ui.anywidget()`** — wraps all synced traits into one reactive `.value`. Convenient but coarser.

Read [rich-representations.md](reference/rich-representations.md) before wiring either.

## Keep in Mind

- **The user is editing too.** The notebook can change between your calls —
  re-inspect notebook state if it's been a while since you last looked.
- **Deletions are destructive.** Deleting a cell removes its variables from
  kernel memory — restoring means recreating the cell and re-running it and
  its dependents. If intent seems ambiguous, ask first.
- **Installing packages changes the project.** `ctx.install_packages()` adds
  real dependencies — confirm when it's not obvious from context.

## References

- [finding-marimo.md](reference/finding-marimo.md) — how to find and invoke the right marimo
- [gotchas.md](reference/gotchas.md) — cached module proxies and other traps
- [rich-representations.md](reference/rich-representations.md) — custom widgets and visualizations
- [notebook-improvements.md](reference/notebook-improvements.md) — improving existing notebooks
