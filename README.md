# Lab 5 environment

This environment provides tools for:

- Rendering a Markdown file to PDF (`pandoc` + `tectonic`)
- Building and publishing a documentation website (`mkdocs` + `mkdocs-material`)

## Setup

```bash
conda env create -f environment.yml
conda activate md2pdf
```

## Render Markdown to PDF

```bash
pandoc input.md -o output.pdf --pdf-engine=tectonic
```

## Build an MkDocs website

1. Initialise a new MkDocs site (skip this if `mkdocs.yml` already exists):

   ```bash
   mkdocs new .
   ```

   This creates an `mkdocs.yml` config file and a `docs/` folder. Put your
   Markdown pages inside `docs/` (start with `docs/index.md`).

2. Preview the site locally:

   ```bash
   mkdocs serve
   ```

   Open the printed URL (usually `http://127.0.0.1:8000`) in a browser.
   The page reloads automatically as you edit files in `docs/`.

3. (Optional) Use the Material theme by adding this to `mkdocs.yml`:

   ```yaml
   theme:
     name: material
   ```

## Push the MkDocs website to GitHub Pages

1. Create a GitHub repository and push your project to it, if you haven't
   already:

   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   git branch -M main
   git remote add origin https://github.com/<your-username>/<your-repo>.git
   git push -u origin main
   ```

2. Build and deploy the site to the `gh-pages` branch in one step:

   ```bash
   mkdocs gh-deploy
   ```

   This builds the site and pushes it to a `gh-pages` branch on the
   `origin` remote (it uses `ghp-import` under the hood).

3. On GitHub, go to **Settings → Pages** for your repository and confirm
   the source is set to the `gh-pages` branch. Your site will be published
   at:

   ```
   https://<your-username>.github.io/<your-repo>/
   ```

4. Any time you update the docs, rebuild and redeploy:

   ```bash
   mkdocs gh-deploy
   ```
