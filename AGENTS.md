# This project

A bilingual (Japanese main / English secondary) Quarto blog built with
[babelquarto](https://github.com/ropensci-review-tools/babelquarto), published
to Quarto Pub at https://uchidamizuki.quarto.pub/blog.

## Writing a new post

- Japanese (main language): `posts/YYYY/MM/slug.qmd`
- English translation: `posts/YYYY/MM/slug.en.qmd`
- A post can exist in only one language; babelquarto simply omits it from the
  other language's listing.

## Rendering and publishing

All rendering happens **locally**, not in CI. CI (`.github/workflows/publish.yml`)
only runs `quarto publish quarto-pub --no-render` against the already-committed
`docs/` folder. This is deliberate: CI previously re-ran
`babelquarto::render_website()` on a fresh Linux checkout, which required every
R/Python/Julia package this blog has ever used to be reinstalled there, and hit
environment-parity failures that were hard to reproduce and fix on Linux.

To publish changes:

```r
babelquarto::render_website()
```

then commit `docs/`, `_freeze/`, and the edited `posts/` files together, and
push to `main`.

## Known gotchas

### `_freeze/` cache is not always persisted by babelquarto

`babelquarto::render_website()` renders each language in a **temporary
directory** copy of the project, then copies only the rendered `docs/`
(or `docs/<lang>/`) output back — it never copies the regenerated `_freeze/`
cache back to the real project. Ordinarily this is invisible, because the next
render just finds the same (still-valid) cache already on disk and reuses it.

It becomes a problem when a post's freeze cache is genuinely missing or stale
(e.g. after renaming a `.qmd` file, or after editing its frontmatter/content).
`render_website()` will silently re-execute that post to fill the gap in its
own temp dir, then throw the regenerated cache away — so the *next* render
looks fine locally (fast, cache hit, because the temp-dir cache existed within
that one process) but the committed `_freeze/` on disk, and therefore CI/the
live site, is still missing it.

Symptom: CI fails with `Error in library(): there is no package called '...'`
on a specific post, even though that same post renders fine locally.

Fix: render that one post **directly in the real project directory**,
bypassing babelquarto's temp-dir indirection, so its freeze cache actually
lands in `_freeze/`:

```r
# main-language (ja) post — render in place directly
quarto::quarto_render("posts/YYYY/MM/slug.qmd", as_job = FALSE)

# .en.qmd post — babelquarto strips the language suffix before rendering,
# so temporarily rename it to the bare name, render, then move the
# resulting freeze dir to the .en-suffixed name:
#   1. rename slug.en.qmd -> slug.qmd (after moving any existing slug.qmd
#      and _freeze/.../slug/ aside)
#   2. quarto::quarto_render("posts/YYYY/MM/slug.qmd", as_job = FALSE)
#   3. rename slug.qmd back to slug.en.qmd
#   4. rename _freeze/.../slug/ -> _freeze/.../slug.en/
#   5. restore the original slug.qmd and its _freeze/.../slug/ from backup
```

Then run `babelquarto::render_website()` once more to regenerate correct
`docs/` output for the whole site using the now-valid cache, and verify with a
quick scan for `processing file:` in the render log (a cache hit produces no
such line — if it's still there after the second run, something is still
wrong).

### R `arrow` + Python `pyarrow` can't share a process on Windows

Loading R's `arrow` package and importing Python's `pyarrow` (e.g. via
`reticulate` from a chunk that uses `pandas.read_parquet`) in the *same*
process fails with a DLL symbol collision on Windows
(see apache/arrow#40073). It surfaces as an `ImportError: DLL load failed`
when importing `pyarrow.parquet`'s compiled `lib`, not necessarily a crash.

Workaround: run the Python/pyarrow side in an isolated subprocess via
`callr::r()`, e.g.:

```r
callr::r(function() {
  reticulate::use_python("D:/uchid/Documents/.virtualenvs/r-reticulate/Scripts/python.exe", required = TRUE)
  builtins <- reticulate::import_builtins(convert = FALSE)
  pd <- reticulate::import("pandas", convert = FALSE)
  df <- pd$read_parquet("some.parquet")
  as.character(builtins$repr(df))  # keep the return value simple/serializable
})
```

Returning simple values (strings, not live Python object proxies) avoids
serialization issues across the callr process boundary. See
`posts/2022/06/use-parquet-instead-of-csv.en.qmd` for a worked example, paired
with a plain (non-executed) ` ```python ` fence just above it so the reader
still sees idiomatic Python source.

### Julia

Use `juliaup default <channel>`, not `juliaup override set <channel>` — the
override only applies to the literal directory it's run in, and does not
propagate into babelquarto's temp-directory copies used for the English
render pass.

Julia packages must be installed into the **project-local** environment
(`Project.toml`/`Manifest.toml` at the repo root), not the global environment,
since QuartoNotebookRunner resolves packages relative to the project:

```
julia --project=. -e 'import Pkg; Pkg.add(["JuMP", "Ipopt"])'
```

### Python

The Python venv reticulate uses is pinned by a project-level `.Renviron`
(gitignored) via `RETICULATE_PYTHON`. Manage it with `uv`:

```
uv venv --python 3.12 <path from .Renviron>
uv pip install --python <path> -r requirements.txt
```
