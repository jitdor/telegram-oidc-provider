# Telegram OIDC Provider

A self-hosted **OAuth 2.0 / OpenID Connect** identity provider that uses **Telegram** for authentication. Users scan a QR code with their Telegram app and approve the login inside Telegram – no passwords required.

It runs as a standalone server, or can be embedded as a library: every instance is built from an injected context (`config`, database/store, key store, Telegram API), with no module-level state.

## Features

- QR code login via Telegram bot deep link
- OAuth 2.0 Authorization Code flow with PKCE (S256, required)
- OpenID Connect discovery, JWKS (with `kid`/`use`/`alg` and multi-key rollover), `/userinfo`
- Rotating refresh tokens with **reuse detection** (a replayed token revokes the user's whole grant for that client)
- Token revocation (RFC 7009) and introspection (RFC 7662) with a `jti` denylist
- Conditional authentication policies (group membership, user ID lists, username regex, Premium status, language, logical operators)
- Per-client scope restrictions (`allowed_scopes`) enforced at `/authorize`
- Consent records with user-facing withdrawal (`/apps`, `/revoke` in the bot)
- Confidential clients: salted scrypt secret hashes, `client_secret_basic` and `client_secret_post`
- Per-IP rate limiting on `/token`, `/authorize`, the status poll, `/userinfo`, `/introspect`, `/revoke`
- Login page served under a strict Content-Security-Policy, `no-store`
- SQLite storage with versioned schema migrations; all timestamps are epoch seconds

## Requirements

- **Node.js 22.13+** (uses the built-in `node:sqlite`, which is unflagged from 22.13; see `.nvmrc`). Node 22 prints an
  `ExperimentalWarning` for `node:sqlite`; the npm scripts suppress it.
- A Telegram bot token from [@BotFather](https://t.me/BotFather)
- A public HTTPS URL (Telegram delivers updates by webhook)

## Installation

```bash
git clone https://github.com/jitdor/telegram-oidc-provider.git
cd telegram-oidc-provider
npm ci
cp .env.example .env              # then edit it
npm run register-client -- --client-id my-app --name "My App" \
  --redirect-uri https://my-app.example.com/callback
npm start
```

On start the server generates a signing key (in `KEYS_DIR`) if there is none, sets the Telegram webhook, and listens.

## Configuration

| Variable                     | Description                                                           | Default                 |
|------------------------------|-----------------------------------------------------------------------|-------------------------|
| `PORT` / `HOST`              | Listen address                                                        | `3000` / `0.0.0.0`      |
| `BASE_URL`                   | Public HTTPS base URL: the issuer and webhook host                    | `http://localhost:3000` |
| `TELEGRAM_BOT_TOKEN`         | Bot token                                                             | *(required)*            |
| `TELEGRAM_WEBHOOK_SECRET`    | 16–256 chars of `A-Za-z0-9_-`; authenticates webhook calls            | *(required)*            |
| `TELEGRAM_BOT_USERNAME`      | Overrides the username learned from `getMe`                           | –                       |
| `DB_PATH`                    | SQLite database file                                                  | `./data.db`             |
| `KEYS_DIR`                   | Signing key directory                                                 | `./keys`                |
| `ACCESS_TOKEN_TTL`           | Access token lifetime (`90s`, `15m`, `12h`, `30d`, or seconds)       | `15m`                   |
| `ID_TOKEN_TTL`               | ID token lifetime                                                     | `15m`                   |
| `REFRESH_TOKEN_TTL`          | Refresh token lifetime (sliding, per rotation)                        | `30d`                   |
| `AUTH_REQUEST_TTL`           | How long a QR code stays valid                                        | `2m`                    |
| `AUTH_CODE_TTL`              | Authorization code lifetime                                           | `60s`                   |
| `REEVALUATE_POLICY_ON_REFRESH` | Re-run the client policy on every refresh                           | `true`                  |
| `ISSUE_REFRESH_TOKENS`       | `offline_access`: only when that scope is granted; `always`: on every code exchange | `offline_access` |
| `REFRESH_REUSE_GRACE`        | Seconds during which replaying a just-rotated refresh token only fails instead of revoking the grant (`0` = strict) | `0` |
| `TRUST_PROXY`                | Trust `X-Forwarded-*` (set behind Caddy/Nginx so rate limits see real IPs) | `false`            |
| `RATE_LIMIT_TOKEN`, `RATE_LIMIT_AUTHORIZE`, `RATE_LIMIT_STATUS`, `RATE_LIMIT_USERINFO`, `RATE_LIMIT_INTROSPECT`, `RATE_LIMIT_REVOKE` | Requests per minute per IP (`0` disables) | `30`, `30`, `120`, `120`, `120`, `60` |

The server refuses to start without a strong `TELEGRAM_WEBHOOK_SECRET`: anyone who knows it can forge Telegram updates, i.e. approve logins as any user.

## Registering OAuth clients

`scripts/register-client.js` is scriptable (flags or environment) and falls back to prompts only on a TTY:

```bash
npm run register-client -- \
  --client-id my-app --name "My App" \
  --redirect-uri https://my-app.example.com/callback \
  --scopes "openid profile telegram offline_access" \
  --generate-secret \            # or --secret …, or neither for a public client
  --first-party \                # skip the consent prompt after the first approval
  --policy-file policy.json \    # or --policy '{"type":"is_premium","value":true}'
  --json
```

Environment equivalents: `CLIENT_ID`, `CLIENT_NAME`, `CLIENT_REDIRECT_URIS` (comma separated), `CLIENT_SCOPES`, `CLIENT_SECRET`, `CLIENT_GENERATE_SECRET=1`, `CLIENT_FIRST_PARTY=1`, `CLIENT_POLICY_FILE`, `CLIENT_POLICY`, `DB_PATH`. Running it again with the same client id updates the client.

Validation: redirect URIs must be absolute, fragment-free and HTTPS (plain HTTP only for `localhost`/loopback); scopes must be a subset of `openid profile telegram offline_access`; policies are schema-checked; secrets must be ≥ 16 characters. Secrets are stored as salted scrypt hashes. Clients registered by older versions (unsalted SHA-256) keep working and are upgraded to scrypt on their next successful authentication.

## Signing keys and rotation

Keys live in `KEYS_DIR` as `<kid>.pem` (PKCS#8, mode 0600) plus an `active` file naming the signing key; the `kid` is the RFC 7638 thumbprint. Every key in the directory is published in `/jwks` and accepted for verification, so rollover is:

```bash
npm run keys -- generate          # new key, published but not yet signing
# restart; wait > 5 min (JWKS is served with max-age=300)
npm run keys -- activate <kid>    # restart: new tokens are signed with it
# wait > ACCESS_TOKEN_TTL / ID_TOKEN_TTL
npm run keys -- retire <old-kid>  # restart: stop publishing it
```

A `private.pem` from earlier versions is still loaded. **Back the directory up**: losing it invalidates all issued JWTs. The key store is an interface (`getSigningKey`, `getVerificationKey`, `getJwks`), so a KMS/HSM-backed implementation can replace the file store.

## OAuth 2.0 / OIDC usage

### Authorization request

```
GET /authorize
    ?client_id=YOUR_CLIENT_ID
    &redirect_uri=https://yourapp.example.com/callback
    &response_type=code
    &scope=openid%20profile%20telegram
    &state=xyz123
    &nonce=abc
    &code_challenge=BASE64URL(SHA256(code_verifier))
    &code_challenge_method=S256
```

- `redirect_uri` must match a registered URI **exactly** (string comparison). An unknown `client_id` or unregistered redirect URI renders an `invalid_request` error page; nothing is redirected to an unverified URI.
- All other errors (`unsupported_response_type`, `invalid_request` for missing/invalid PKCE, `invalid_scope`, `login_required` for `prompt=none`) are redirected to the client as `?error=…&error_description=…&state=…&iss=…`.
- Requested scopes must be within the client's `allowed_scopes`. Without `scope`, `openid profile telegram` (intersected with the allowed scopes) is used.
- On success the browser is redirected to `redirect_uri?code=…&state=…&iss=…` (`iss` per RFC 9207). If the user denies, or fails the client's policy, it is redirected with `error=access_denied`.

### Token endpoint

Accepts `application/x-www-form-urlencoded` (standard) or JSON. Confidential clients authenticate with HTTP Basic (`client_secret_basic`) or `client_secret` in the body (`client_secret_post`); public clients send only `client_id`.

```bash
curl -X POST https://telegram-oidc-provider.example.com/token \
  -u "YOUR_CLIENT_ID:YOUR_SECRET" \
  -d grant_type=authorization_code -d code=AUTH_CODE \
  -d redirect_uri=https://yourapp.example.com/callback -d code_verifier=YOUR_PKCE_VERIFIER
```

```json
{
  "access_token": "eyJ...",
  "token_type": "Bearer",
  "expires_in": 900,
  "scope": "openid profile telegram",
  "refresh_token": "…",
  "id_token": "eyJ..."
}
```

A refresh token is issued only when the `offline_access` scope was requested and granted (set `ISSUE_REFRESH_TOKENS=always` for the previous behaviour). Refresh with `grant_type=refresh_token&refresh_token=…` (optionally `scope=` to narrow; asking for a wider scope fails with `invalid_scope` without consuming the token).

Each use **rotates** the token. Presenting an already-rotated token is treated as theft and revokes every refresh token of that user/client pair. By default this is strict: two concurrent refreshes of the same token also count, so clients must serialize refreshes or the user is logged out. `REFRESH_REUSE_GRACE=<seconds>` relaxes this for buggy-but-honest clients: a replay within that window just fails with `invalid_grant` (it never yields tokens) and the successor survives; later replays still revoke the grant.

An authorization code can be exchanged once; replaying it revokes the tokens it produced. A code is burnt on *any* presentation after lookup — including by the wrong client or with a wrong PKCE verifier — so a leaked code is never redeemable afterwards (at the cost that its holder can prevent the legitimate redemption).

Errors are JSON `{ "error": "…", "error_description": "…" }` with RFC 6749 codes, `Cache-Control: no-store`.

### Tokens

- Access tokens are RFC 9068 JWTs (`typ: at+jwt`, RS256, `kid`), `aud: ["<issuer>/userinfo", "<client_id>"]`, with `client_id`, `scope`, `jti`, `auth_time`.
- ID tokens carry `aud`/`azp` = client id, `nonce`, `auth_time`, and profile claims for the granted scopes.
- `profile` → `name`, `given_name`, `family_name`, `preferred_username`, `picture`, `locale`; `telegram` → `telegram_id`, `telegram_is_premium`.

### Userinfo

```bash
curl https://telegram-oidc-provider.example.com/userinfo -H "Authorization: Bearer ACCESS_TOKEN"
```

Requires an access token (ID tokens are rejected) whose audience includes the IdP's userinfo resource, that is not revoked, and whose grant is still consented. Requires the `openid` scope.

### Introspection and revocation

- `POST /introspect` (RFC 7662, client-authenticated): `token=…`. A client may only introspect its own tokens. For access tokens this also re-evaluates the client's policy live, so resource servers that introspect get per-request enforcement.
- `POST /revoke` (RFC 7009, client-authenticated): `token=…`. A refresh token revokes its whole rotation family; an access token is added to the `jti` denylist (honoured by `/userinfo` and `/introspect`; self-validating resource servers will accept it until it expires).

### Discovery

- `/.well-known/openid-configuration`
- `/jwks`

## Conditional authentication policies

Policies are stored per client as JSON and validated at registration.

| Type                   | Description                                    | Parameters                           |
|------------------------|------------------------------------------------|--------------------------------------|
| `user_id_in_list`      | Telegram ID in list                            | `user_ids`: array                    |
| `user_id_not_in_list`  | Telegram ID not in list                        | `user_ids`: array                    |
| `username_matches`     | Username matches regex                         | `pattern`: string                    |
| `group_membership`     | Member of a group                              | `chat_id`, `role` (optional)         |
| `any_group_membership` | Member of at least one group                   | `chat_ids`: array, `role` (optional) |
| `all_group_membership` | Member of all groups                           | `chat_ids`: array, `role` (optional) |
| `is_premium`           | Has (or lacks) Telegram Premium                | `value`: boolean                     |
| `language_code_in`     | Telegram client language; matches the full tag (`pt-br`) or its primary subtag (`pt`), case-insensitive | `codes`: array |

Operators: `and`, `or` (`conditions`: array), `not` (`condition`). Unknown types/operators and API errors fail closed. Group checks need the bot to be a member of the group.

```json
{
  "operator": "or",
  "conditions": [
    { "type": "is_premium", "value": true },
    { "type": "group_membership", "chat_id": -1003333333333, "role": "administrator" }
  ]
}
```

### When policies are evaluated

- **At login**, against the live Telegram user object from the `/start` update (so `is_premium` and `language_code` are current). The profile — including Premium status and language — is stored on approval.
- **On every refresh** (unless `REEVALUATE_POLICY_ON_REFRESH=false`), against the stored profile and live group membership. A failure revokes the grant's refresh tokens.
- **On introspection** of access tokens.
- **Not** for self-validated JWTs: a resource server that only verifies access tokens against the JWKS keeps accepting a token for up to `ACCESS_TOKEN_TTL` (15 min) after group removal, a policy change or a revocation. High-security resource servers should call `/introspect`.

## Consent

Approving a login records consent for exactly the requested scopes. An access token stays valid for `/userinfo`/`/introspect` only while every scope it carries is still consented, and was consented no later than the token was issued; consenting to additional scopes later does not affect existing tokens. For **first-party** clients (`--first-party`) a later login that asks for no more than was consented to is approved without a prompt; third-party clients always prompt. In the bot, `/apps` lists the apps a user has granted access to, with revoke buttons, and `/revoke <client_id>` does the same. Revoking deletes the consent and all refresh tokens; outstanding access tokens stop passing `/userinfo` and `/introspect` immediately.

## Embedding as a library

```js
import { Bot } from 'grammy';
import { createIdp, defineConfig, createDatabase, createFileKeyStore } from 'telegram-oidc-provider';

const config = defineConfig({ issuer: 'https://id.example.com', telegramWebhookSecret: '…' });
const bot = new Bot(process.env.TELEGRAM_BOT_TOKEN);
await bot.init();
const { app, ctx } = await createIdp({
  config,
  bot,                                   // handlers are registered on it; its API is used for group checks
  db: createDatabase('/var/lib/idp.db'),
  keys: await createFileKeyStore('/var/lib/idp-keys'),
});
await app.listen({ port: 3000 });
```

For finer control use the pieces directly: `createIdpContext({ config, db | store, keys, telegram, clock, logger })` returns the context (`ctx.oauth` is the transport-agnostic OAuth service), `buildApp(ctx)` the Fastify adapter, and `createBotHandlers(ctx)` / `registerBotHandlers(bot, ctx)` the Telegram adapter. Several instances with different configs can run in one process.

## Deployment

### Caddy

```
telegram-oidc-provider.example.com {
    reverse_proxy localhost:3000
}
```

Set `TRUST_PROXY=true` behind a reverse proxy, and make sure `BASE_URL` matches the public URL exactly.

### Systemd

```ini
[Unit]
Description=Telegram OIDC Provider
After=network.target

[Service]
Type=simple
User=telegram-oidc-provider
WorkingDirectory=/opt/telegram-oidc-provider
EnvironmentFile=/opt/telegram-oidc-provider/.env
ExecStart=/usr/bin/npm start
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

## Security notes

- PKCE (S256) mandatory; verifier format checked; code comparison constant-time.
- Redirect URIs validated at registration and matched exactly; errors are only redirected to verified URIs.
- Deep-link tokens are stored hashed, short-lived, bound to the browser session (status polling) and to the first Telegram account that opens them — only that account can approve or deny.
- Authorization codes and refresh tokens are consumed/rotated with single conditional `UPDATE`s (no check-then-act races); replays revoke what was issued.
- Client secrets: scrypt (N=2^15, r=8, p=1) with per-secret salt, constant-time comparison, single `invalid_client` error.
- Access tokens verified for signature (by `kid`), `RS256`, `typ: at+jwt`, issuer, audience, expiry, `jti` denylist and consent.
- Login page: all values HTML-escaped, no inline script, CSP `default-src 'none'; script-src 'self' …; frame-ancestors 'none'`, `Cache-Control: no-store`, `__Host-` session cookie over HTTPS.
- Telegram webhook secret compared in constant time; required at startup.

## Development

```bash
npm run dev       # start the server, restarting on file changes
npm test          # node:test suite (each test builds its own in-memory IdP)
npm run lint      # ESLint
npm run typecheck # tsc --checkJs over JSDoc types (src/types.js)
npm run check     # lint, typecheck and test in one go
```

CI runs all three on Node 22.13, 22 and 24.

## Known limitations

- Rate limits are in-memory, per process.
- Multi-instance deployments need a shared database and the same `KEYS_DIR` contents on every instance (or a custom key store).
- Revocation of access tokens is only visible to `/userinfo`, `/introspect`, and resource servers that introspect.
- Only the authorization code grant; no dynamic client registration, no `prompt=none`/silent login.

## License

MIT
