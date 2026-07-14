# S-Mailer Blog

Content for the S-Mailer blog, published at
[blog.mailer.smartek.co.mz](https://blog.mailer.smartek.co.mz).

## Layout

The Jekyll site lives in [`landing/`](landing), which keeps the repo root free
for anything else we publish later (docs, status, …). Posts go in
`landing/_posts/` as `YYYY-MM-DD-title.md`.

`landing/` is a build source, not a URL prefix — the site is still served from
the root of the domain, so no post URL changes.

## Local preview

```bash
cd landing
bundle install
bundle exec jekyll serve
```

## Deployment

Pushing to `main` runs [`.github/workflows/pages.yml`](.github/workflows/pages.yml),
which builds `landing/` and publishes it to GitHub Pages. The Pages source has to
be **GitHub Actions** (not "deploy from a branch"), because the built-in Jekyll
build can only read from the repo root or `/docs`.

© Smartek — MIT licensed (see [LICENSE](LICENSE)).
