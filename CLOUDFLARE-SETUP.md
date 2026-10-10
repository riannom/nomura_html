# Pointing nomura.network at GitHub Pages (Cloudflare)

Site repo: `riannom/nomura_html`. Home page at `nomura.network/`, stream at `nomura.network/COL/`.

## 1. Verify the domain on GitHub (do this first)
1. GitHub → your **profile** Settings → **Pages** (not the repo's settings) → **Add a domain** → `nomura.network`.
2. GitHub shows a TXT record named `_github-pages-challenge-riannom`. Add it in Cloudflare (DNS → Add record → TXT, name `_github-pages-challenge-riannom`, value as shown).
3. Back on GitHub, click **Verify**. Leave the TXT record in place permanently.

## 2. Set the custom domain on the repo
`riannom/nomura_html` → Settings → **Pages** → Custom domain: `nomura.network` → Save.
(The site deploys via Actions, so no CNAME file is needed.)

## 3. Cloudflare DNS records (all **DNS only / grey cloud**)

| Type | Name | Content |
|------|------|---------|
| A | `@` | `185.199.108.153` |
| A | `@` | `185.199.109.153` |
| A | `@` | `185.199.110.153` |
| A | `@` | `185.199.111.153` |
| AAAA | `@` | `2606:50c0:8000::153` |
| AAAA | `@` | `2606:50c0:8001::153` |
| AAAA | `@` | `2606:50c0:8002::153` |
| AAAA | `@` | `2606:50c0:8003::153` |
| CNAME | `www` | `riannom.github.io` |

- There is already an **orange** `www` record that is broken (error 1016). **Edit/replace it**, don't add a second one.
- Proxy status must be **DNS only (grey)** on all of these, or GitHub can't issue its certificate.

## 4. HTTPS / HTTP settings
- Wait for GitHub to issue the certificate (usually under an hour; Settings → Pages shows it).
- On GitHub: leave **Enforce HTTPS unchecked** so plain `http://` works.
- In Cloudflare → SSL/TLS → Edge Certificates: **Always Use HTTPS = Off**, **HSTS = Off**.
- No Page Rules / Redirect Rules needed.
- If you ever turn the orange proxy on: SSL/TLS mode **Full**, never Flexible (Flexible causes a redirect loop).

## Do NOT touch
- MX records (Google mail for adrian@nomura.network)
- TXT `v=spf1 ...` and `google-site-verification=...`
- Don't create wildcard (`*`) records.

## Notes
- One custom domain covers all paths: `nomura.network/` = home page, `nomura.network/COL/` = stream.
- The old `riannom.github.io/nomura_html/` link will redirect to `nomura.network/`.
- DNS can take a few hours to propagate; set up well before the event.
- Chrome now tries HTTPS first, so let the cert issue even though HTTPS isn't enforced.
- Use the `https://` address for the admin page when entering the token.
