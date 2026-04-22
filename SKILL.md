---
name: marimo-pair
description: >-
  Create, edit, and pair inside marimo notebooks. Use when the user wants to
  write a marimo notebook file, start a notebook, or work inside a running
  notebook session.
allowed-tools: Bash(bash **/scripts/discover-servers.sh *), Bash(bash **/scripts/execute-code.sh *), Read
---

# marimo Pair Programming Protocol

This skill is the single marimo skill. Use it for both notebook-file authoring
and live-session pairing.

marimo notebooks are a dataflow graph: cells are the unit of computation, and
marimo re-executes downstream cells automatically. Treat notebook structure,
layout, and display flow as part of the notebook's meaning, not as cleanup to
do later.

## Philosophy

- **Cells are the main lever.** Use them to structure work, debugging, reuse,
  and presentation.
- **Understand intent first.** When clear, act. When ambiguous, clarify.
- **Follow existing signal.** Check imports, `pyproject.toml`, existing cells,
  `dir(ctx)`, and current layout before reaching for external tools.
- **Prefer the smallest correct change.** Preserve the existing cell graph,
  inspection flow, and layout unless the user wants a redesign.
- **Keep notebooks simple.** Show UI elements consistently, let reactivity do
  its job, and avoid control-flow scaffolding that fights the notebook model.

## Two Working Modes

Use the right mental model for the job:

- **Notebook file authoring** uses `app = marimo.App(...)` and `@app.cell(...)`
  in a `.py` notebook file.
- **Live notebook pairing** uses a running notebook session plus
  `marimo._code_mode` to inspect and mutate `ctx.cells`.

The same notebook may move between both modes, so keep the guidance consistent:

- file mode cares about saved `@app.cell(...)` structure and `column=...`
  anchors in the notebook source
- live mode cares about flat `ctx.cells` order plus `cell.config.column`
  anchors in the running session

Do not confuse one representation for the other.

## Quickstart for New Notebooks

When setting up a new notebook, default to an intentional multicolumn layout
instead of one long vertical stack.

- Default to `app = marimo.App(width="columns")` unless the notebook is truly a
  simple scratchpad.
- Create a setup cell at the top of the first column. Put `import marimo as mo`
  there along with the notebook's other imports, configuration, shared
  constants, and stable helpers that belong in setup.
- Put data loading directly below the setup cell in the first column.
- Put helper functions, classes, and other reusable definitions in their own
  cells below the data-loading cells, still in the first column.
- Reserve the last column as empty breathing room. In a new three-column
  notebook, add a markdown cell in the third column whose content is exactly
  `leave space`.
- Do not put anything else in the last column unless the user explicitly wants
  a different layout.
- Put displays, exploratory outputs, and heavier or more situational
  computations in the center columns only: the second through second-to-last
  columns.
- Keep each center column bounded. A column should contain at most a few
  screens' worth of cells before you start a new column or simplify the
  presentation.

The default visual model is:

- first column = durable foundations
- center columns = analysis and presentation
- last column = visual margin

Use a small number of intentional lanes rather than many weakly justified
columns.

## Column Discipline

For users who care about notebook presentation, column structure is a primary
constraint. Treat it as part of the notebook's architecture, not as incidental
styling.

- Before editing, inspect the notebook's current column layout and preserve it
  unless the user explicitly wants a redesign.
- When extending an existing notebook, decide whether the new content belongs
  in an existing column or starts a new one.
- When working in a notebook that already uses columns, do not casually insert
  cells in ways that collapse or scramble the visual story.
- Assume spacer cells, hidden markdown cells, and summary blocks may be
  deliberate layout anchors.

### Technical model you must follow

marimo's column model is sparse and order-dependent:

- `marimo.App(width="columns")` enables true multicolumn rendering.
- The first cell of a visual column typically carries explicit `column=N`.
- Cells with no explicit column inherit the previous cell's column.
- Reordering cells can therefore change layout even if column metadata is
  unchanged.
- Saved notebook configs are typically sparse: only column-boundary cells need
  explicit markers.

When mutating a notebook, reason about both:

- cell order
- column anchors

Do not treat one without the other.

### Editing rules for existing notebooks

- Preserve existing explicit column anchors whenever possible.
- Do not renumber columns just because a different numbering scheme looks
  cleaner.
- If you insert a cell into the middle of an existing column, usually leave it
  without `column=` so it inherits naturally.
- If you start a new visual column, make the first cell a Python cell with
  explicit `column=N`.
- If markdown is involved, remember that the first cell in a saved visual
  column should still be a Python cell.
- After editing, verify that prose, tables, widgets, and charts still read in
  the intended sequence.

## Running Notebooks

Use these commands when working with notebook files directly:

```bash
# Run as a script
uv run <notebook.py>

# Run interactively in the browser
uv run marimo run <notebook.py>

# Edit interactively
uv run marimo edit <notebook.py>

# Check notebook structure and common mistakes
uvx marimo check <notebook.py>
```

Run `marimo check` before handing notebook work back to the user.

## How to Discover Servers and Execute Code

Two operations: **discover servers** and **execute code**.

| Operation | Script | MCP |
|-----------|--------|-----|
| Discover servers | `bash scripts/discover-servers.sh` | `list_sessions()` tool |
| Execute code | `bash scripts/execute-code.sh -c "code"` | `execute_code(code=..., session_id=...)` tool |
| Execute code (multiline) | `bash scripts/execute-code.sh <<'EOF'` | same |
| Execute code (by URL) | `bash scripts/execute-code.sh --url http://localhost:2718 -c "code"` | same (with `url` param) |

Scripts auto-discover sessions from the local server registry. Use `--port` to
target a specific server when multiple are running, `--session` to target a
specific session when multiple notebooks are open on the same server, or
`--url` to skip discovery and connect directly.

Only servers started with `--no-token` register in the local server registry.
If discovery comes up empty, the server likely has token auth. Ask the user for
the token and set the `MARIMO_TOKEN` environment variable before calling the
execute script so the token does not leak in process listings.

**Always discover before starting.** Background-task completion messages do not
mean the server died. If no servers are found and the user wants a notebook,
start one as a background task so the session gets cleaned up automatically.

If there is no notebook file yet, pick a descriptive filename from context
instead of asking.

**Avoid shell escaping issues.** `-c` is fine for one-liners, but for multiline
code or code with quotes, backticks, or `${}`, use a heredoc or a file:

```bash
bash scripts/execute-code.sh <<'EOF'
import marimo._code_mode as cm

async with cm.get_context() as ctx:
    ctx.create_cell("x = 1")
EOF
```

## Writing Notebook Files

### Setup cell

Put notebook imports in a single setup cell so dependencies are obvious and
stable.

- Import stdlib, third-party, and project modules in the setup cell.
- Return imported modules and functions from setup, then pass them as cell
  arguments.
- Avoid scattering imports across many cells.

```python
@app.cell
def _():
    import marimo as mo
    import matplotlib.pyplot as plt
    from pathlib import Path
    return mo, plt, Path
```

### Script mode detection

Use `mo.app_meta().mode == "script"` to distinguish CLI execution from
interactive use:

```python
@app.cell
def _(mo):
    is_script_mode = mo.app_meta().mode == "script"
    return (is_script_mode,)
```

### UI and reactivity rules

- Show UI elements in both script and interactive modes. Change the data
  source, not the overall notebook structure.
- Do not guard cells with unnecessary `if` checks just to wait for dependencies.
  Let marimo's reactivity control execution.
- Do not use broad `try/except` blocks for normal control flow. Let errors
  surface unless you are handling a specific expected exception with a real
  recovery path.
- When a notebook already displays the object under test, prefer updating that
  existing final expression in place instead of splitting parsing and display
  into extra cells.

### Output and markdown rules

- Marimo only renders the final expression of a cell.
- Default to `@app.cell(hide_code=True)` for markdown-producing and
  UI-producing cells.
- Standalone markdown cells are fine.
- In multicolumn notebooks, markdown cells often serve a layout role as well as
  a content role. Preserve their placement carefully.

```python
@app.cell(hide_code=True)
def _(mo):
    mo.md("# Section Title\n\nShort description.")
    return
```

### Notebook hygiene

- Use underscore-prefixed loop variables when names should stay cell-private:
  `for _name, _model in items: ...`
- Prefer `pathlib.Path` over `os.path`.
- Use PEP 723 metadata when a notebook should be self-contained as a script:

```python
# /// script
# requires-python = ">=3.12"
# dependencies = [
#     "marimo",
# ]
# ///
```

### Local API docs

If you need a quick local API reference for a marimo function, inspect it from
Python:

```bash
uv run python -c "import marimo as mo; help(mo.ui.form)"
```

## Executing Code in a Running Notebook

Every execute-code call runs inside the notebook's kernel. All cell variables
are in scope, but scratch definitions you create in that request do not persist
between calls. Use execute-code to inspect, prototype, and validate before you
commit changes to the notebook's real cell graph.

To mutate the notebook's dataflow graph — create, edit, delete, install
packages, and run cells — use `marimo._code_mode`:

```python
import marimo._code_mode as cm

async with cm.get_context() as ctx:
    cid = ctx.create_cell("x = 1")
    ctx.install_packages("pandas")
    ctx.run_cell(cid)
```

You **must** use `async with`. Without it, operations silently do nothing.
All `ctx.*` methods are synchronous: they queue operations and the context
manager flushes them on exit. Do **not** `await` them.

The kernel supports top-level `await`, so use `async with` directly. Do not
wrap calls in `asyncio.run(...)`.

**Cells are not auto-executed.** `create_cell` and `edit_cell` are structural
changes only. Use `run_cell` to queue execution.

## Saving Notebook Changes

When you edit a running notebook through `code_mode`, updating the live session
is separate from persisting the `.py` file. Treat the live session and the
saved file as different states until save succeeds.

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

- Use the code from `ctx.cells`; that is the authoritative live notebook state.
- `cell.code` is the raw notebook cell body. Do not include generated wrapper
  code or notebook-file `return` statements when calling `edit_cell`.
- Do not assume a direct HTTP call to `/api/files/save` will work without the
  frontend's authenticated session.
- When column layout matters, preserve the live `cell.config` values from
  `ctx.cells` when saving.

## Inspect First

The `code_mode` API can change between marimo versions, and each running server
could be on a different version. Inspect what is actually available at the
start of each session.

```python
import marimo._code_mode as cm

async with cm.get_context() as ctx:
    ctx
```

When layout matters, inspect column structure early:

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

Use this to identify explicit anchors, runs of inherited cells, and empty or
hidden cells acting as layout separators.

## Editing Workflow When Columns Matter

Use this workflow when the notebook already has columns or should gain them.

1. Inspect the current cell list and record each cell's `id`, `name`, and
   column anchor.
2. Identify which cells are explicit anchors and which inherit.
3. Decide whether the new content belongs in an existing column, starts a new
   anchor, or replaces an existing spacer or placeholder cell.
4. Prefer editing an existing empty or placeholder cell before creating a new
   one.
5. If you create a new cell inside an existing column, usually leave its column
   unset so it inherits naturally.
6. If you create a new visual column, make the first cell a Python cell with an
   explicit `column=N`.
7. After mutations, re-inspect the live cell list and confirm the anchors still
   imply the intended layout.
8. Save using the live `ctx.cells` configs.

When the user specifically values layout, this verification is required.

## Widgets and Reactivity

Anywidget state lives outside marimo's reactive graph.

- For anywidget traitlets, you can read or set them directly.
- For `mo.ui.*` elements, use `ctx.set_ui_value(element, new_value)` inside
  `code_mode`.
- To bridge an anywidget trait into the reactive graph, pick one strategy per
  widget:
  - `mo.state` + `.observe()` when you want narrow explicit bridging
  - `mo.ui.anywidget()` when a single reactive `.value` is good enough

Read [rich-representations.md](reference/rich-representations.md) before wiring
either approach.

## Guard Rails

Skip these and the notebook breaks:

- Install packages via `ctx.install_packages()`, not `uv add` or `pip`, when
  you are mutating a live notebook session.
- Never write to the notebook `.py` file directly while a live session owns it.
- No temp-file dependencies in notebook cells. `pathlib.Path("/tmp/...")` in
  notebook code is usually a bug.
- Avoid empty cells. Prefer `edit_cell` into an existing empty cell rather than
  creating new ones.
- Do not destroy column anchors accidentally.
- Do not assign `column=` mechanically to every new cell.
- Do not reorder cells without checking inherited-column effects.
- Do not create a new visual column with a markdown cell anchor.

## Keep in Mind

- The user may be editing at the same time. Re-inspect notebook state if it has
  been a while since you last looked.
- Deletions are destructive. Deleting a cell removes its variables from kernel
  memory.
- Installing packages changes the project. Confirm when it is not obviously
  implied by the task.

## Troubleshooting

### `SyntaxError` or `ImportError` from `execute-code.sh`

Code sent through `execute-code.sh` runs inside the running marimo kernel. The
script does not invoke a local Python, so these errors usually mean the code
being sent is wrong, not that the local interpreter or venv is wrong.

Use a heredoc for multiline code. Do not try to compress compound statements
onto one shell line with `;`.

### User keeps getting prompted to allow Bash commands

The skill declares `allowed-tools` in its frontmatter, but some clients may
still prompt for each Bash call. Add the absolute script paths to the client's
allowlist if repeated prompts are a problem:

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

## References

- [finding-marimo.md](reference/finding-marimo.md) — how to find and invoke the right marimo
- [gotchas.md](reference/gotchas.md) — cached module proxies and other traps
- [rich-representations.md](reference/rich-representations.md) — custom widgets and visualizations
- [notebook-improvements.md](reference/notebook-improvements.md) — improving existing notebooks
