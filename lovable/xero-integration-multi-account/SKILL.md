---
name: xero-integration-multi-account
description: Use when building a MULTI-TENANT Xero integration in Lovable where each end user connects THEIR OWN Xero organisation, and the app reads/writes Xero as that user with their own access + refresh tokens. Covers the full OAuth2 lifecycle in Supabase Edge Functions, per-user encrypted token storage with RLS, refresh/rotation, tenant selection, scopes, and Xero API gotchas (granular scope migration, date/money/tax normalisation, idempotent writes, rate limits). Not for single-org apps where you (the builder) connect one Xero you own — that needs only one stored token set, not per-user keying.
---

# Multi-account / multi-tenant Xero (each user connects their own Xero)

This skill is for a **multi-tenant SaaS built in Lovable**: every customer links
**their own** Xero organisation, and your app reads/writes Xero **as that
customer**, with **their** tokens.

> **First, make sure you actually need this.** Lovable has no managed
> single-principal Xero connector (unlike some platforms). So you own the OAuth
> lifecycle regardless. The real question is *how many Xero logins exist*:
>
> - **Single-org app** — you (the builder) connect **one** Xero you own
>   (internal dashboard, single-merchant store, prototype). You still run OAuth
>   once, but you store **one** token set and skip all the per-user keying below.
>   That is a trimmed-down version of this skill, not a different mechanism.
> - **Multi-tenant** — many *separate* Xero logins, one per customer. That is
>   this skill: per-user token rows, tenant selection, the works.
>
> Don't confuse Xero **multi-org** with **multi-tenant**. `GET /connections`
> returns several *organisations* under ONE login — still one principal.
> Multi-tenant = many *separate* Xero logins, one per customer.

## Where this code runs in Lovable

Lovable's backend is **Supabase**: Postgres for storage, **Edge Functions (Deno)**
for server code, Supabase secrets for config. All OAuth and token handling in this
skill lives in **Edge Functions** — never in the React/Vite frontend, and never in
the browser. Key consequences that shape everything below:

- **Edge Functions are stateless.** The function that starts consent and the
  function that handles the callback are **separate cold-start invocations** with
  no shared memory. Anything you need across the two hops (the `state`, and the
  PKCE `code_verifier` if you use PKCE) **must be persisted** to a table. This is
  the number-one porting trap from a long-running Node server.
- **Use `fetch`, not the `xero-node` SDK, for the OAuth lifecycle.** `xero-node`
  keeps OAuth/PKCE state in process memory, which does not survive across two
  stateless invocations, and its `openid-client` dependency is finicky under Deno.
  Hand-rolling the three OAuth calls (authorize redirect, token exchange, refresh)
  with `fetch` is short and reliable. You *may* `import { XeroClient } from
  "npm:xero-node"` for read/write API calls if it loads in your edge runtime, but
  the flow itself should be `fetch`-based.
- **Crypto is Web Crypto (`crypto.subtle`), not Node `crypto`.** The blob format
  differs — see the encryption helpers.

## Decision check before building

Build the multi-tenant version only if ALL are true:
- Each customer connects their **own** Xero org (not yours).
- The app acts under **each customer's** Xero identity, not a shared one.
- You can securely store and rotate per-user refresh tokens (Supabase + RLS, below).

If any is false, use the single-org simplification (one stored token set).

## 1. Register your own Xero app

In the Xero developer portal (developer.xero.com) create a **Web app** OAuth2 app
(confidential — it holds a secret, which your Edge Functions can):
- Grab **Client ID** and **Client Secret**.
- Register your redirect URI as your **callback Edge Function URL**, exactly:
  `https://<project-ref>.supabase.co/functions/v1/xero-callback`
  (your `<project-ref>` is in Lovable's Supabase settings). Add your local dev
  Supabase URL too if you test locally — Xero allows multiple redirect URIs.
  **Tell the user explicitly what redirect URIs to register for both their
  deployed and dev environments.**
- Decide your scopes (see **Scopes** next). You MUST include **`offline_access`**
  or Xero returns no refresh token.

Store **only your app's** Client ID/Secret in **Supabase secrets** (via Lovable's
Supabase integration), plus a 32-byte **`TOKEN_ENC_KEY`** (base64) for encrypting
stored tokens. `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are available to
Edge Functions automatically. **Never** put per-user tokens in secrets — they go
in the database, encrypted.

## Scopes

Scopes are **additive** — request the minimum needed and re-run the OAuth consent
flow to add more later. You **cannot** widen scope on an existing token; the user
must re-consent. To get a refresh token at all you must include `offline_access`.

### User / OpenID scopes (identity)

| Scope | Description |
| --- | --- |
| `offline_access` | required for a refresh token (offline connection) |
| `openid` | use the user's identity |
| `profile` | first name, last name, full name, Xero user id |
| `email` | email address |

### Accounting API — prefer granular scopes

Xero is replacing **broad** scopes with **granular** ones. Since **March 2026**
all new and existing Web/PKCE apps are assigned granular scopes; custom
connections since **29 April 2026**. Broad scopes keep working for apps that
already used them only until **September 2027**. **Always prefer the granular
scopes** — use `accounting.invoices`, NOT `accounting.transactions`.

Deprecated broad scope → the granular scopes that replace it:

| Deprecated (broad) | New granular scopes |
| --- | --- |
| `accounting.transactions` | `accounting.invoices`, `accounting.payments`, `accounting.banktransactions`, `accounting.manualjournals` |
| `accounting.transactions.read` | `accounting.invoices.read`, `accounting.payments.read`, `accounting.banktransactions.read`, `accounting.manualjournals.read` |
| `accounting.reports.read` | `accounting.reports.aged.read`, `accounting.reports.balancesheet.read`, `accounting.reports.banksummary.read`, `accounting.reports.budgetsummary.read`, `accounting.reports.executivesummary.read`, `accounting.reports.profitandloss.read`, `accounting.reports.trialbalance.read`, `accounting.reports.taxreports.read` |

What each granular scope covers (read variants are the same but GET-only):

| Scope | Grants |
| --- | --- |
| `accounting.invoices` | Invoices, CreditNotes, Quotes, PurchaseOrders, RepeatingInvoices, LinkedTransactions, Items |
| `accounting.payments` | Payments, BatchPayments, Overpayments, Prepayments |
| `accounting.banktransactions` | BankTransactions, BankTransfers |
| `accounting.manualjournals` | ManualJournals |
| `accounting.reports.*.read` | the matching report only (balancesheet, profitandloss, trialbalance, taxreports = GSTReport/BASReport, etc.) |

### Accounting scopes that were never broad (use as-is)

| Scope | Grants |
| --- | --- |
| `accounting.settings` / `accounting.settings.read` | Accounts, BrandingThemes, Currencies, Items, InvoiceReminders, Organisation, TaxRates, TrackingCategories, Users |
| `accounting.contacts` / `accounting.contacts.read` | Contacts, ContactGroups |
| `accounting.attachments` / `accounting.attachments.read` | attachments across most resources |
| `accounting.journals.read` | Journals (general ledger) |
| `accounting.budgets.read` | Budgets |

A typical invoicing SaaS therefore wants:
```
openid profile email offline_access accounting.contacts accounting.invoices
```

Store this string in the `XERO_SCOPES` secret (space-separated).

Source of truth (re-check periodically — dates and assignments change):
https://developer.xero.com/documentation/guides/oauth2/scopes

## 2. Per-user token store (Supabase, encrypted, RLS-locked)

One row per (user, Xero tenant). Store the token set encrypted as JSON, plus the
selected `tenantId`. Add a short-lived flow table for the `state` handshake.

```sql
-- xero_connections: one row per (your user, Xero org)
create table public.xero_connections (
  id            uuid primary key default gen_random_uuid(),
  user_id       uuid not null references auth.users(id) on delete cascade,
  tenant_id     text not null,                 -- Xero org id
  tenant_name   text,
  token_set_enc text not null,                 -- AES-256-GCM ciphertext of the token set JSON
  status        text not null default 'active',-- 'active' | 'needs_reconnect'
  expires_at    timestamptz not null,
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now(),
  unique (user_id, tenant_id)
);
alter table public.xero_connections enable row level security;
-- INTENTIONALLY no anon/authenticated policies. Only the service role (used inside
-- Edge Functions) may read or write this table. NEVER add a policy that lets the
-- logged-in user SELECT token_set_enc — tokens must never reach the browser.

-- xero_oauth_flows: short-lived CSRF/state handshake between start and callback
create table public.xero_oauth_flows (
  state         text primary key,
  user_id       uuid not null references auth.users(id) on delete cascade,
  created_at    timestamptz not null default now(),
  expires_at    timestamptz not null
);
alter table public.xero_oauth_flows enable row level security;
-- service-role only, same as above.
```

AES-256-GCM helpers using **Web Crypto** (Deno). Note the blob is **two parts**
(`iv.ciphertext`) because Web Crypto appends the GCM auth tag to the ciphertext —
do NOT port the Node three-part `iv.tag.enc` split, it will fail to decrypt.

```ts
// supabase/functions/_shared/crypto.ts
const rawKey = Uint8Array.from(atob(Deno.env.get("TOKEN_ENC_KEY")!), (c) => c.charCodeAt(0)); // 32 bytes
const key = await crypto.subtle.importKey("raw", rawKey, { name: "AES-GCM" }, false, ["encrypt", "decrypt"]);

const b64   = (b: Uint8Array) => btoa(String.fromCharCode(...b));
const unb64 = (s: string) => Uint8Array.from(atob(s), (c) => c.charCodeAt(0));

export async function encrypt(plain: string) {
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const ct = await crypto.subtle.encrypt({ name: "AES-GCM", iv }, key, new TextEncoder().encode(plain));
  return `${b64(iv)}.${b64(new Uint8Array(ct))}`;            // iv.ciphertext+tag
}
export async function decrypt(blob: string) {
  const [iv, ct] = blob.split(".").map(unb64);
  const pt = await crypto.subtle.decrypt({ name: "AES-GCM", iv }, key, ct);
  return new TextDecoder().decode(pt);
}
```

A shared service-role Supabase client for all the functions:

```ts
// supabase/functions/_shared/admin.ts
import { createClient } from "npm:@supabase/supabase-js";
export const admin = createClient(
  Deno.env.get("SUPABASE_URL")!,
  Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!,   // service role bypasses RLS — Edge Functions only
);
```

## 3. Shared Xero config

Keep one tiny config module so paths and scopes live in one place (mirror the
single-tenant rule: all Xero access in one place).

```ts
// supabase/functions/_shared/xero.ts
export const XERO = {
  clientId:     Deno.env.get("XERO_CLIENT_ID")!,
  clientSecret: Deno.env.get("XERO_CLIENT_SECRET")!,
  redirectUri:  Deno.env.get("XERO_REDIRECT_URI")!,   // the xero-callback function URL
  scopes:       Deno.env.get("XERO_SCOPES") ?? "openid profile email offline_access accounting.contacts accounting.invoices",
  authorize:    "https://login.xero.com/identity/connect/authorize",
  token:        "https://identity.xero.com/connect/token",
  connections:  "https://api.xero.com/connections",
  api:          "https://api.xero.com/api.xro/2.0",
};
export const basicAuth = () => btoa(`${XERO.clientId}:${XERO.clientSecret}`);
const b64url = (b: Uint8Array) => btoa(String.fromCharCode(...b)).replace(/\+/g,"-").replace(/\//g,"_").replace(/=+$/,"");
export const randomState = () => b64url(crypto.getRandomValues(new Uint8Array(16)));
```

## 4. Start consent (Edge Function, knows the user)

`state` is your CSRF defense and your link back to the initiating user. Persist it
server-side (the `xero_oauth_flows` row) — never trust it blind on return. This
function requires the user's Supabase session JWT so you know *which of your users*
started the flow (keep its `verify_jwt = true`, the default).

```ts
// supabase/functions/xero-start/index.ts   (verify_jwt = true)
import { admin } from "../_shared/admin.ts";
import { XERO, randomState } from "../_shared/xero.ts";

Deno.serve(async (req) => {
  const jwt = req.headers.get("Authorization")?.replace("Bearer ", "");
  const { data: { user } } = await admin.auth.getUser(jwt!);
  if (!user) return new Response("Unauthorized", { status: 401 });

  const state = randomState();
  await admin.from("xero_oauth_flows").insert({
    state, user_id: user.id,
    expires_at: new Date(Date.now() + 10 * 60_000).toISOString(),
  });

  const url = new URL(XERO.authorize);
  url.searchParams.set("response_type", "code");
  url.searchParams.set("client_id", XERO.clientId);
  url.searchParams.set("redirect_uri", XERO.redirectUri);
  url.searchParams.set("scope", XERO.scopes);
  url.searchParams.set("state", state);

  return new Response(JSON.stringify({ url: url.toString() }), {
    headers: { "Content-Type": "application/json" },
  });
});
```

> **Why no PKCE here.** Your Edge Functions can hold the client secret, so the
> standard confidential web-app flow is correct: the **secret** authenticates the
> token exchange and `state` covers CSRF. PKCE exists for public clients that
> *can't* keep a secret (mobile/SPA); bolting it onto a confidential server flow
> adds a `code_verifier` you'd have to persist across the two stateless
> invocations for no security gain. Don't use PKCE for this — secret + `state`.

**Lovable preview iframe gotcha.** Lovable's in-editor preview renders your app in
an iframe, and Xero's login page refuses to be framed (`frame-ancestors`), so a
same-frame `window.location.href = consentUrl` shows a **blank white page**. Detect
the iframe and open consent in a new top-level tab. Open the tab **synchronously in
the click handler** (before the `await`) so it isn't blocked as a popup:

```ts
const inIframe = window.self !== window.top;
const popup = inIframe ? window.open("about:blank", "_blank") : null;
const { data } = await supabase.functions.invoke("xero-start");   // sends the user's JWT
if (popup) popup.location.href = data.url;          // iframe (preview): new tab
else window.location.href = data.url;               // published .lovable.app: same tab
```

The published app at `<your-app>.lovable.app` is **not** iframed, so the same-tab
redirect works there — handle both. After OAuth, let TanStack Query's
refetch-on-focus refresh the connection list in the original tab.

## 5. Callback — exchange, pick tenant(s), store

Xero redirects the **browser** here with no Supabase JWT, so this function MUST be
public: set **`verify_jwt = false`** for `xero-callback` (in
`supabase/config.toml`, e.g. `[functions.xero-callback]\nverify_jwt = false`, or in
the function's settings). You enforce trust via `state`, not a JWT.

```ts
// supabase/functions/xero-callback/index.ts   (verify_jwt = false)
import { admin } from "../_shared/admin.ts";
import { XERO, basicAuth } from "../_shared/xero.ts";
import { encrypt } from "../_shared/crypto.ts";

const APP_URL = Deno.env.get("APP_URL")!;   // e.g. https://<your-app>.lovable.app
const back = (q: string) => new Response(null, { status: 302, headers: { Location: `${APP_URL}/xero${q}` } });

Deno.serve(async (req) => {
  const u = new URL(req.url);
  const code = u.searchParams.get("code");
  const state = u.searchParams.get("state");
  if (!code || !state) return back("?error=missing_params");

  const { data: flow } = await admin.from("xero_oauth_flows").select("*").eq("state", state).single();
  if (!flow || new Date(flow.expires_at) < new Date()) return back("?error=bad_state");

  // exchange the code for tokens (confidential flow: Basic auth with the secret)
  const tokenRes = await fetch(XERO.token, {
    method: "POST",
    headers: { Authorization: `Basic ${basicAuth()}`, "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      grant_type: "authorization_code",
      code,
      redirect_uri: XERO.redirectUri,
    }),
  });
  if (!tokenRes.ok) return back("?error=token_exchange");
  const tokenSet = await tokenRes.json();   // { access_token, refresh_token, expires_in, ... }

  // which orgs did this single consent authorize?
  const conns = await fetch(XERO.connections, {
    headers: { Authorization: `Bearer ${tokenSet.access_token}` },
  }).then((r) => r.json());

  // A single consent can authorize MULTIPLE orgs that all SHARE ONE token set.
  // Don't assume conns[0]. Encrypt ONCE and write the IDENTICAL ciphertext to every
  // row — this lets refresh (§6) find and update all sibling rows together.
  const enc = await encrypt(JSON.stringify(tokenSet));
  const expiresAt = new Date(Date.now() + tokenSet.expires_in * 1000).toISOString();
  for (const c of conns) {
    await admin.from("xero_connections").upsert(
      { user_id: flow.user_id, tenant_id: c.tenantId, tenant_name: c.tenantName,
        token_set_enc: enc, status: "active", expires_at: expiresAt },
      { onConflict: "user_id,tenant_id" },
    );
  }
  await admin.from("xero_oauth_flows").delete().eq("state", state);
  return back("?connected=1");
});
```

## 6. Use it per request (refresh + persist rotation)

A stateless function instantiates fresh each call, so "never cache a client" is
automatic. Refresh slightly early, rotate the refresh token, and update **all
sibling rows** that shared the old token set.

```ts
// supabase/functions/_shared/getToken.ts
import { admin } from "./admin.ts";
import { XERO, basicAuth } from "./xero.ts";
import { encrypt, decrypt } from "./crypto.ts";

export async function getAccessToken(userId: string, tenantId: string) {
  const { data: conn } = await admin.from("xero_connections")
    .select("*").eq("user_id", userId).eq("tenant_id", tenantId).single();
  if (!conn) throw new Error("User has not connected this Xero org");

  let tokenSet = JSON.parse(await decrypt(conn.token_set_enc));

  if (new Date(conn.expires_at) <= new Date(Date.now() + 60_000)) {  // refresh with headroom
    const res = await fetch(XERO.token, {
      method: "POST",
      headers: { Authorization: `Basic ${basicAuth()}`, "Content-Type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({ grant_type: "refresh_token", refresh_token: tokenSet.refresh_token }),
    });
    if (!res.ok) {
      // refresh token is dead → surface a reconnect state, don't loop forever
      await admin.from("xero_connections").update({ status: "needs_reconnect" })
        .eq("user_id", userId).eq("token_set_enc", conn.token_set_enc);
      throw new Error("Xero refresh failed — prompt reconnect");
    }
    tokenSet = await res.json();                         // refresh_token ROTATES — must persist
    const enc = await encrypt(JSON.stringify(tokenSet));
    const expiresAt = new Date(Date.now() + tokenSet.expires_in * 1000).toISOString();
    // CRITICAL multi-tenant trap: orgs that shared ONE token set all hold the SAME
    // (now-rotated, invalidated) refresh token. Update EVERY sibling row, matched on
    // the OLD ciphertext, or the others flip to needs_reconnect on their next call.
    await admin.from("xero_connections")
      .update({ token_set_enc: enc, expires_at: expiresAt, updated_at: new Date().toISOString() })
      .eq("user_id", userId).eq("token_set_enc", conn.token_set_enc);
  }
  return tokenSet.access_token as string;
}
```

Example write — note the explicit `Xero-Tenant-Id` header (there is no implicit
"current org"), `summarizeErrors=false`, `unitdp=4`, and an idempotency key:

```ts
const accessToken = await getAccessToken(userId, tenantId);
await fetch(`${XERO.api}/Invoices?unitdp=4&summarizeErrors=false`, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${accessToken}`,
    "Xero-Tenant-Id": tenantId,                              // REQUIRED — wrong id writes to wrong books
    "Idempotency-Key": order.idempotencyKey ?? crypto.randomUUID(),
    "Content-Type": "application/json",
    Accept: "application/json",
  },
  body: JSON.stringify({ Invoices: [{ /* Type, Contact, LineItems, ... */ }] }),
});
```

## Date + money + tax normalisation

- **Dates.** Because you're calling the REST API directly (not the SDK), responses
  contain Xero's Microsoft-JSON date form (`/Date(1518685950940+0000)/`), which
  `new Date()` **cannot** parse. Always run incoming dates through a normaliser and
  emit ISO 8601:

  ```ts
  export function normalizeXeroDate(v: unknown): Date | null {
    if (v == null) return null;
    if (v instanceof Date) return isNaN(v.getTime()) ? null : v;
    if (typeof v === "string") {
      const m = v.match(/\/Date\((-?\d+)([+-]\d{4})?\)\//);
      const d = m ? new Date(Number(m[1])) : new Date(v);
      return isNaN(d.getTime()) ? null : d;
    }
    return null;
  }
  ```
  When **sending** dates, use `YYYY-MM-DD`.

- **Money.** Send numbers, not pre-formatted currency strings. Control rounding
  with the `unitdp` query param (unit decimal places, up to 4). If you store money
  in Postgres `numeric` (returned as strings by Supabase), convert back to numbers
  before any client-side schema parse. Amounts like `AmountDue` come back as numbers.

- **Tax is the org's, not yours.** Each connected organisation applies **its own**
  tax config, so `AmountDue` / `Total` may exceed the sum of your line amounts —
  e.g. a 15% GST org turns a 2495 line into 2869.25. The `UnitAmount` you posted is
  still correct; do **not** "fix" this discrepancy. Set `LineAmountTypes`
  (`Exclusive`, `Inclusive`, `NoTax`) deliberately if you need control. **In
  multi-tenant this varies per customer** — never hard-code a tax assumption;
  whatever each org returns is correct for that org.

## Idempotency (avoid duplicate writes)

Writes can be duplicated by retries and double-clicks. Defend at two layers.

**Layer 1 — app level (cheapest):** don't call Xero twice for the same logical
action. Upsert contacts by email (see gotchas) and dedup the order on a stable
identifier (client-supplied token, or a Postgres unique constraint) so a
double-submitted form never reaches Xero twice.

**Layer 2 — Xero idempotency (the safety net):** pass the `Idempotency-Key` header
on the mutating call. Xero caches the first response and returns it for any repeat
with the same key, so a network-timeout retry can't create a second invoice.

Rules that bite (from Xero's idempotency guide):

- **Only POST/PUT/PATCH** honour the key; it is ignored on GET.
- **Max 128 characters.** A single `crypto.randomUUID()` (36 chars) is plenty —
  use that as the default. Xero's "concatenate four UUIDs" advice is for extreme
  volume; if you do that, strip hyphens (4 × 32 = 128) to stay in limit.
- **Keys expire after ~6 minutes.** They protect transient retries, not long-term
  dedup. Long-term dedup is Layer 1's job.
- **Same key + a *different* request → `400` "used with a different request".** A
  request differs if the URL, body, or method changes. Persist the key alongside
  the record and reuse the *identical* request when retrying.
- **Errors are cached too.** A keyed request that errored internally returns the
  *same cached error* on retry even after the underlying problem is fixed. To
  recover, **GET to check whether the resource was actually created**; if not,
  create again with a **new** key.

When a user *re-submits* a previously **failed** order later, mint a **new** key —
but first GET to confirm no invoice was actually created, so you don't duplicate
one that silently succeeded.

## Other Xero gotchas (these bite in production)

- **`offline_access` is mandatory for refresh.** Omit it and you get an access
  token that dies in ~30 min with no way to refresh — the user must re-consent.
- **Token lifetimes:** access token ~30 min; refresh token ~60 days and **rotates
  on every refresh**. You MUST persist the new refresh token each time or the next
  refresh fails. A refresh token also dies if unused for 60 days.
- **Always send `Xero-Tenant-Id` explicitly** on every API call. There is no
  implicit "current org." With multiple connected orgs, the wrong id silently
  writes to the wrong company's books.
- **Re-fetch `/connections` periodically**, not just at callback — the connected-org
  list can change as users add/remove orgs.
- **Contacts are not auto-deduplicated by email.** Posting an invoice with a new
  contact name creates a NEW contact every time. Upsert yourself: look up by email
  (or `ContactID`) via `GET /Contacts` and reuse the id; otherwise you litter the
  org with duplicates.
- **Invoice status flow:** `DRAFT` → `SUBMITTED` → `AUTHORISED`. Only `AUTHORISED`
  invoices are real receivables and expose an online payment URL. Use `ACCREC`
  (sales) vs `ACCPAY` (bills) correctly.
- **Rate limits (per tenant):** ~60 calls/minute, 5,000/day, plus an app-wide
  per-minute cap and ~5 concurrent requests. On **429** read the `Retry-After`
  header and back off — multi-tenant apps hit this fast when looping over orgs.
  Spread/queue calls per tenant.
- **`summarizeErrors=false`** on batch create calls returns per-item validation
  errors instead of one opaque failure.

### Lovable / Supabase-specific gotchas

- **`xero-callback` must be `verify_jwt = false`.** Xero's browser redirect carries
  no Supabase JWT; leave it on and every connect returns 401. Trust comes from
  `state`, which you verify against the `xero_oauth_flows` row.
- **The `state` value MUST be persisted.** Edge Functions are stateless — start
  and callback are separate invocations with no shared memory.
- **Web Crypto blob ≠ Node blob.** Web Crypto appends the GCM tag to the ciphertext
  → two-part `iv.ciphertext`. Do not port a Node three-part `iv.tag.enc` splitter.
- **Service-role key stays in Edge Functions.** Never expose it (or tokens) to the
  browser. The token tables have RLS with **no** user-facing policies.
- **`npm:xero-node` may not load** under the edge runtime; the `fetch`-based flow
  here avoids the dependency entirely. If you use the SDK for read/write calls,
  test that it imports first.

## Security checklist (do not skip)

- **Encrypt** the stored token set at rest; the key (`TOKEN_ENC_KEY`) lives in
  Supabase secrets.
- **Verify `state`** on every callback against the persisted flow row (CSRF).
- **`xero-callback` is public (`verify_jwt = false`); every other Xero function
  requires the user's JWT.**
- **RLS on token tables with no user policies** — only the service role touches them.
- **Never expose tokens or the service-role key to the browser** or any
  client-readable storage.
- On a failed refresh, **mark the connection `needs_reconnect` and prompt
  re-connect** — do not silently retry forever.
- Keep all Xero access in **one place** (`_shared/xero.ts`, `_shared/getToken.ts`)
  so paths, tenant handling, and gotchas live together.

## Don't want to hand-roll it?

Managed token-vault platforms (**Nango**, **Composio**, etc.) implement per-user
OAuth + refresh + encryption for Xero and many other providers, and call out to
them from an Edge Function. For a multi-provider SaaS this is often the pragmatic
choice.

## Verify

1. Two app users each connect a **different** Xero org; confirm two distinct token
   rows and that each user's writes land in their own org (check `Xero-Tenant-Id`).
2. Force a near-expiry token and confirm transparent refresh **and** that the
   rotated refresh token is persisted to every sibling row.
3. Revoke the app in one user's Xero and confirm your app surfaces a "reconnect"
   state (`needs_reconnect`) instead of looping on failed refreshes.
4. Post the same invoice twice with the same `Idempotency-Key` and confirm only one
   invoice is created.
5. Hit `xero-callback` directly in a browser and confirm it isn't blocked by a 401
   (i.e. `verify_jwt = false` is actually set).
