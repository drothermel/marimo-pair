# Gotchas

## Private variables are cell-scoped

Variables with a `_` prefix are **private to the cell that defines them** in
marimo. They cannot be referenced from other cells — you'll get a `NameError`.

This matters when building notebooks programmatically. A common mistake:

```python
# Cell A
_df = pd.DataFrame(results)   # _df is private to this cell

# Cell B — FAILS
mo.ui.table(_df)               # NameError: name '_df' is not defined
```

**Fix:** Either merge both into one cell, or use a non-private name (`df`).

## Redefining a public name across cells

Each public name has one owning cell. Defining it again in another cell fails
with `Multiply-defined names`. This is easy to hit when building a notebook
incrementally — a second cell reassigns `df`, `results`, `data`, etc.

```python
# Cell A
df = pd.read_csv("data.csv")

# Cell B — FAILS: df already defined in Cell A
df = df.dropna()               # Multiply-defined names: df
```

**Fix — pick one:**

- **Edit the owning cell** if the step belongs there (`ctx.edit_cell`).
- **Use a new name** when later cells need the result (`clean = df.dropna()`).
- **Use a private `_` name** for a throwaway intermediate (`_clean = df.dropna()`).

`ctx.graph.cells[cid].defs` shows what a cell already owns.

## Duplicate public imports across cells

The same single-definition rule applies to imports: a public name (like `pd`)
can only be defined in one cell. If two cells both `import pandas as pd`, you
get a `Multiply-defined names` error at validation.

**Fix:** Use a `_` prefix on the second import (`import pandas as _pd`) or
consolidate imports into a shared cell.

## `inspect.getsource()` on methods is indented

`inspect.getsource()` on a class method preserves the original indentation.
Passing this to `ast.parse()` fails with `IndentationError`.

```python
# FAILS
src = inspect.getsource(SomeClass.some_method)
tree = ast.parse(src)  # IndentationError: unexpected indent

# FIX
import textwrap
src = textwrap.dedent(inspect.getsource(SomeClass.some_method))
tree = ast.parse(src)
```

## Cached module availability

Some libraries cache optional-dependency availability at import time. Installing
a package mid-session via `ctx.packages.add()` won't update those caches.
The user may need to restart the kernel — but try known workarounds first.

### Polars + pyarrow

`df.to_pandas()` fails with `ModuleNotFoundError: pa.Table requires 'pyarrow'`.

**Workaround** — if this error occurs after installing pyarrow mid-session,
run the following via `execute-code` (scratchpad), NOT in a cell. The patch
mutates the cached module object in the running kernel, so it doesn't need to
persist in the notebook.

```python
import pyarrow as _pa
import polars.dataframe.frame as _frame_mod
_frame_mod.pa = _pa
```

Then re-run the failing cell.

## `create_cell` without `name=` produces unparsable cells on save

When you call `ctx.create_cell(code)` without passing `name=`, the new cell's
name is empty. The live session is fine, but on save the codegen tries to emit
`def ():` (no identifier), fails with `invalid syntax`, and falls back to
writing the cell as `app._unparsable_cell(r"""...""")` instead of a normal
`@app.cell` function.

You'll see warnings like:

```
Generated code for cell  is invalid, falling back to unparsable cell.
Error: invalid syntax (<unknown>, line 2)
```

**Always pass `name="_"`** (or another valid identifier) when creating cells:

```python
ctx.create_cell(code, after="PKri", hide_code=True, name="_")
```

**Recovery if it already happened:** marimo can re-parse `_unparsable_cell`
blocks on session reload, but **every cell ID changes** in the process, so any
cell IDs you captured earlier in the session become stale. Re-inspect
`ctx.cells` to get the new IDs before further edits, then re-save with the
recovered live state.

Symptoms of operating on stale IDs after a reload:

```
KeyError: "Cell 'ZOWR' not found in notebook or pending adds"
```
