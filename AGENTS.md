# AGENTS.md

This is a GitHub Pages personal resume site built with Jekyll.

## Architecture

- **All content lives in `_config.yml`** — this is the single source of truth. Edit content here, not in `index.md`.
- **Theme:** `sproogen/resume-theme` loaded via `remote_theme` in `_config.yml`. No local theme files.
- **`index.md`** only sets `layout: default` — do not add content here.
- **`assets/main.scss`** imports the theme stylesheet. Custom styles go here if needed.
- **`docs/jobs/`** (PDF certificates) is gitignored — changes there won't commit.
- **`Gemfile.lock`** is gitignored.
- **`README.md`** is a GitHub profile README, not repo documentation.

## Commands

```bash
bundle install          # install deps (first time only)
bundle exec jekyll serve # local preview at http://localhost:4000
```

No lint, test, or typecheck setup exists — this is a static site.

## Deployment

GitHub Pages auto-deploys from the `main` branch. No CI workflows.

## Content editing

- `_config.yml` sections: `content` array defines resume sections (experience, education, publications, etc.).
- Each section has a `layout` field (`left`, `top-middle`, `list`, `text`).
- Images referenced in config must exist in `images/`.
- `favicon` config path must match a file in `images/`.
