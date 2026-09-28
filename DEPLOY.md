# Hosting: Cloudflare Workers (static assets)

Static single page, served from a Cloudflare Worker's static-assets bundle —
no VPS involved. Deploys automatically via GitHub Actions on push to `main`.

- Site source: `site/public/` (`index.html` + `.well-known/mta-sts.txt`)
- Worker config: `wrangler.jsonc` (`name: tsjeng-nl`)
- Deploy workflow: `.github/workflows/deploy.yml` (`cloudflare/wrangler-action`)

## Domains

| Hostname            | Role                       | How                        |
| -------------------- | --------------------------- | --------------------------- |
| `tsjeng.nl`           | canonical                   | Workers Custom Domain       |
| `www.tsjeng.nl`       | serves site                 | Workers Custom Domain       |
| `mta-sts.tsjeng.nl`   | serves MTA-STS policy file  | Workers Custom Domain       |
| `*.tsjeng.nl`         | CNAME to `tsjeng.nl`        | plain DNS, not Worker-bound |

Workers Custom Domains only route the *exact* hostname they're created for —
the wildcard CNAME does **not** automatically route other subdomains to the
Worker. Any new subdomain that needs to be served must get its own Custom
Domain entry (Cloudflare dashboard → Workers & Pages → tsjeng-nl → Settings →
Domains & Routes, or via the API: `PUT /accounts/{account_id}/workers/domains`
with `{hostname, service: "tsjeng-nl", zone_id, environment: "production"}`).

## Email (Google Workspace, not Workers)

Mail is unrelated to the Worker — MX points straight at Google
(`smtp.google.com`). Keep in the zone:

- `TXT @ v=spf1 include:_spf.google.com -all`
- `TXT _dmarc v=DMARC1;p=reject;...` (Cloudflare + EasyDMARC reporting)
- `TXT google._domainkey ...` (Google Workspace DKIM)
- `TXT _mta-sts v=STSv1; id=...` plus the actual policy file served at
  `https://mta-sts.tsjeng.nl/.well-known/mta-sts.txt` (see the Custom Domain
  note above — this hostname needs its own Custom Domain or the policy file
  becomes unreachable, which happened once during the VPS → Workers move).

## First-time setup (already done, for reference)

1. **Local structure**: `wrangler.jsonc` with `assets.directory: "./site/public"`,
   `account_id` set, matching the pattern used by dethijs.nl / janne-alberts.nl.
2. **GitHub repo**: `kanariezwart/tsjeng-nl` (public), with
   `.github/workflows/deploy.yml` running `wrangler deploy` via
   `cloudflare/wrangler-action@v3` on every push to `main`.
3. **Repo secrets**: `CLOUDFLARE_API_TOKEN` (Workers Scripts edit scope) and
   `CLOUDFLARE_ACCOUNT_ID`. Set once via `gh secret set` or the GitHub UI —
   not reusable across repos, each repo needs its own.
4. **Remove the old VPS DNS records first**, same reason as any Cloudflare
   migration: a conflicting existing A/CNAME record blocks Custom Domain
   creation. Deleted the VPS `A` on `@` and the `www` `CNAME`.
5. **Create the Custom Domains** for `tsjeng.nl`, `www.tsjeng.nl`, and
   `mta-sts.tsjeng.nl`, each bound to the `tsjeng-nl` Worker service. This
   auto-creates proxied `AAAA @ 100::` (and per-hostname) DNS records with
   Cloudflare-managed certs — don't hand-edit those AAAA records.

## No local dev environment

Node/wrangler aren't installed on this machine, so there's no `wrangler dev`
preview — changes go straight out via a push to `main`. Install Node (e.g.
`brew install node`) if a local preview loop becomes worth it.
