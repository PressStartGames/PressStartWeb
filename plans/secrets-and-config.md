# Plan: secrets in Infisical, public config in git

Written 2026-11-07 with the founder. Goal: **one place for every secret** (Infisical Cloud, free plan, hosted by
Infisical) and **one place for every public setting** (a config map in this repository). Cloudflare and GitHub hold
only copies that Infisical keeps in sync. Nothing here contains a value.

## Decisions (founder, 2026-11-07)

| Topic | Decision |
|---|---|
| Secrets | All of them in Infisical Cloud (free): viewable after saving, synced to Cloudflare Workers and GitHub |
| Public settings | A config map in git, one file per environment; never in Infisical, never secret |
| Existing values | Moved as they are from `.env.secrets.local`; the founder re-rolls them later, in Infisical |
| `CONFIRM_LINK_SECRET` (production) | Replaced with a new value during the move: there are no subscribers (checked by the founder), so no sent link breaks |
| `RESEND_API_KEY` (production) | Its value was never recorded: the founder creates a new Resend key during the move and deletes the old one |
| After the move | `.env.secrets.local` and `scripts/sync-secrets.mjs` are deleted; `INFRASTRUCTURE.md` keeps names and where to re-roll |

## Infisical layout

Project **`press-start-web`**, environments **`staging`** and **`production`** (the free plan allows 3; the default
`development` is kept for local `wrangler dev` values or deleted). Two folders per environment, so each destination
receives only what it needs:

| Folder | Secrets | Synced to |
|---|---|---|
| `/worker` | `WORKOS_API_KEY`, `SESSION_SECRET`, `CONFIRM_LINK_SECRET`, `RESEND_API_KEY` | the API Worker of that environment (`press-start-signups-staging`, `press-start-signups`) as Worker **secrets** |
| `/github` | `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID` | the GitHub Environment of the same name (`staging`, `production`) in `PressStartGames/PressStartWeb` |
| `/tools` (production only) | `GROWTHBOOK_SECRET_API_KEY` | not synced (used only to manage GrowthBook through its API) |

**Syncs: 4** (free plan: 10). A sync carries a whole folder, so re-rolling or adding a key needs no new sync; only a
new destination does. Each sync overwrites its destination and removes what Infisical does not hold (everything is
managed there). The other Workers (site, admin, beta redirect) use no secrets.

Dropped: `CLOUDFLARE_DNS_TOKEN_OLD` (revoked), the repository variable `VITE_SIGNUP_ENDPOINT` (unused since GitHub
Pages was retired).

## The public config map

`config/staging.json` and `config/production.json` (plus `config/development.json` and `config/e2e.json` for local
and test builds): every public value once, grouped by app.

- **client:** base, site URL, sign-up endpoint, fallback URL, Turnstile site key, GA4 id, Web Analytics token,
  GrowthBook client key and host
- **admin:** API base
- **api** (`server/wrangler.toml` vars): site URL, API URL, allowed origins, admin origins, cookie domain and name,
  WorkOS client id, mail from / mode / allowlist, daily send cap, D1 id
- **site-worker** vars: GrowthBook client key and host, robots noindex

`scripts/config.mjs` writes the files the tools read (`client/.env.<mode>`, `admin/.env.<mode>`, the `vars` blocks
of `server/wrangler.toml` and `site-worker/wrangler.jsonc`) from the map. `pnpm lint` fails if a written file
differs from the map, so a value is changed in exactly one place. The existing typed env checks stay.

## Steps

| # | Step | Who |
|---|---|---|
| 1 | Infisical account (Cloud), 2FA, organization, project `press-start-web` with `staging` and `production` | Founder |
| 2 | A machine identity for Claude (setup only, removed in step 9), its credentials into `.env.secrets.local` | Founder |
| 3 | App connections in Infisical: Cloudflare (a new API token) and GitHub (the Infisical GitHub app on the repository) | Founder |
| 4 | A new production Resend key into `.env.secrets.local` | Founder |
| 5 | Folders and secrets loaded from `.env.secrets.local` through the Infisical API (no value printed); new production `CONFIRM_LINK_SECRET` generated in Infisical | Claude |
| 6 | The 4 syncs; check each destination lists exactly the expected names | Claude |
| 7 | Verify: staging deploy (push to `staging`), a sign-up and its confirm link; production deploy (founder approves), a sign-up and confirm, the founder signs in to admin | Claude + founder |
| 8 | The config map and `scripts/config.mjs`; generated files unchanged in content; CI green | Claude |
| 9 | Retire: delete `.env.secrets.local`, `scripts/sync-secrets.mjs`, the setup identity, the repository variable; old Resend key deleted; `INFRASTRUCTURE.md`, `CLAUDE.md` and the skills point to Infisical and the config map | Claude + founder |

Step 8 is independent of steps 1–7 and can run first.

## Never break

- A sign-up on the live beta and its confirm link; admin sign-in.
- No secret value printed, committed, or written outside Infisical (and, until step 9, the git-ignored key file).
- Production changes deploy only through the approval-gated workflow.
