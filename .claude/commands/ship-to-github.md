---
description: Scan for secrets, push the code to GitHub, deploy a GitHub Page via Actions, screenshot the site, and write the README and repo About
argument-hint: [repo URL or owner/name — omit to reuse the existing origin]
allowed-tools: Bash, Read, Edit, Write, Glob, Grep, WebFetch, mcp__playwright__browser_navigate, mcp__playwright__browser_resize, mcp__playwright__browser_take_screenshot, mcp__playwright__browser_close
---

# Ship this project to GitHub

Target repository: **$1**

If that is empty, use the existing `git remote get-url origin`. If there is no
origin either, stop and ask which repo to use — do not invent one or create a
repo the user did not name.

Accept any of these forms and normalise them:
`https://github.com/OWNER/REPO`, `https://github.com/OWNER/REPO.git`,
`git@github.com:OWNER/REPO.git`, or bare `OWNER/REPO`.

Work through the phases in order. **The secret scan comes first** — the point of
scanning is to stop a secret from ever reaching GitHub, and a push cannot be
taken back: force-pushing over a leaked commit does not remove it from forks,
caches, or the events API. If anything is pushed before it is cleared, the only
correct advice is to rotate the credential.

---

## Phase 1 — Scan for sensitive data (blocking)

Scan the working tree, and every commit that is not yet on the remote.

```bash
# What is about to be published, and what is already tracked
git status --short
git diff --stat @{u}..HEAD 2>/dev/null || echo "no upstream yet — all commits are new"

# High-signal secret patterns
grep -rniE '(api[_-]?key|secret|passwd|password|passphrase|token|credential|private[_-]?key)[[:space:]]*[:=]' \
  --exclude-dir=.git --exclude-dir=node_modules . | head -40

# Provider-specific key shapes
grep -rniE '(AKIA[0-9A-Z]{16}|ghp_[A-Za-z0-9]{36}|github_pat_[A-Za-z0-9_]{22,}|sk-[A-Za-z0-9]{20,}|sk-ant-[A-Za-z0-9-]{20,}|xox[baprs]-[A-Za-z0-9-]{10,}|AIza[0-9A-Za-z_-]{35}|-----BEGIN [A-Z ]*PRIVATE KEY-----)' \
  --exclude-dir=.git --exclude-dir=node_modules . | head -40

# Files that should essentially never be committed
find . -path ./.git -prune -o \( \
  -name '.env*' -o -name '*.pem' -o -name '*.key' -o -name '*.p12' -o -name '*.pfx' \
  -o -name '*.keystore' -o -name 'id_rsa*' -o -name 'id_ed25519*' \
  -o -name '.npmrc' -o -name '.netrc' -o -name 'credentials*' \
  -o -name '*.sqlite' -o -name '*.db' -o -name '.DS_Store' \) -print 2>/dev/null

# Personal data worth a second look before it goes public
grep -rniE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}' \
  --exclude-dir=.git --exclude-dir=node_modules . | head -20
```

Rules for triaging the hits:

- Distinguish a **real** secret from a placeholder (`YOUR_KEY_HERE`,
  `user@example.com`), a variable name, or a doc string. Report only what is real
  — a wall of false positives trains the user to ignore the scan.
- A real secret is a **hard stop**. Do not push. Tell the user exactly which file
  and line, and that the credential should be rotated if it was ever committed.
  Offer to move it to an untracked config file or an environment variable.
- Add junk files (`.DS_Store`, `node_modules/`, `.env`) to `.gitignore` and
  `git rm --cached` them if already tracked.
- Real personal data (the user's own email, an internal hostname) is a
  judgment call: surface it, say the repo is about to be public, and let the
  user decide. Do not silently strip it.

Ask for confirmation before continuing if anything at all is unresolved.

## Phase 2 — Upload the code

```bash
git init -q                        # only if not already a repo
git add -A
git commit -m "<a real message describing the change>"
git branch -M main
git remote add origin <normalised URL>    # or: git remote set-url origin <URL>
git push -u origin main
```

Notes that save a debugging round-trip:

- **The repo must already exist on GitHub and be empty** (no auto-created
  README), or the push conflicts. If `gh` is installed, `gh repo create
  OWNER/REPO --public --source=. --remote=origin` handles both at once.
- `Permission denied (publickey)` with a key sitting in `~/.ssh` usually means
  the key is **passphrase-protected and not loaded into ssh-agent** — ssh
  silently skips it and reports the same error as an unregistered key. Check
  with `ssh-add -l`; if it says "The agent has no identities", ask the user for
  the passphrase and load it with `ssh-add --apple-use-keychain ~/.ssh/<key>`
  (macOS), so it persists. Confirm with `ssh -T git@github.com`, which prints
  the account name on success.
- If the remote has commits you do not (someone edited via the web UI),
  `git fetch && git rebase origin/main` — never force-push over their work.
- Never write a credential into a file, a remote URL, or a commit. If a
  temporary askpass helper is needed, delete it in the same command.

## Phase 3 — GitHub Page via Actions

Write `.github/workflows/static.yml` (create it, or edit it in place if a Pages
workflow already exists — **do not add a second one**; two workflows deploying
on the same push race each other and one silently overwrites the other):

```yaml
name: Deploy static content to Pages
on:
  push:
    branches: ["main"]
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
concurrency:
  group: "pages"
  cancel-in-progress: false
jobs:
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: '.'          # narrow this if the repo has non-site files
      - id: deployment
        uses: actions/deploy-pages@v4
```

Adjust for the project: a build step (`npm ci && npm run build`) and
`path: './dist'` for a framework app; `path: '.'` only when the repo root really
is the site.

**Pages must be enabled once, and only the user can do it:**
Settings → Pages → Build and deployment → Source → **GitHub Actions**.
`actions/configure-pages` with `enablement: true` is documented to do this
automatically but fails in practice — the default `GITHUB_TOKEN` cannot create
the Pages site. Do not burn runs retrying it; send the user to
`https://github.com/OWNER/REPO/settings/pages` and wait.

Verify without needing repo-admin rights:

```bash
curl -s https://api.github.com/repos/OWNER/REPO | grep '"has_pages"'
curl -s "https://api.github.com/repos/OWNER/REPO/actions/runs?per_page=3" | \
  python3 -c "import sys,json;[print(r['name'],r['status'],r['conclusion']) for r in json.load(sys.stdin)['workflow_runs']]"
curl -s -o /dev/null -w '%{http_code}\n' https://OWNER.github.io/REPO/
```

`has_pages: false` means it is still not enabled. Note that job logs are **not**
readable without repo admin (`Must have admin rights to Repository`), so
diagnose from step conclusions and ask the user to paste the error if needed.

If the URL 404s and the account has a user site at `OWNER.github.io`, that 404 is
being served by the **user site**, not by this repo — the project page does not
exist yet. Once it does, `OWNER.github.io/REPO/` takes precedence automatically.
Nothing about the user site needs to change.

Then fix the 404s the site itself causes:

- **`/favicon.ico`** — every browser requests it. Add an inline data-URI icon so
  there is no extra file and no extra request. Prefer
  `data:image/svg+xml;base64,...` over raw SVG: raw SVG needs an
  `xmlns="http://www.w3.org/2000/svg"` attribute, which puts a literal `http://`
  into the source and trips "no external resources" audits.
- **`404.html`** at the site root, styled like the project. Resolve its home link
  at runtime (`location.pathname.split('/')[1]`) so it is correct under the
  `/REPO/` project-page base path, where a hardcoded `/` would leave the site.
- Any `src`/`href` pointing at a file that is not published — check
  `grep -noiE '(src|href)="[^"]*"'` against the deployed file list. Root-absolute
  paths (`/assets/x.png`) break on a project page; make them relative.

Confirm both: `/` returns 200, and a nonexistent path returns 404 with the
custom page.

## Phase 4 — Screenshot

Capture the deployed page with the Playwright MCP server so the README shows the
thing rather than only describing it. Take it from a local server, not the live
URL — the screenshot should match the code being pushed, and Pages can lag a
push by a minute or two.

```bash
mkdir -p docs                       # the screenshot lives here, out of the site root
python3 -m http.server 8765 &       # or the project's own dev server
```

Then, with the `playwright` MCP tools:

- `browser_navigate` to `http://127.0.0.1:8765/` — a headless browser cannot open
  `file://` URLs reliably, which is why the server is worth the extra step.
- `browser_resize` to `1440x900`, so the layout is the desktop one and the image
  is not a phone-width column.
- `browser_take_screenshot` with `fullPage: true` and an **absolute** `filename`
  (`/abs/path/to/repo/docs/screenshot.png`). A relative name is resolved against
  the MCP server's own working directory, which is not necessarily where you
  want the file — it can land in the repo root or somewhere outside it entirely.
  The target directory must already exist or the call fails with `ENOENT`, and
  if a stray copy does appear elsewhere, delete it before committing.
- Read the resulting PNG back and look at it. A blank page, a half-loaded
  layout, or an error toast is worse than no screenshot at all.

Afterwards: `browser_close`, stop the server, and keep the MCP server's scratch
output out of the repo — `.playwright-mcp/` belongs in `.gitignore`.

If the project has no visual UI (a CLI, a library), skip this phase rather than
screenshotting a terminal for the sake of it.

## Phase 5 — README

Create `README.md`, or edit the existing one rather than overwriting it — keep
any badges, licence text, or sections the user wrote. Base it on what the code
actually does; read the source and any `CLAUDE.md` first instead of guessing.

Cover: one-line description of what it is, a **live demo link** to the Pages URL,
the screenshot from Phase 4 (`![alt](docs/screenshot.png)`, right under the demo
link, with alt text that describes the interface for anyone who cannot see it),
how to run it locally, the notable
features/architecture, the tech stack, and any constraint a contributor would
otherwise break. Say plainly if it is a demo rather than a product.

## Phase 6 — Repo About

Set the description and the homepage to the Pages URL. This needs an
authenticated API call — `GITHUB_TOKEN` is not available locally.

```bash
gh repo edit OWNER/REPO \
  --description "<one-line summary>" \
  --homepage "https://OWNER.github.io/REPO/"
```

If `gh` is not installed (check with `which gh`; on macOS, `brew install gh`
then `gh auth login`), do not fake it. Give the user the exact values and the
link — `https://github.com/OWNER/REPO` → the ⚙ next to **About** → Description,
Website, and tick "Use your GitHub Pages website". Topics are worth adding too.

---

## Finish

Report, with no hedging about what was not verified:

- the live Pages URL, and the HTTP status you actually observed for it
- what the secret scan found, including "nothing" if that is the truth
- exactly which steps still need the user (enabling Pages, `gh auth login`,
  setting About by hand)
- whether the screenshot was captured and refreshed, or why it was skipped
- anything deliberately left alone

Respect any `CLAUDE.md` in the repo throughout — its constraints outrank the
generic advice above. In particular, do not add a build step, a dependency, or
an extra file to a project whose stated constraints forbid them.
