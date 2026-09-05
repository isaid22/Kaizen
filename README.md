# KC's Notebook

A personal writing/notes site built with [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.

## Structure

- `content/posts/` — longer writing
- `content/notes/` — shorter notes
- `hugo.toml` — site config, menu, theme params
- `themes/PaperMod` — the theme, added as a **git submodule**

## Local workflow

Install Hugo (extended edition, v0.146.0 or newer — this project was built and tested with v0.150.1):

```bash
# macOS
brew install hugo

# Linux (apt often ships an old version — grab the .deb from
# https://github.com/gohugoio/hugo/releases instead if `hugo version`
# reports anything older than 0.146.0)
sudo apt install hugo

# Windows
choco install hugo-extended
```

Preview locally (includes drafts):

```bash
hugo server -D
```

Then open http://localhost:1313

New post:

```bash
hugo new content posts/my-post-title.md
```

New note:

```bash
hugo new content notes/my-note-title.md
```

Edit the file, set `draft = false` when ready, then commit and push. Hosting rebuilds automatically (see below).

## Deploying (free, via Cloudflare Pages)

1. Push this repo to a **new GitHub repo** (see step-by-step below).
2. Go to https://dash.cloudflare.com → **Workers & Pages** → **Create** → **Pages** → **Connect to Git**, and pick this repo.
3. Build settings:
   - Framework preset: **Hugo**
   - Build command: `hugo --gc --minify`
   - Build output directory: `public`
   - Environment variable: `HUGO_VERSION` = `0.150.1`
4. Important — submodules: in the Cloudflare Pages project settings, under **Settings → Builds & deployments**, make sure "Enable git submodules" is turned on (it's on by default for new projects, but check if the theme folder ever shows up empty on deploy).
5. Deploy. Every future `git push` to `main` triggers a new build automatically.
6. Add your own domain for free under **Custom domains** once you have one, or just use the free `*.pages.dev` URL Cloudflare gives you.

## Changing the theme later

Because the theme lives in `themes/PaperMod` as its own git submodule, swapping it out is just:

```bash
git submodule deinit themes/PaperMod
git rm themes/PaperMod
git submodule add --depth=1 https://github.com/<user>/<new-theme>.git themes/<new-theme>
```

then update `theme = '...'` in `hugo.toml`. Browse more themes at https://themes.gohugo.io/.
