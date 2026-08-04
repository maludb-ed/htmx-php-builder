---
name: malumail-send
description: >-
  Send transactional email through the MaluMail API (api.malumail.com) and
  integrate MaluMail sending into an application. Use when an app needs to send
  email programmatically — welcome/verification/receipt/notification/password-reset
  messages, alerts, or any outbound mail — through MaluMail; when wiring up
  MaluMail sending; when managing a MaluMail suppression list; or when debugging
  MaluMail send failures (401/403/429/502, verified sending domains, API keys,
  Bearer auth). Triggers on "send email via MaluMail", "MaluMail API", "api.malumail.com",
  "transactional email", "suppression list".
---

# MaluMail Send API

Send transactional email over a small REST API. This is the **live** contract at
`api.malumail.com` — every field, status code, and behavior below matches the
running service. Do not invent fields or endpoints beyond what is listed here.

- **Base URL:** `https://api.malumail.com`  (HTTPS only; non-HTTPS → `400`)
- **Auth:** `Authorization: Bearer <api-key>` on every request
- **Bodies/responses:** JSON

## Prerequisites (one-time, in the portal at https://malumail.com/portal/)

1. **Verified sending domain.** You may only send `from` an address whose domain
   is verified on the account (ownership TXT + `SPF include:_spf.malumail.com` +
   DKIM published). Unverified `from` domain → `403`. DKIM/SPF are then applied
   automatically; you never sign anything.
2. **API key.** Portal → **API Keys** → create. Shown **once**, unrecoverable.
   Format `mm_` + 48 lowercase hex chars (regex `^mm_[a-f0-9]{48}$`). Store as a
   secret; never commit it. Revoking in the portal makes it `401` immediately.

## Send an email — `POST /v1/send`

Request JSON:

| Field       | Type               | Required | Notes |
|-------------|--------------------|----------|-------|
| `from`      | string             | yes | Valid email; its domain must be a verified sending domain (else `403`). |
| `from_name` | string             | no  | Display name → `"Name" <from>`. |
| `to`        | string \| string[] | yes | 1–50 addresses. All go in **one shared message** (recipients see each other). |
| `subject`   | string             | yes | Non-empty. |
| `text`      | string             | one of | Plain-text body. |
| `html`      | string             | one of | HTML body. |

At least one of `text`/`html` is required. Both → `multipart/alternative`.
Server auto-generates `Date`, `Message-ID`, `MIME-Version`.

Success `200`:

```json
{"status":"sent","accepted":["alice@example.com"],
 "rejected":[{"email":"bob@example.com","reason":"suppressed:bounce"},
             {"email":"bad","reason":"invalid_address"}]}
```

- **Partial success is still `200`** — always inspect `rejected`.
- `rejected` reasons: `invalid_address`, or `suppressed:<reason>` (never delivered).
- If **all** recipients are rejected → `400 {"error":"No deliverable recipients.","rejected":[...]}`.

Example:

```bash
curl -sS -X POST https://api.malumail.com/v1/send \
  -H "Authorization: Bearer $MALUMAIL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"from":"noreply@yourdomain.com","from_name":"Your App",
       "to":"alice@example.com","subject":"Welcome",
       "text":"Thanks for signing up.","html":"<p>Thanks for signing up.</p>"}'
```

PHP (the plugin-standard helper — `app/mail.php`, loaded by `bootstrap.php`, per php-patterns):

```php
<?php
declare(strict_types=1);

/**
 * Send one email via MaluMail. Returns the decoded API response.
 * Throws RuntimeException on transport errors and non-2xx responses.
 */
function malumail_send(array $mail): array
{
    $ch = curl_init('https://api.malumail.com/v1/send');
    curl_setopt_array($ch, [
        CURLOPT_POST           => true,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_TIMEOUT        => 30,
        CURLOPT_HTTPHEADER     => [
            'Authorization: Bearer ' . getenv('MALUMAIL_API_KEY'),
            'Content-Type: application/json',
        ],
        CURLOPT_POSTFIELDS     => json_encode($mail, JSON_THROW_ON_ERROR),
    ]);
    $body   = curl_exec($ch);
    $status = curl_getinfo($ch, CURLINFO_RESPONSE_CODE);
    $errno  = curl_errno($ch);
    curl_close($ch);

    if ($errno !== 0 || $body === false) {
        throw new RuntimeException('MaluMail transport error.');
    }
    $decoded = json_decode((string) $body, true);
    if ($status !== 200) {
        // 429/502: caller may retry with backoff. 4xx: fix the request. Never log the key.
        throw new RuntimeException("MaluMail send failed ({$status}): " . ($decoded['error'] ?? 'unknown'));
    }
    return $decoded;   // inspect $decoded['rejected']
}
```

```php
$result = malumail_send([
    'from'      => 'noreply@yourdomain.com',
    'from_name' => 'Your App',
    'to'        => $user['email'],
    'subject'   => 'Reset your password',
    'html'      => $html,
    'text'      => $text,
]);
log_activity(db(), 'email', null, 'sent', ['to' => $user['email'], 'rejected' => $result['rejected']]);
```

Python:

```python
import os, requests
r = requests.post("https://api.malumail.com/v1/send",
    headers={"Authorization": f"Bearer {os.environ['MALUMAIL_API_KEY']}"},
    json={"from":"noreply@yourdomain.com","to":"alice@example.com",
          "subject":"Welcome","html":"<p>Thanks!</p>"}, timeout=30)
r.raise_for_status()
for rej in r.json()["rejected"]:
    ...  # suppressed/invalid recipients
```

## Suppression management

`/v1/send` silently refuses suppressed addresses (they appear in `rejected`).
Bounces/complaints are auto-suppressed by the platform. Manage the list:

- `GET /v1/suppressions[?search=<substr>]` → `{"suppressions":[{email,reason,created_at,is_global}]}` (your entries + global, ≤1000).
- `POST /v1/suppressions` `{"email":..,"reason":"manual"|"unsubscribe"}` → `201`; `409` if already present.
- `DELETE /v1/suppressions?email=<addr>` → `200`; `404` if none. Global entries cannot be deleted via API.

## Status codes

| Code | Meaning | Retry? |
|------|---------|--------|
| 200 | Sent (check `rejected`) | — |
| 201 | Suppression created | — |
| 400 | Validation / bad JSON / all recipients undeliverable | no — fix request |
| 401 | Missing/malformed/invalid key or inactive account | no |
| 403 | `from` domain not verified on this account | no — verify domain |
| 404 | Unknown endpoint / no such suppression | no |
| 409 | Already suppressed | no |
| 429 | Rate limit (`N/hour, M/day`) | yes — back off |
| 500 | Key has no relay credential (recreate key) | no |
| 502 | Mail relay refused (SMTP error included) | yes — backoff |

Error bodies are `{"error":"<message>"}` (the all-rejected `400` also has `rejected`).

## Behaviors to design around

- Sender authorization and suppression are **enforced server-side** — cannot be bypassed.
- **Rate limits count each accepted recipient as one unit** (`to` of 10 = 10 units); over limit → `429`, message not sent.
- Duplicate recipients are collapsed. SPF/DKIM/DMARC handled by the platform.

## Not supported (do not offer these)

No attachments/inline images, no `cc`/`bcc`/`Reply-To`/custom headers, no
templates or merge variables, no scheduling, no open/click tracking, no
`List-Unsubscribe` header, no delivery webhooks. Bodies are `text`/`html` only.

**For bulk or individualized mail, send one request per recipient** (avoids
recipients seeing each other and enables per-recipient bodies); honor `429`
backoff. The ≤50 `to` array is only for a genuinely shared message.

## Integration rules

1. Send `from` a verified domain; store the key as a secret.
2. On `200`, inspect `rejected`: treat `suppressed:*` as permanent (don't retry
   those recipients); `invalid_address` is a data-quality issue.
3. **Retry only** `429`/`502`/transient `5xx` with exponential backoff. There is
   **no idempotency key** — never retry a request that already returned `200`
   (it would double-send). Retry only when you did not receive `200`.
4. Feed your own unsubscribe/complaint signals into `POST /v1/suppressions`.
5. Warm up volume gradually on a new domain/account.

## Plugin integration (applications built with htmx-php-builder)

- MaluMail is the **default email channel** for every application: password
  resets and email verification (php-session-auth flows), 2FA recovery notices,
  and application notifications all go through `malumail_send()`.
- `MALUMAIL_API_KEY` lives in environment config alongside `DB_*` — never in
  code, the database, or the browser.
- **Log every send to the activity log** (recipient, subject, rejected list) —
  outbound mail is activity memory. Never log the API key or full message bodies.
- Email bodies are rendered with the same `view()` helper (`app/views/emails/*.php`),
  escaped with `e()` — one template per message type, text and HTML variants.
- An application's unsubscribe screen posts to `POST /v1/suppressions` with
  `reason: "unsubscribe"` (a dedicated full page, per the design system).

*Verified against the live service. Source of truth: the `/v1/send` handler in
`html/mailapi/index.php` on the web app — reconcile this file with it if the API grows.*
