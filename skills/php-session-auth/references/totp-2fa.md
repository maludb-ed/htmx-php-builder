# Authenticator-app 2FA (TOTP)

Every application offers optional per-user two-factor authentication with a standard authenticator app (RFC 6238 TOTP: Google Authenticator, 1Password, Authy, …). Apps may additionally enforce it per role, but the mechanism is the same.

Library: `composer require spomky-labs/otphp endroid/qr-code` (TOTP + QR rendering; the QR is generated server-side as a data URI — no external chart/QR service, ever).

## Schema

```sql
ALTER TABLE users ADD COLUMN totp_secret text,            -- encrypted at rest (libsodium secretbox, key in env)
                  ADD COLUMN totp_enabled_at timestamptz, -- null = 2FA off
                  ADD COLUMN totp_last_timestep bigint;   -- replay guard

CREATE TABLE totp_recovery_codes (
    id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id    bigint NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    code_hash  text   NOT NULL,                            -- password_hash() of the code
    used_at    timestamptz,
    created_at timestamptz NOT NULL DEFAULT now()
);
```

## Enrollment (settings screen — dedicated full page, per the design system)

1. Generate a secret; render the QR (otpauth URI with `issuer` = app name, label = user email) plus the manual-entry key. The secret stays in the session, **not** in the database, until confirmed.
2. User submits one valid code → verify (window ±1 timestep) → only then persist the encrypted secret, set `totp_enabled_at`, and generate **10 single-use recovery codes**, shown exactly once (stored hashed). Log `totp_enabled`.
3. Disabling 2FA requires a current TOTP or recovery code (and re-authentication), never just a click. Log `totp_disabled`.

## Login flow (applies to password AND Google sign-in)

1. Primary auth succeeds → if `totp_enabled_at` is null, establish the session normally. Otherwise **do not log the user in**: store `$_SESSION['pending_2fa_user_id']` (a distinct key — `user_id` stays unset), `session_regenerate_id(true)`, and redirect to the dedicated `/login/2fa` page (full navigation, minimal auth layout, ids `auth-2fa-form` / `auth-2fa-field-code`).
2. Verify the submitted code with a ±1 timestep window. **Replay guard:** reject any timestep ≤ `totp_last_timestep`; store the accepted one.
3. Recovery path: the same form accepts a recovery code; verify against unused hashes, mark `used_at`, and warn when few remain.
4. On success: promote the session — `session_regenerate_id(true)`, move pending id to `user_id`, mint a fresh CSRF token, clear the pending key, log `login_2fa`. On failure: generic error; rate-limit exactly like password attempts (per-account + per-IP counters, 429 lockout) — six digits brute-force in hours without it.
5. The pending state expires (10 minutes) and grants access to nothing: every authed page checks `user_id`, not the pending key.

## Rules

- 2FA is a property of the **account**, not the login method — Google sign-ins hit the same challenge.
- Server clock discipline matters: NTP on the Ubuntu host; never widen the window beyond ±1 to "fix" drift.
- Secrets are encrypted at rest with a key from environment config; a database dump alone must not yield working seeds.
- Trusted-device ("remember this browser for 30 days") is **off by default**; if a project enables it, the cookie holds a random token hashed in a `trusted_devices` table and is cleared by password change or 2FA re-enrollment.
- All 2FA screens are dedicated full pages in the minimal auth style — never a modal or an HTMX swap (session promotion is a navigation boundary, same as login).
