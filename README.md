# VAHID.DEV — Personal Website

Static personal site for **Vahid**, built with [Hugo](https://gohugo.io/) and the
[hugo-bearcub](https://github.com/clente/hugo-bearcub) theme (a port of [Bear Blog](https://bearblog.dev/)).

> No JavaScript, no tracking, tiny pages. Just Markdown.

## Stack

- **Generator:** Hugo (standard binary — no Node, no Dart Sass).
- **Theme:** `themes/hugo-bearcub` as a git **submodule** (pinned to a commit, not vendored).
- **Hosting:** GitHub Pages (build via GitHub Actions — see `.github/workflows/deploy.yml`).
- **Domain:** `vahid.dev` (Cloudflare-managed DNS, Full SSL). `static/CNAME` pins the custom domain.

## Local development

```bash
hugo server          # live preview at http://localhost:1313
hugo --gc --minify   # production build into ./public
```

Requires Hugo >= 0.145.0.

## Project layout

```
hugo.toml              # site config + navigation (Home / Blog; RSS rendered by theme)
content/
  _index.md           # Home page (welcome + about)
  posts/
    _index.md         # Blog index
    *.md              # Blog posts (Markdown)
static/
  CNAME               # vahid.dev  (GitHub Pages custom domain)
  favicon.svg
themes/hugo-bearcub/  # git submodule
.github/workflows/
  deploy.yml          # GitHub Pages Actions deploy
```

## Adding a post

```bash
hugo new content posts/my-post.md
```

Edit the front matter (`title`, `date`, `description`) and write Markdown. Commit, push to a
feature branch, open a PR. After merge to `main`, GitHub Actions rebuilds and redeploys.

## Notes

- `generateSocialCard` is disabled (it needs the Hugo *extended* binary for image processing).
- `CODEOWNERS` requires `@Kazempour` review on all PRs.
- Cloudflare: keep the 4 `vahid.dev` A records → GitHub Pages IPs and `CNAME www →
  vahid-development.github.io`; SSL/TLS mode **Full**; orange-cloud after GitHub issues the cert.
