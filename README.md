# sunilburigen.com — Jekyll site

GitHub Pages builds this automatically on every push to `main`.

## Structure
- `index.html`     — homepage (plain static, passed through untouched)
- `writing.html`   — Field Notes index; auto-lists every post (no manual editing)
- `_layouts/post.html` — shared design for all articles
- `_posts/`        — one Markdown file per article
- `_config.yml`    — site config; permalink keeps article URLs at /posts/<slug>.html
- `CNAME`          — custom domain binding (do not delete)

## Add a new article
1. Create `_posts/YYYY-MM-DD-your-slug.md`
2. Add front matter (copy from the existing post) and write the body
3. `git add . && git commit -m "Add article: ..." && git push`
The index updates itself. URL becomes /posts/your-slug.html

## Preview locally (optional)
    gem install bundler jekyll
    jekyll serve
    # open http://localhost:4000
