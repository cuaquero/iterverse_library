# Iterverse Library

The e-book and reading server for [Iterverse](https://github.com/cuaquero/iterverse_hub),
BTECH IT's umbrella student/staff platform. Production runs at
`library.iterverse.net`.

This repo is a **fork of [Kavita](https://github.com/Kareadita/Kavita)**
(`Kareadita/Kavita`, GPL v3), an open-source reading server. Almost all of
the code here is upstream Kavita. Iterverse added the pieces listed under
"What Iterverse changed" below. Everything else, including the readers,
scanner, metadata handling and user management, is upstream's work and is
documented in upstream's README and wiki.

## Who can sign in

Students and staff sign in through Cloudflare Access (one-time PIN), not
with Kavita passwords. The `kavita-oidc-bridge` Worker (below) is the
OIDC provider Kavita talks to. It lets someone in only if iterverse_hub
reports an active enrollment for their email (`product: "library"`, "any
active enrollment in any course", not a per-course grant) or the email is a
Hub platform admin. See iterverse_hub's `worker/src/entitlement.ts` and its
README for the rule.

Every allowed login gets the `Login` role and access to the library named
`BTECH Catalog`. Emails in the bridge's `ADMIN_EMAILS` secret also get the
`Admin` role.

## What Iterverse changed

Changes live in this repo's git history on top of upstream commit
`1a0ce0051` (2026-09-24). `git log` and `git diff upstream/develop` show the
full set.

| Change | Where |
|---|---|
| **Branding.** Product name "Iterverse Library", icons, favicon, dark accent color, and the two-tone wordmark in the nav header and login screen. The name is one constant (`APP_NAME`) so a future rename or upstream merge touches one file. "Kavita+" naming is left alone because it refers to upstream's separate paid service, not this app. | `UI/Web/src/app/branding.ts`, `UI/Web/src/assets/` |
| **Support links.** Help and update-instructions links point to BTECH IT (`it@btech.edu`) instead of Kavita's Discord and wiki. About-system page and update modal changed accordingly. About 14 "learn more" links to `wiki.kavitareader.com` were removed. | `UI/Web/src/app/`, `UI/Web/src/assets/langs/` |
| **Upstream credit.** A "Powered by Kavita" link in the side nav replaces the Kavita+ donate banner. | `UI/Web/src/app/sidenav/` |
| **OIDC fix.** `GetClaimsFromUserInfoEndpoint` is set to `false`, because ASP.NET's OIDC handler rejects the UserInfo response from Cloudflare Access. The id_token already carries every claim needed. Filed upstream as [Kareadita/Kavita#4949](https://github.com/Kareadita/Kavita/issues/4949). | `Kavita.Server/Extensions/IdentityServiceExtensions.cs` |
| **Admin book upload.** Admins can upload books into a library from Manage Libraries. One file per request, 1 GB per-file limit in Kavita (a proxy such as Cloudflare may enforce a lower cap), saved into a sub-folder of a library root, never overwrites, and the client triggers one scan when the batch finishes. Endpoint is `POST /api/BookUpload`, admin policy only. Kept in new files to keep upstream merges clean. | `Kavita.Server/Controllers/BookUploadController.cs`, `UI/Web/src/app/admin/_modals/upload-books-modal/` |
| **`kavita-oidc-bridge/`.** The OIDC provider Worker, see below. | `kavita-oidc-bridge/` |
| **Build and CI.** `Dockerfile`, `entrypoint.sh` and `.github/workflows/` differ from upstream's current `develop`. | `Dockerfile`, `entrypoint.sh`, `.github/` |

## kavita-oidc-bridge

A small Cloudflare Worker (`kavita-oidc-bridge/`, deployed separately from
Kavita) that acts as Kavita's OIDC authority. Cloudflare Access cannot hand
Kavita a role claim by itself, so the bridge sits between them.

Routes (`src/index.ts`): `/.well-known/openid-configuration`, `/jwks`,
`/authorize`, `/token`. Flow:

1. Kavita sends the browser to `/authorize`, which is gated by a Cloudflare
   Access Application. The Worker also verifies the signed
   `Cf-Access-Jwt-Assertion` itself (`access.ts`).
2. The Worker checks iterverse_hub's `POST /api/entitlement/check` with
   `product: "library"` (`roster.ts`). If the check fails or Hub returns an
   error, the user is sent back to Kavita with `error=not_entitled`.
3. It issues a short-lived signed code (60 seconds), then `/token` returns
   an RS256 id_token (1 hour) with the email and a role claim.

Configuration (`wrangler.jsonc` vars, not secret): `ISSUER`, `CLIENT_ID`,
`ALLOWED_REDIRECT_URI` (`https://library.iterverse.net/signin-oidc`),
`SIGNING_KID`, `SIGNING_KEY_PUBLIC_JWK`, `ACCESS_TEAM_DOMAIN`,
`ACCESS_AUD`, `ROSTER_API_URL`.

Worker secrets: `CLIENT_SECRET`, `SIGNING_KEY_PRIVATE_JWK`, `ADMIN_EMAILS`
(JSON array), `ROSTER_SERVICE_KEY` (must match Hub's `SERVICE_KEY`).

Known limits:

- `ADMIN_EMAILS` is a hand-maintained copy of the "Instructor Dashboard
  Users" Access list. It does not update by itself.
- The library name `BTECH Catalog` is hard-coded in the role claim
  (`library-BTECH Catalog`). Kavita re-syncs library grants on every login
  from claims named `library-<Name>`, so renaming the library in Kavita
  without updating the bridge removes everyone's access at their next login.
- Some comments in `src/index.ts` and `roster.ts` still refer to the retired
  Reader product (`koodo-bridge`).

Run and deploy from `kavita-oidc-bridge/`: `npm install`, `npx wrangler dev`,
`npx wrangler deploy`.

## Running and building

Build and install steps are upstream's. See
[upstream's wiki](https://wiki.kavitareader.com/getting-started),
`INSTALL.txt`, and `build.sh` (builds the .NET solution and the Angular UI
in `UI/Web`), `docker-build.sh`, `monorepo-build.sh`. For local use, Kavita
listens on `localhost:5000` and the first-run setup creates the admin
account in the UI.

## Upstream Kavita

Kavita is a cross-platform reading server for manga, comics and books
(cbr, cbz, zip/rar, 7zip, raw images, epub, pdf), created by
[majora2007](https://github.com/majora2007) and the Kavita contributors.
Upstream's own README, wiki, Discord, issue tracker, Kavita+ subscription
and sponsor information are at
[github.com/Kareadita/Kavita](https://github.com/Kareadita/Kavita). They
describe upstream's project, not this deployment. Do not send Iterverse
support questions to upstream.

To pull upstream changes, fetch the `upstream` remote
(`https://github.com/Kareadita/Kavita.git`) and merge `upstream/develop`.

## Support

BTECH IT: `it@btech.edu`.

## License

GNU GPL v3 (see `LICENSE`), the same license as upstream. Kavita is
copyright its contributors (2020-2026). Iterverse modifications are
distributed under the same license. See `CONTRIBUTING.md` and `SECURITY.md`
(both inherited from upstream and addressed to upstream's maintainers).
