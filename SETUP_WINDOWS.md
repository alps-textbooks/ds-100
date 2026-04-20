# Windows Local Book Setup

This repo's checked-in [environment.yml](./environment.yml) is pinned for macOS.
On Windows, the most reliable way to render the textbook locally is to create a
Conda environment from the repo-compatible
[environment-windows.yml](./environment-windows.yml).

These steps are for validating that the textbook renders correctly with Jupyter
Book, MyST, and Sphinx. By default, the book build does **not** execute
notebooks because [`content/_config.yml`](./content/_config.yml) sets:

```yaml
execute:
  execute_notebooks: "off"
```

That is enough to verify:

- MyST roles and directives
- figures and captions
- cross-references
- TOC structure
- theme/layout/static assets

## 1. Start clean

Open PowerShell in the repo root:

```powershell
cd D:\Work\ds-100
```

If your current `textbook-win` environment has already accumulated dependency
conflicts, remove it before recreating it:

```powershell
conda deactivate
conda env remove -n textbook-win -y
```

## 2. Create the repo-compatible environment

Create the environment directly from the pinned Conda file:

```powershell
conda env create -f environment-windows.yml
conda activate textbook-win
```

This environment tracks the major versions from the repo's original
[`environment.yml`](./environment.yml), but removes macOS-specific build pins so
Conda can resolve Windows-compatible packages. It explicitly includes
`setuptools`, because `myst_nb -> jupyter_cache` imports `pkg_resources` from
that package during book startup.

## 3. Verify the critical packages

The build depends on `pkg_resources`, which comes from `setuptools`. Verify that
it is available along with the repo's expected Jupyter Book stack:

```powershell
python -c "import pkg_resources, sphinx, myst_nb, myst_parser, jupyter_book; print('pkg_resources ok'); print('sphinx', sphinx.__version__)"
```

Expected Sphinx version:

```text
4.5.0
```

## 4. Build the book

Run:

```powershell
jupyter-book build content
```

If the build succeeds, the output will be written to:

[`content/_build/html`](./content/_build/html)

## 5. Preview the rendered book locally

Serve the built HTML:

```powershell
cd content\_build\html
python -m http.server --bind 127.0.0.1 8000
```

Open:

```text
http://127.0.0.1:8000
```

## 6. Edit notebooks while previewing

In a second PowerShell window:

```powershell
conda activate textbook-win
cd D:\Work\ds-100
jupyter lab
```

Edit notebooks under [`content/ch`](./content/ch), save them, then rebuild:

```powershell
cd D:\Work\ds-100
jupyter-book build content
```

Refresh the browser after each rebuild.

## 7. If you want to validate notebook execution too

The environment file already includes the common runtime/scientific packages
used by this repo. If you also want to validate code execution, temporarily
change [`content/_config.yml`](./content/_config.yml):

```yaml
execute:
  execute_notebooks: force
```

Then clean and rebuild:

```powershell
jupyter-book clean content
jupyter-book build content
```

This is stricter and may fail on runtime/data issues that are unrelated to
formatting.

## Common failure modes

### `sphinxcontrib.applehelp ... needs at least Sphinx v5.0`

Cause: a newer transitive `sphinxcontrib-*` package was installed into an older
Jupyter Book environment.

Fix: recreate the environment from
[`environment-windows.yml`](./environment-windows.yml).

### `No module named 'pkg_resources'`

Cause: `setuptools` is missing or broken in the environment.

Fix:

```powershell
conda activate textbook-win
conda install setuptools=67.8.0 wheel=0.38.4 -y
```

If that happened after multiple ad-hoc installs, recreating the environment is
safer than continuing to patch it.

### `lxml.html.clean module is now a separate project`

Cause: `nbconvert==6.5.3` from this older Jupyter Book stack expects the older
`lxml` layout, but a newer `lxml` was installed transitively.

Fix: recreate the environment from
[`environment-windows.yml`](./environment-windows.yml). It pins
`lxml=4.9.3`, which is compatible with this render stack.

### `ResolutionImpossible` mentioning `jupyter-cache` and `SQLAlchemy`

Cause: packages were installed manually with `pip` on top of the pinned Conda
environment, producing a dependency set that differs from the repo-compatible
stack.

Fix: recreate the environment from
[`environment-windows.yml`](./environment-windows.yml) instead of mixing in
manual `pip` upgrades.

### Build uses the wrong TOC

The standard book build uses [`content/_toc.yml`](./content/_toc.yml), not the
root-level [`_toc.yml`](./_toc.yml).
