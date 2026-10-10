# Live-stream redirect

A tiny static site on GitHub Pages. Viewers open one stable link. If the stream is **live**, they get
sent to the current YouTube URL. If it's **interrupted**, they see *"Stream interrupted, we'll be back
in a minute"* and the page checks again every 30 s, then redirects on its own once you're live again.

Vanilla HTML/JS, no build step. Uses only free GitHub Pages and GitHub Actions.

- **Repo:** https://github.com/riannom/nomura_html
- **Landing page:** https://nomura.network/ (also https://riannom.github.io/nomura_html/)
- **Viewer link:** https://nomura.network/COL/ (`/col/` redirects there; paths are case-sensitive)
- **Admin page:** https://nomura.network/COL/admin.html
- **No-token fallback:** https://github.com/riannom/nomura_html/actions/workflows/update-stream.yml (Run workflow)

| File | Purpose |
|---|---|
| `index.html` | Root landing page for nomura.network, with a button to `COL/`. |
| `COL/index.html` | Viewer page. Embeds the YouTube stream from `COL/stream.json` (polled every 30 s) and shows a configurable placeholder when not live (edit the `PLACEHOLDER` block at the top). |
| `COL/admin.html` | Streamer page. Takes a URL, a status and a token, then sends a `repository_dispatch` to the GitHub API. |
| `col/index.html` | Redirects lowercase `/col/` to `/COL/` (keeps the query string). |
| `COL/stream.json` | `{"url": "..." or "" (cleared), "status": "live"\|"interrupted", "updated": "<ISO time>"}` |
| `.github/workflows/update-stream.yml` | Checks the input, writes `COL/stream.json` with `jq`, commits it and deploys Pages (root `index.html`, `COL/`, `col/`, `.nojekyll`). |
| `CNAME.example` | Template for a custom domain. Rename it to `CNAME` once you pick a domain. |
| `.nojekyll` | Turns off Jekyll processing. |

## Tip: you may not need updates at all

YouTube's `https://www.youtube.com/channel/<CHANNEL_ID>/live` and `https://www.youtube.com/@<handle>/live`
always point to the channel's **current** live stream. If you restart under the same channel, set
`stream.json`'s `url` to one of those once. After that you only flip `status` between `live` and `interrupted`,
or you can just leave it on `live`.

## One-time setup

1. **Create the repo** (public is fine) and push these files to `main`.
2. **Set the Pages source to GitHub Actions**: *Settings → Pages → Build and deployment → Source: **GitHub Actions***.
   The workflow publishes the site itself. It doesn't use "Deploy from a branch".
3. **Run once**: *Actions → Update stream → Run workflow* (or push any commit). This makes the first deploy.
   Your site is at `https://<owner>.github.io/<repo>/` (for this repo: https://riannom.github.io/nomura_html/).
4. *(Optional, for the admin page)* **Create a fine-grained personal access token**:
   *GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token*
   - **Repository access:** *Only select repositories* → this repo only.
   - **Repository permissions:** **Contents: Read and write**. GitHub adds **Metadata: Read-only**
     automatically, and you need nothing else. GitHub's "Permissions required for fine-grained personal access
     tokens" page lists `POST /repos/{owner}/{repo}/dispatches` under *Contents* (write).
   - Set a sensible expiration. Store the token in a password manager. **Never commit it.**
   - Note: with Contents write, the token can also push to this repo. That's why it's limited to this one repo.

## Updating the stream

**Option A: admin page (token).** Open `https://<site>/COL/admin.html`, paste the YouTube URL, choose
`live` or `interrupted`, paste the token and press **Update stream**. GitHub answers `204` and the page
shows a success message. You can tick *Remember on this device* to keep the token in that browser's
`localStorage` (only on a device you trust). *Forget saved token* removes it.

**Option B: no token.** While logged into GitHub, use the link at the bottom of the admin page
(*Actions → Update stream → Run workflow*), fill in `url` and `status`, and press **Run workflow**.

**Clearing the URL.** When there is no stream link yet, press **Clear stream URL** on the admin page
(or tick **clear** in *Run workflow*). The workflow writes `"url": ""` and status `interrupted` (from the admin page;
*Run workflow* keeps the status you pick). While the URL is empty, `COL/index.html` shows only the placeholder with
`PLACEHOLDER.noStreamMessage` and the countdown to `PLACEHOLDER.startTime`. It doesn't load any video or the channel fallback.
Semantics: `clear: true` always means "empty the URL". Without it, an empty URL with `interrupted` keeps the previous
URL (which may already be empty), and an empty URL with `live` is rejected.

Either way, viewers switch within **about a minute**: roughly 20–40 s for the workflow and deploy, plus up to 30 s for the next poll.

### How it flows

```
COL/admin.html ──POST /repos/{owner}/{repo}/dispatches (event_type=update-stream, client_payload={url,status[,clear]})──▶ GitHub
   or Actions "Run workflow" (workflow_dispatch inputs url,status,clear)
        │
        ▼
update-stream.yml (concurrency group "stream-update", runs one at a time)
  1. validate: status ∈ {live, interrupted}; url must be https://(www.|m.)youtube.com/… or https://youtu.be/…
     clear=true writes url "" (no validation needed); otherwise empty url + interrupted keeps the previous url,
     empty url + live is rejected
  2. write COL/stream.json with jq --arg (inputs come in via env vars, never ${{ }} in the script)
  3. commit + push only if url/status changed
  4. upload-pages-artifact + deploy-pages (in the same run, because pushes made with GITHUB_TOKEN don't trigger other workflows)
        │
        ▼
COL/index.html polls stream.json?t=<now> with cache:'no-store' every 30 s → full-window player when live, placeholder otherwise
```

A push to `main` (normal edits) also runs the workflow. It skips the update steps and just deploys.

### Configuring owner/repo in COL/admin.html

In this repo they are already set at the top of `COL/admin.html`:

```js
var CONFIG = { owner: 'riannom', repo: 'nomura_html' };
```

If you blank them (`''`), `COL/admin.html` works them out from a `https://<owner>.github.io/<repo>/COL/admin.html` URL on its own,
but **with a custom domain (nomura.network) it can't**, so keep them set.

## Custom domain (CNAME)

1. Rename `CNAME.example` to `CNAME` and put your domain in it (e.g. `live.example.com`). The workflow copies it
   into the published site. Also enter the same domain under *Settings → Pages → Custom domain*. With Actions-based
   deploys, this setting is what counts.
2. DNS at your registrar:
   - **Subdomain** (`live.example.com`): `CNAME` record pointing to `<owner>.github.io`.
   - **Apex** (`example.com`): `A` records to `185.199.108.153`, `185.199.109.153`, `185.199.110.153`,
     `185.199.111.153` (and optionally `AAAA` to `2606:50c0:8000::153`, `…8001::153`, `…8002::153`, `…8003::153`).
3. Once the certificate is issued, tick **Enforce HTTPS**. You can verify the domain under your account's
   *Settings → Pages* to stop anyone else from taking it over.
4. Set `CONFIG.owner` and `CONFIG.repo` in `COL/admin.html` (see above).

## Security notes

- The token is never in the repo. It sits in the streamer's browser (only if *Remember* is ticked) and is
  sent only to `api.github.com`. The pages have a strict Content-Security-Policy and load no third-party scripts.
- URLs are checked in three places: in the admin page (to help the user), in the workflow (this is the real gate)
  and in `COL/index.html` (only `https` on `youtube.com`, `www.`/`m.youtube.com` or `youtu.be`, with no credentials or ports).
- `COL/admin.html` is public but useless without a token. It's marked `noindex`.
- Free-tier limits: Pages is public, and the GitHub Pages CDN caches for about 10 min, so the viewer adds `?t=` to its requests to skip the cache.
