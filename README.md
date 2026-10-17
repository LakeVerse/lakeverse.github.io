# Portfolio site

A static Jekyll site for GitHub Pages — no custom plugins, so GitHub builds it automatically.

## Before publishing

1. Pick the name and set `title` in `_config.yml` (plus `github_username`, `linkedin_username`, optionally `email`).
2. Create the repo:
   - `<username>.github.io` → site lives at `https://<username>.github.io/`, leave `baseurl: ""`
   - any other name, e.g. `portfolio` → site lives at `https://<username>.github.io/portfolio/`, set `baseurl: "/portfolio"`
3. Push, then in the repo: **Settings → Pages → Build and deployment → Deploy from a branch → `main` / root**.

## Adding content

- **A project:** add a file to `_projects/`, copying the front matter of an existing one.
  `status` is one of `building`, `designing`, `shipped`, `planned` (labels set in `_config.yml`);
  `order` sorts within a group; `repo` shows a "View the code" link when filled in.
  Empty status groups are hidden, so "Shipped" appears once something moves there.
- **A post:** add `_posts/YYYY-MM-DD-title.md` with `title:` in front matter.

## Preview locally (optional)

```
bundle install
bundle exec jekyll serve
```
Then open http://localhost:4000.
