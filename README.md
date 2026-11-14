# Fabric in Practice

Personal blog built with [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme, deployed to GitHub Pages via GitHub Actions.

## Writing a post

1. Work on unpublished posts in `drafts/`. It's gitignored, so drafts never reach the public repo.
2. To publish, move the file to `content/posts/` and set `draft: false` in its front matter.
3. Push to `main`. The workflow in `.github/workflows/hugo.yml` builds and deploys the site.

## Local preview

```bash
hugo server -D
```

`-D` includes drafts. Open http://localhost:1313.

## First clone

The theme is a git submodule:

```bash
git clone --recurse-submodules <repo-url>
```
