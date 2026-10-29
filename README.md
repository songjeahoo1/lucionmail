# LUCION Mail

Production-oriented, single-mailbox web client for the existing Zoho Mail account `developer@lucion.kro.kr`. The frontend never receives Zoho tokens or secrets. The backend is a Node service that owns the OAuth flow and calls Zoho Mail REST APIs.

> **Connection status:** This project has not been authorized against the account because no Zoho OAuth credentials or account access were supplied. The app does not include sample mail or simulate successful sends. Real account validation remains pending until you complete the account setup below.

## Project layout

```text
src/                 React + TypeScript interface
server/index.ts      OAuth, encrypted token storage and Zoho API proxy
outputs/             User-facing exports (not needed for runtime)
work/                Scratch space
.env.example         Backend configuration names and descriptions
netlify.toml         Static site build settings
```

## Zoho research and supported scope

Zoho's Mail REST API documents OAuth 2.0 server-side authorization, account and folder listing, messages in a folder, search, message content, attachment info and downloads, sending with attachments, drafts, read/unread updates, flags, moving/archiving, spam and delete actions. This implementation uses the REST API rather than building an SMTP/IMAP service. The documented endpoints and scopes are described in [Zoho Mail API index](https://www.zoho.com/mail/help/api/), [OAuth 2.0 guide](https://www.zoho.com/mail/help/api/using-oauth-2.html), [message API](https://www.zoho.com/mail/help/api/email-api.html), [search API](https://www.zoho.com/mail/help/api/get-search-emails.html), and [attachment API](https://www.zoho.com/mail/help/api/post-send-email-attachment.html).

The app requests `ZohoMail.accounts.READ`, `ZohoMail.messages.ALL`, and `ZohoMail.folders.ALL`. The broad `messages.ALL` scope is needed for a combined read/write client; the folder scope enables folder listing. Zoho scopes only grant what the account and organization policy permit. This workspace cannot establish the account's plan, data center, organization policy, app authorization eligibility, sender verification, or API quota. Confirm these in the account/Zoho Admin before treating a feature as available. Zoho documents IMAP/SMTP as a separate option requiring IMAP to be enabled, with paid organization accounts using `imappro.zoho.com:993` and `smtppro.zoho.com:465` or `:587`; account-specific server settings and policy prevail. See [Zoho IMAP access settings](https://www.zoho.com/mail/help/imap-access.html). No SMTP/IMAP credentials are used here.

### Current UI/API behavior

- Folder list is sourced from the connected account, and folder creation calls Zoho. Only real folders returned by Zoho are shown; Starred is a provider search filter using Zoho's documented has:flags search syntax. No fake folders or message counts are generated.
- Inbox listing, provider-side full search (`entire:` syntax), reading details, attachment downloads, send, draft creation, mark read, flag, archive, spam, and delete call Zoho APIs.
- Search defaults to Zoho's `entire:` search. You may use Zoho terms such as `sender:address@example.com`, `subject:invoice`, `content:phrase`, `has:attachment`, and `fromDate:01-Jan-2026` per Zoho's [search syntax](https://www.zoho.com/mail/help/search-syntax.html). Zoho's documented search API is account-wide; the current interface's folder selector does not constrain searches.
- Snooze, folder rename/delete, saved drafts retrieval, Reply All, date and attachment filters are not yet implemented. The unread filter applies only to the currently loaded page. Pagination fetches additional provider pages of 50 messages. Folder APIs and message APIs exist, but these paths need more endpoint-specific handling before being exposed as working actions. Reply sends through Zoho's documented reply endpoint. Forward pre-fills a new message with the sanitized original content; it does not use a specialized forward endpoint. Reply All is not enabled because the documented Mail REST reply action only specifies Reply.
- Composer supports To/Cc/Bcc, subject, rich text entry (bold, italic, underline, links), a server-side Zoho draft request, send, and up to 10 attachments (25 MB each, 35 MB total). Executable attachment extensions are rejected. Zoho may enforce tighter account limits.
- Message HTML is sanitized on the backend; scripts, forms, inline event handlers, and external image sources are removed. Remote images are blocked for privacy. Links open in a new tab with `noopener noreferrer`.

This is a single-user private app. It has no multi-tenant account model or public sign-up. Keep the deployment private to the account owner.

## Local setup

Requirements: Node.js 20 or later and npm.

1. Copy `.env.example` to `.env` and set the values described below. Never commit `.env`.
2. Install packages with `npm install`.
3. Run `npm run dev`. The UI is at `http://localhost:5173`, the backend at `http://localhost:8787`.
4. Select **Connect Zoho Mail**, authorize the listed scopes, and confirm that the real account folders load.

Generate a 32-byte encryption key (for example, `node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"`) and use it for `TOKEN_ENCRYPTION_KEY`. Set a separate long random `SESSION_SECRET`. Token storage is AES-256-GCM encrypted in `.data/tokens.enc`, which is ignored by Git. Protect and back up this file only if you need to preserve the authorization across a local restart; disconnect deletes it.

## Zoho OAuth configuration

1. In the Zoho API Console, create a **Server-based Application**.
2. Register the exact redirect URI `APP_BASE_URL/api/auth/callback` (local example: `http://localhost:8787/api/auth/callback`; production example: `https://api.lucion.kro.kr/api/auth/callback`). Zoho requires an exact match.
3. Configure the OAuth client ID and secret only in the backend environment.
4. Set `ZOHO_ACCOUNTS_BASE` and `ZOHO_API_BASE` to the account's actual Zoho data-center hosts. Examples are `.com` and `.eu`; do not guess the region. Mail API examples use `https://mail.zoho.com`.
5. Set `APP_BASE_URL` to the public backend URL and `FRONTEND_ORIGIN` to the exact frontend origin, with no trailing slash. In production use HTTPS and same-site subdomains, e.g. `https://mail.lucion.kro.kr` and `https://api.lucion.kro.kr`.
6. Set `TOKEN_ENCRYPTION_KEY`, `SESSION_SECRET`, `ALLOWED_FROM=developer@lucion.kro.kr`, and `NODE_ENV=production`.
7. In Zoho Mail, verify that `developer@lucion.kro.kr` is an authorized sender for the connected user. The backend pins the From address to `ALLOWED_FROM`; the frontend cannot choose a different sender.
8. Start the connection flow. OAuth `state` is random and HMAC-checked; the authorization code is exchanged server-side. The refresh token is encrypted at rest. Access tokens refresh automatically.

If Zoho says scope or API access is unavailable, review the organization admin policy and account subscription with the Zoho administrator. Do not replace the error with local mock data.

## Deployment

### Netlify frontend

`netlify.toml` configures the Vite build and publishes `dist/`. Set the Netlify build environment variable `VITE_API_BASE=https://api.lucion.kro.kr` for a separate API host; the default `/api` is intended for a same-origin reverse proxy. If you use another API hostname, update the `connect-src` value in `netlify.toml` to match. For a separate `api.lucion.kro.kr` host, set the frontend API base in `src` deployment configuration to the API origin and keep both hosts under `lucion.kro.kr` so the secure session cookie remains same-site. Set the backend's `FRONTEND_ORIGIN` to the exact Netlify/custom domain. Use a custom domain before production OAuth; allowlist the callback URL in Zoho.

### Backend

Run this Node service on a persistent HTTPS-capable host (for example, a small private container/VM service). Set all `.env.example` values in the host's secret manager. Persist `.data/tokens.enc` on an encrypted durable volume, restrict filesystem access to the service account, and back it up securely. The included token store uses a local file; a stateless serverless runtime or ephemeral filesystem is **not supported as-is**. For a serverless deployment, replace this store with a managed secret/KMS-backed encrypted store before deployment. Expose only the API host, not filesystem paths or debug endpoints. Keep the frontend origin allowlist exact.

## Checks and account validation

Build check: `npm run build`.

After OAuth setup, use this manual acceptance procedure with the real account:

1. Connect, reload, and confirm the session stays connected. Disconnect and confirm the app reports remote revocation; if it cannot, revoke LUCION Mail under Zoho Connected Apps.
2. Compare Inbox folders and message subjects against Zoho Mail; open a known message and confirm sender, recipient, date, body, and attachments.
3. Search a known sender and subject and compare results in Zoho. The search uses Zoho's account-wide search API, not only currently loaded messages.
4. Mark a message read and starred, refresh, and verify the state in Zoho Mail.
5. Archive, report a disposable test message as spam, and delete a disposable test message. Verify each action in Zoho.
6. Send a test message to an account you control, then reply from that external account and verify both directions. Do not send a test to an unintended recipient.
7. Send a small safe attachment and download an existing attachment. Confirm executable file extensions are blocked and attachment names are treated as downloads.
8. Inspect browser network requests and built frontend assets: they must not contain client secrets, OAuth tokens, or refresh tokens.
9. Verify production OAuth redirect, TLS, allowed frontend origin, durable encrypted token storage, restart behavior, logs, and rate limiting.

No live account or production host was available in this implementation session. The checks above are procedures, not claims of completed live-account verification.

## Security checklist

- [x] No Zoho secrets are bundled into the browser; OAuth exchange and API calls are server-side.
- [x] `.env`, token data, build outputs and dependencies are ignored by Git.
- [x] OAuth state is validated; redirect URI is constructed from one configured backend URL.
- [x] OAuth token storage is AES-256-GCM encrypted; session cookie is HttpOnly.
- [x] Sender address is pinned server-side; Zod validates recipients and message input.
- [x] Exact origin allowlist, secure production cookies, request size limit, rate limit, and security headers.
- [x] HTML sanitization, external image blocking, safe link attributes, attachment path ID validation and download size limit.
- [ ] Production secret manager/KMS and persistent encrypted storage must be provisioned by the deployment owner.
- [ ] Confirm current Zoho account API plan/policy, sender authorization, regional API hosts, and any service-specific rate or message-size limits.
- [ ] Add automated integration/security tests and review server/API error sanitization before opening access beyond the account owner.
