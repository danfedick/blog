# blog.fedick.net

Daniel Fedick's long-form blog: post-quantum cryptography, PKI, secrets management, and Ethereum.

Built with [Hugo](https://gohugo.io/) (extended) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme, deployed to GitHub Pages by `.github/workflows/hugo.yaml` on every push to `main`. The resume site at [fedick.net](https://fedick.net) is separate and is not part of this repo.

## Clone

```bash
git clone --recurse-submodules <repo-url>
# or, in an existing clone:
git submodule update --init --recursive
```

## Add a post

1. Create `content/posts/<slug>.md` (the file name becomes the URL: `https://blog.fedick.net/posts/<slug>/`).
2. Start it with YAML front matter:

   ```yaml
   ---
   title: "Post title"
   date: 2026-10-08            # or 2026-10-08T09:00:00-04:00; future dates do not publish
   draft: false                # true keeps it out of the build
   description: "One or two sentences used for the summary, meta description, and social cards."
   tags: ["post-quantum", "pki"]
   # slug: custom-url-slug     # optional; overrides the file name in the URL
   ---
   ```

3. Write Markdown below the front matter. Fenced code blocks get syntax highlighting and a copy button.
4. Preview locally, then commit and push to `main`:

   ```bash
   hugo server -D        # http://localhost:1313, -D also shows drafts
   hugo --minify         # production build into public/
   ```

You can also scaffold a post with `hugo new content posts/<slug>.md`.

## Layout

| Path | Purpose |
| --- | --- |
| `hugo.yaml` | Site config (title, menu, PaperMod params, RSS) |
| `content/posts/` | Blog posts |
| `content/about.md` | About page |
| `static/CNAME` | Custom domain for GitHub Pages (`blog.fedick.net`) |
| `themes/PaperMod` | Theme, pinned as a git submodule |
| `.github/workflows/hugo.yaml` | Build and deploy to GitHub Pages (no secrets) |

## Feeds

- All posts: `https://blog.fedick.net/index.xml` (full post content)
- Per section or tag: `.../posts/index.xml`, `.../tags/<tag>/index.xml`

## Upgrading

- Hugo: bump `HUGO_VERSION` in `.github/workflows/hugo.yaml`.
- PaperMod: `cd themes/PaperMod && git fetch && git checkout <commit-or-tag>`, then commit the submodule change.
