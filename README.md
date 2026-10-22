# Chosen of the Harvest — Documentation

Source for the official documentation site of the **Chosen of the Harvest** Unity
template, published with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/)
to GitHub Pages.

**Live site:** <https://camenduru.github.io/ChosenOfTheHarvestDoc/>

## Local preview

```bash
python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS / Linux
source .venv/bin/activate

pip install -r requirements.txt
mkdocs serve
```

Then open <http://127.0.0.1:8000>.

## Build

```bash
mkdocs build --strict
```

The generated site is written to `site/` (git-ignored).

## Deploy

Pushing to `main` triggers `.github/workflows/deploy.yml`, which builds the site
in strict mode and publishes it to the `gh-pages` branch. Enable Pages in the
repository settings with the **Deploy from a branch → `gh-pages` / `root`**
source the first time.

To deploy manually:

```bash
mkdocs gh-deploy --force
```

## Editing

- All pages live under `docs/`.
- Navigation and theme configuration live in `mkdocs.yml`. Adding a page means
  adding it to the `nav:` list, otherwise `--strict` will warn about an
  unreferenced file.
- Screenshots live in `docs/assets/images/` and are referenced as
  `../assets/images/<name>.png` (adjust the relative depth per page).
