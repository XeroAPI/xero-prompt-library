---
name: xero-integration-single-account
description: Use when YOU (the app builder) connect ONE Xero organisation that you own, and the whole app reads/writes that single set of books — internal dashboards, a single-merchant storefront, a back-office tool, or a prototype. Covers the OAuth2 lifecycle in Supabase Edge Functions, one encrypted token set with RLS, refresh/rotation, scopes, and Xero API gotchas (granular scope migration, date/money/tax normalisation, idempotent writes, rate limits). Not for apps where each end user connects their OWN Xero org — use xero-integration-multi-account for that.
---

# Single-account Xero (you connect one Xero org you own)

This skill is for a **single-org app built in Lovable**: there is exactly one Xero
organisation — **yours** — and every visitor sees the same books through your one
connection. There are no per-user Xero accounts.

> **First, make sure this is the right variant.** Lovable has no managed
> single-principal Xero connector, so you own the OAuth lifecycle either way. The
> deciding question is *how many Xero logins exist*:
>
> - **One — yours.** Internal dashboard over your own books, a single-merchant
>   store, a back-office tool, a prototype. **This skill.** One stored token set,
>   no per-user keying.
> - **Many — one per customer.** Each end user links *their own* Xero. Stop and
>   use **`xero-integration-multi-account`** instead. You'd need a per-user token
>   vault, tenant selection, and multi-tenant refresh rotation — none of which is
>   below.
>
> Note: one Xero login can still expose several *organisations* (`GET
> /connections`). That's still **one** principal, so it stays in this skill — you
> just pick which org(s) you mean.

## Where this code runs in Lovable

Lovable's backend is **Supabase**: Postgres for storage, **Edge Functions (Deno)**
for server code, Supabase secrets for config. All OAuth and token handling lives in
**Edge Functions** — never in the React/Vite frontend, never in the browser. Two
runtime facts shape the code below:

- **Edge Functions are stateless.** The connect step and the callback are separate
  cold-start invocations with no shared memory, so the `state` value must be
  persisted to a table between them.
- **Use `fetch`, not the `xero-node` SDK, for OAuth.** The SDK keeps OAuth state in
  process memory, which doesn't survive across two stateless invocations, and its
  `openid-client` dependency is awkward under Deno. The whole lifecycle is three
  `fetch` calls (authorize redirect, token exchange, refresh). You *may* use
  `npm:xero-node` for read/write API calls if it imports in your edge runtime, but
  keep the flow `fetch`-based. **Crypto is Web Crypto (`crypto.subtle`)**, not Node
  `crypto`.

## 1. Register your own Xero app

In the Xero developer portal (developer.xero.com) create a **Web app** OAuth2 app
(confidential — it holds a secret, which your Edge Functions can):
- Grab **Client ID** and **Client Secret**.
- Register your redirect URI as your **callback Edge Function URL**, exactly:
  `https://<project-ref>.supabase.co/functions/v1/xero-callback`
  (your `<project-ref>` is in Lovable's Supabase settings). Add your local dev
  Supabase URL too if you test locally. **Tell the user explicitly what redirect
  URIs to register for both their deployed and dev environments.**
- Decide your scopes (see **Scopes** next). You MUST include **`offline_access`**
  or Xero returns no refresh token.

Store **your app's** Client ID/Secret in **Supabase secrets** (via Lovable's
Supabase integration), plus a 32-byte **`TOKEN_ENC_KEY`** (base64) for encrypting
the stored token. `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are available to
Edge Functions automatically. The refresh token **rotates on every refresh**, so it
must live in a **mutable** store (the database) — never in a Supabase secret.

## Scopes

Scopes are **additive** — request the minimum and re-run consent to add more later.
You **cannot** widen scope on an existing token; you must re-consent. To get a
refresh token at all you must include `offline_access`.

(This table is identical to the multi-account skill — keep the two in sync if Xero's
assignments change.)

### User / OpenID scopes (identity)

| Scope | Description |
| --- | --- |
| `offline_access` | required for a refresh token (offline connection) |
| `openid` | use the user's identity |
| `profile` | first name, last name, full name, Xero user id |
| `email` | email address |

### Accounting API — prefer granular scopes

Xero is replacing **broad** scopes with **granular** ones. Since **March 2026** all
new and existing Web/PKCE apps are assigned granular scopes; custom connections
since **29 April 2026**. Broad scopes keep working for apps that already used them
only until **September 2027**. **Always prefer the granular scopes** — use
`accounting.invoices`, NOT `accounting.transactions`.

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

A typical single-org dashboard wants something like:
```
openid profile email offline_access accounting.contacts.read accounting.invoices.read accounting.reports.profitandloss.read
```
Store the chosen string in the `XERO_SCOPES` secret (space-separated).

Source of truth (re-check periodically — dates and assignments change):
https://developer.xero.com/documentation/guides/oauth2/scopes

## 2. Token store (one encrypted row, RLS-locked)

There is no end user, so there is no `user_id`. Key by `tenant_id` so the table
naturally holds your one org (one row) — or a few rows if your single login owns
several orgs you want to act on. Add a short-lived flow table for the `state`
handshake.

```sql
-- xero_connection: the org(s) under YOUR single Xero login
create table public.xero_connection (
  tenant_id     text primary key,              -- Xero org id
  tenant_name   text,
  token_set_enc text not null,                 -- AES-256-GCM ciphertext of the token set JSON
  status        text not null default 'active',-- 'active' | 'needs_reconnect'
  expires_at    timestamptz not null,
  updated_at    timestamptz not null default now()
);
alter table public.xero_connection enable row level security;
-- INTENTIONALLY no anon/authenticated policies. Only the service role (used inside
-- Edge Functions) may read or write this table. NEVER let the browser read
-- token_set_enc — surface only derived data (reports, invoices) through your own API.

-- xero_oauth_flows: short-lived CSRF/state handshake between connect and callback
create table public.xero_oauth_flows (
  state      text primary key,
  created_at timestamptz not null default now(),
  expires_at timestamptz not null
);
alter table public.xero_oauth_flows enable row level security;  -- service-role only
```

AES-256-GCM helpers using **Web Crypto** (Deno). The blob is **two parts**
(`iv.ciphertext`) because Web Crypto appends the GCM auth tag to the ciphertext — do
NOT use a Node-style three-part `iv.tag.enc` split, it will fail to decrypt.

```ts
// supabase/functions/_shared/crypto.ts
const rawKey = Uint8Array.from(atob(Deno.env.get("TOKEN_ENC_KEY")!), (c) => c.charCodeAt(0)); // 32 bytes
const key = await crypto.subtle.importKey("raw", rawKey, { name: "AES-GCM" }, false, ["encrypt", "decrypt"]);
const b64   = (b: Uint8Array) => btoa(String.fromCharCode(...b));
const unb64 = (s: string) => Uint8Array.from(atob(s), (c) => c.charCodeAt(0));

export async function encrypt(plain: string) {
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const ct = await crypto.subtle.encrypt({ name: "AES-GCM", iv }, key, new TextEncoder().encode(plain));
  return `${b64(iv)}.${b64(new Uint8Array(ct))}`;
}
export async function decrypt(blob: string) {
  const [iv, ct] = blob.split(".").map(unb64);
  const pt = await crypto.subtle.decrypt({ name: "AES-GCM", iv }, key, ct);
  return new TextDecoder().decode(pt);
}
```

Shared service-role client and Xero config:

```ts
// supabase/functions/_shared/admin.ts
import { createClient } from "npm:@supabase/supabase-js";
export const admin = createClient(
  Deno.env.get("SUPABASE_URL")!,
  Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!,   // service role bypasses RLS — Edge Functions only
);

// supabase/functions/_shared/xero.ts
export const XERO = {
  clientId:     Deno.env.get("XERO_CLIENT_ID")!,
  clientSecret: Deno.env.get("XERO_CLIENT_SECRET")!,
  redirectUri:  Deno.env.get("XERO_REDIRECT_URI")!,   // the xero-callback function URL
  scopes:       Deno.env.get("XERO_SCOPES") ?? "openid profile email offline_access accounting.contacts.read accounting.invoices.read",
  authorize:    "https://login.xero.com/identity/connect/authorize",
  token:        "https://identity.xero.com/connect/token",
  connections:  "https://api.xero.com/connections",
  api:          "https://api.xero.com/api.xro/2.0",
};
export const basicAuth = () => btoa(`${XERO.clientId}:${XERO.clientSecret}`);
const b64url = (b: Uint8Array) => btoa(String.fromCharCode(...b)).replace(/\+/g,"-").replace(/\//g,"_").replace(/=+$/,"");
export const randomState = () => b64url(crypto.getRandomValues(new Uint8Array(16)));
```

## 3. Connect (one-time) — start consent

Connecting your org is normally a **one-time admin action**. Use the standard
confidential web-app flow: the client **secret** authenticates the token exchange
and `state` covers CSRF — no PKCE (that's for public clients that can't keep a
secret).

> **Gate this function.** Anyone who can reach `xero-start` can kick off a connect
> and, via the callback, write a row into your single token table. Require an
> authenticated **admin** (your app's own notion of admin) — keep `verify_jwt =
> true` and check the caller. Otherwise a stranger could attach a different org to
> your app's books.

```ts
// supabase/functions/xero-start/index.ts   (verify_jwt = true)
import { admin } from "../_shared/admin.ts";
import { XERO, randomState } from "../_shared/xero.ts";

Deno.serve(async (req) => {
  const jwt = req.headers.get("Authorization")?.replace("Bearer ", "");
  const { data: { user } } = await admin.auth.getUser(jwt!);
  if (!user /* || !isAdmin(user) */) return new Response("Forbidden", { status: 403 });

  const state = randomState();
  await admin.from("xero_oauth_flows").insert({
    state, expires_at: new Date(Date.now() + 10 * 60_000).toISOString(),
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

**Lovable preview iframe gotcha.** Lovable's in-editor preview renders your app in
an iframe, and Xero's login refuses to be framed (`frame-ancestors`), so a
same-frame redirect shows a **blank white page**. Detect the iframe and open consent
in a new top-level tab, opening the tab **synchronously in the click handler**
(before the `await`) so it isn't blocked as a popup:

```ts
const inIframe = window.self !== window.top;
const popup = inIframe ? window.open("about:blank", "_blank") : null;
const { data } = await supabase.functions.invoke("xero-start");   // sends your admin JWT
if (popup) popup.location.href = data.url;          // iframe (preview): new tab
else window.location.href = data.url;               // published .lovable.app: same tab
```

The published app at `<your-app>.lovable.app` is not iframed, so the same-tab
redirect works there — handle both.

## 4. Callback — exchange, store your org

Xero redirects the **browser** here with no Supabase JWT, so this function MUST be
public: set **`verify_jwt = false`** for `xero-callback` (in `supabase/config.toml`,
e.g. `[functions.xero-callback]\nverify_jwt = false`). Trust comes from `state`.

```ts
// supabase/functions/xero-callback/index.ts   (verify_jwt = false)
import { admin } from "../_shared/admin.ts";
import { XERO, basicAuth } from "../_shared/xero.ts";
import { encrypt } from "../_shared/crypto.ts";

const APP_URL = Deno.env.get("APP_URL")!;   // e.g. https://<your-app>.lovable.app
const back = (q: string) => new Response(null, { status: 302, headers: { Location: `${APP_URL}/settings/xero${q}` } });

Deno.serve(async (req) => {
  const u = new URL(req.url);
  const code = u.searchParams.get("code");
  const state = u.searchParams.get("state");
  if (!code || !state) return back("?error=missing_params");

  const { data: flow } = await admin.from("xero_oauth_flows").select("*").eq("state", state).single();
  if (!flow || new Date(flow.expires_at) < new Date()) return back("?error=bad_state");

  const tokenRes = await fetch(XERO.token, {
    method: "POST",
    headers: { Authorization: `Basic ${basicAuth()}`, "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({ grant_type: "authorization_code", code, redirect_uri: XERO.redirectUri }),
  });
  if (!tokenRes.ok) return back("?error=token_exchange");
  const tokenSet = await tokenRes.json();   // { access_token, refresh_token, expires_in, ... }

  // Which org(s) does your login expose? Usually one. If several, either store the
  // one you mean (filter by name/id) or store all — they share this one token set.
  const conns = await fetch(XERO.connections, {
    headers: { Authorization: `Bearer ${tokenSet.access_token}` },
  }).then((r) => r.json());

  const enc = await encrypt(JSON.stringify(tokenSet));
  const expiresAt = new Date(Date.now() + tokenSet.expires_in * 1000).toISOString();
  for (const c of conns) {
    await admin.from("xero_connection").upsert(
      { tenant_id: c.tenantId, tenant_name: c.tenantName, token_set_enc: enc, status: "active", expires_at: expiresAt },
      { onConflict: "tenant_id" },
    );
  }
  await admin.from("xero_oauth_flows").delete().eq("state", state);
  return back("?connected=1");
});
```

## 5. Use it per request (refresh + persist rotation)

Load the row, refresh slightly early, and persist the rotated refresh token. With a
single org this is just a one-row update.

```ts
// supabase/functions/_shared/getToken.ts
import { admin } from "./admin.ts";
import { XERO, basicAuth } from "./xero.ts";
import { encrypt, decrypt } from "./crypto.ts";

export async function getAccessToken(tenantId?: string) {
  const q = admin.from("xero_connection").select("*");
  const { data: conn } = tenantId ? await q.eq("tenant_id", tenantId).single() : await q.limit(1).single();
  if (!conn) throw new Error("Xero is not connected yet");

  let tokenSet = JSON.parse(await decrypt(conn.token_set_enc));

  if (new Date(conn.expires_at) <= new Date(Date.now() + 60_000)) {   // refresh with headroom
    const res = await fetch(XERO.token, {
      method: "POST",
      headers: { Authorization: `Basic ${basicAuth()}`, "Content-Type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({ grant_type: "refresh_token", refresh_token: tokenSet.refresh_token }),
    });
    if (!res.ok) {
      await admin.from("xero_connection").update({ status: "needs_reconnect" }).eq("tenant_id", conn.tenant_id);
      throw new Error("Xero refresh failed — reconnect required");
    }
    tokenSet = await res.json();                          // refresh_token ROTATES — must persist
    await admin.from("xero_connection").update({
      token_set_enc: await encrypt(JSON.stringify(tokenSet)),
      expires_at: new Date(Date.now() + tokenSet.expires_in * 1000).toISOString(),
      updated_at: new Date().toISOString(),
    }).eq("tenant_id", conn.tenant_id);
    // If your one login stored MULTIPLE org rows, they share this token set — match
    // on the old ciphertext and update all of them, not just this row.
  }
  return { accessToken: tokenSet.access_token as string, tenantId: conn.tenant_id as string };
}
```

Example read (note the explicit `Xero-Tenant-Id` header — there is no implicit
"current org"):

```ts
const { accessToken, tenantId } = await getAccessToken();
const pnl = await fetch(`${XERO.api}/Reports/ProfitAndLoss`, {
  headers: { Authorization: `Bearer ${accessToken}`, "Xero-Tenant-Id": tenantId, Accept: "application/json" },
}).then((r) => r.json());
```

Example write (idempotency key, `summarizeErrors=false`, `unitdp=4`):

```ts
const { accessToken, tenantId } = await getAccessToken();
await fetch(`${XERO.api}/Invoices?unitdp=4&summarizeErrors=false`, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${accessToken}`,
    "Xero-Tenant-Id": tenantId,
    "Idempotency-Key": crypto.randomUUID(),
    "Content-Type": "application/json",
    Accept: "application/json",
  },
  body: JSON.stringify({ Invoices: [{ /* Type, Contact, LineItems, ... */ }] }),
});
```

## Date + money + tax normalisation

- **Dates.** Calling REST directly, responses contain Xero's Microsoft-JSON date
  form (`/Date(1518685950940+0000)/`), which `new Date()` can't parse. Normalise
  incoming dates and emit ISO 8601; send dates as `YYYY-MM-DD`.

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

- **Money.** Send numbers, not formatted currency strings. Control rounding with the
  `unitdp` query param (up to 4). Postgres `numeric` comes back as strings via
  Supabase — convert to numbers before any client-side parse. `AmountDue` is a number.

- **Tax is your org's.** Your organisation's tax config is applied to invoices, so
  `AmountDue` / `Total` may exceed the sum of your line amounts (e.g. 15% GST turns a
  2495 line into 2869.25). The `UnitAmount` you posted is still correct — don't
  "fix" the discrepancy. Set `LineAmountTypes` (`Exclusive`, `Inclusive`, `NoTax`)
  deliberately if you need control.

## Idempotency (avoid duplicate writes)

Defend at two layers.

**Layer 1 — app level:** dedup the logical action (a stable order id, a unique
constraint) so a double-submit never reaches Xero twice; upsert contacts by email
(see gotchas).

**Layer 2 — Xero idempotency:** pass an `Idempotency-Key` header on mutating calls.
Xero caches the first response and replays it for repeats with the same key.

Rules that bite:
- **Only POST/PUT/PATCH** honour the key; ignored on GET.
- **Max 128 chars.** A single `crypto.randomUUID()` (36) is plenty.
- **Keys expire after ~6 minutes** — they protect transient retries, not long-term
  dedup (that's Layer 1).
- **Same key + a different request → `400`.** Persist the key with the record and
  reuse the identical request when retrying.
- **Errors are cached too.** A keyed request that errored replays the cached error
  on retry; to recover, GET to check whether the resource was actually created, and
  if not, retry with a **new** key.

## Other Xero gotchas (these bite in production)

- **`offline_access` is mandatory for refresh.** Omit it and the access token dies
  in ~30 min with no way to refresh — you'd have to reconnect.
- **Token lifetimes:** access token ~30 min; refresh token ~60 days and **rotates on
  every refresh**. Persist the new refresh token each time or the next refresh fails.
  A refresh token also dies if unused for 60 days — a low-traffic internal tool can
  hit this, so consider a scheduled keep-alive refresh.
- **Always send `Xero-Tenant-Id`** on every API call — there is no implicit org.
- **Contacts aren't deduplicated by email.** Posting an invoice with a new contact
  name creates a NEW contact each time. Look up by email/`ContactID` via
  `GET /Contacts` and reuse the id.
- **Invoice status flow:** `DRAFT` → `SUBMITTED` → `AUTHORISED`; only `AUTHORISED`
  are real receivables and expose an online payment URL. Use `ACCREC` (sales) vs
  `ACCPAY` (bills) correctly.
- **Rate limits:** ~60 calls/minute, 5,000/day, ~5 concurrent. On **429** read
  `Retry-After` and back off.
- **`summarizeErrors=false`** on batch creates returns per-item validation errors.

### Lovable / Supabase-specific gotchas

- **`xero-callback` must be `verify_jwt = false`.** Xero's browser redirect carries
  no Supabase JWT; leave it on and the connect returns 401. Trust comes from `state`.
- **`xero-start` must be gated** to an admin — it can attach an org to your books.
- **The `state` value MUST be persisted.** Edge Functions are stateless — connect
  and callback are separate invocations with no shared memory.
- **Web Crypto blob ≠ Node blob.** Web Crypto appends the GCM tag to the ciphertext →
  two-part `iv.ciphertext`. Don't port a Node three-part `iv.tag.enc` splitter.
- **Service-role key stays in Edge Functions.** Never expose it (or the token) to the
  browser. The token table has RLS with no user-facing policies.
- **`npm:xero-node` may not load** under the edge runtime; the `fetch`-based flow
  here avoids the dependency. If you use the SDK for read/write calls, test the
  import first.

## Security checklist (do not skip)

- **Encrypt** the stored token set at rest; the key (`TOKEN_ENC_KEY`) lives in
  Supabase secrets, the rotating token in the DB.
- **Verify `state`** on every callback against the persisted flow row (CSRF).
- **`xero-callback` is public (`verify_jwt = false`); `xero-start` requires an admin
  JWT.**
- **RLS on the token table with no user policies** — only the service role touches it.
- **Never expose the token or service-role key to the browser** — surface only
  derived data through your own API.
- On a failed refresh, **mark `needs_reconnect` and prompt re-connect** — don't loop.
- Keep all Xero access in **one place** (`_shared/xero.ts`, `_shared/getToken.ts`).

## Verify

1. Connect your org once; confirm a single `xero_connection` row and that a read
   (e.g. Profit & Loss) returns your books.
2. Force a near-expiry token and confirm transparent refresh **and** that the rotated
   refresh token is persisted.
3. Revoke the app in Xero and confirm the app surfaces a reconnect state
   (`needs_reconnect`) instead of looping on failed refreshes.
4. Hit `xero-callback` directly in a browser and confirm it isn't blocked by 401
   (i.e. `verify_jwt = false` is set).
5. Confirm `xero-start` rejects a non-admin caller.
