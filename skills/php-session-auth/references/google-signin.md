# Sign in with Google (server-side OIDC)

Every application's login page offers "Sign in with Google" alongside email/password. Use the **server-side OAuth 2.0 / OIDC authorization-code redirect flow** — not the Google JS button or One Tap (those inject third-party JS and popup/iframe UI, which fights the no-modal rule and CSP). The login button is a plain link/button to `/auth/google/start`; everything else is PHP.

Library: `composer require league/oauth2-client league/oauth2-google` (light, maintained). Hand-rolling with curl + JWKS verification is acceptable but not the default.

## Flow

1. `/auth/google/start` — generate `state` (session-bound random) and PKCE verifier/challenge, store both in the session, redirect to Google's authorization endpoint with `scope=openid email profile`, the challenge, and a `nonce`.
2. `/auth/google/callback` — verify `state` with `hash_equals` (this is the CSRF guard for the OAuth flow); exchange the code (with the PKCE verifier) for tokens; validate the **ID token**: signature against Google's JWKS, `iss` (`https://accounts.google.com`), `aud` (your client id), `exp`, and the `nonce`. The league provider handles most of this — never trust claims without it.
3. **Require `email_verified: true`.** An unverified Google email is treated as no email: refuse with a generic error.
4. Resolve the account (below), then establish the session exactly like password login: `session_regenerate_id(true)`, set `user_id`, mint a fresh CSRF token, write the activity-log row (`login`, method `google`). **If the account has 2FA enabled, do not establish the session yet** — enter the pending-2FA state per `totp-2fa.md`; Google sign-in does not bypass 2FA.

## Account model

```sql
CREATE TABLE auth_identities (
    id                bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id           bigint NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider          text   NOT NULL,             -- 'google'
    provider_user_id  text   NOT NULL,             -- Google's stable 'sub' claim
    email_at_provider text   NOT NULL,
    created_at        timestamptz NOT NULL DEFAULT now(),
    UNIQUE (provider, provider_user_id)
);
```

`users.password_hash` becomes **nullable** — a Google-only account has none. A null hash means password login is simply impossible for that account (the dummy-hash verify path handles it without enumeration leaks); adding a password later goes through the email-verified reset flow, never a plain form.

## Linking rules (fixed policy)

1. Match `(provider='google', provider_user_id=sub)` → log that user in. The `sub` claim is the identity key; the email is display data (Google emails can change).
2. No identity match, but a user exists with the same normalized, Google-verified email → **auto-link**: insert the identity row, log in. (Safe because Google verified the address; log an `identity_linked` activity row.)
3. No match at all → create the user (null password, email from Google, marked verified) + identity row in one transaction, log in.
4. Never merge two existing users automatically; if the email matches a user who already has a *different* Google identity linked, refuse with a generic error and log it.

## Operational rules

- Client id/secret live in environment config, never in code or the database. Redirect URI is exact-match registered per environment.
- The callback endpoint is idempotent-hostile: consume the stored `state`/verifier on first use (`unset` before the token exchange) so a replayed callback fails.
- Rate-limit `/auth/google/callback` like the login handler; failures return the same generic error as bad passwords.
- Buttons/pages follow the design system: the Google button is part of the minimal auth layout (id `auth-login-google-btn`), and the whole flow is full page navigations.
