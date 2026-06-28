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

**If your Lovable project has its own server runtime, port these handlers to it.**
Lovable supports stacks beyond the React/Vite + Edge Functions default, and some come
with explicit guidance *against* using Supabase Edge Functions as the server layer. A
**TanStack Start** project, for instance, should put this logic in `createServerFn`
server functions; a Next.js project would use route handlers or server actions. The
mapping is mechanical: each piece below (`xero-start`, `xero-callback`, and the
per-request token helper) becomes one server function — only the request/response
wrapper changes. Everything that matters is identical: persisting `state`, the
encrypted token row, refresh-rotation persistence, the `Xero-Tenant-Id` header. Two
things shift on a Node-based runtime: you may use `node:crypto` (or the `xero-node`
SDK) instead of Web Crypto, since the stateless/Deno objections above are
Edge-Function-specific; and secrets load via your stack's own env mechanism, so the
Edge-Function "redeploy after changing a secret" rule (section 1) may not apply. The
SQL, scopes, idempotency, and Xero gotchas carry over unchanged.

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

> **Redeploy after adding or changing any secret — this is the #1 first failure.**
> Edge Functions snapshot their environment at deploy time and do **not** pick up
> newly-added or edited secrets until you redeploy them. The classic symptom is a
> Xero authorize URL containing `client_id=undefined` (or a 401 on the token
> exchange) immediately after you set `XERO_CLIENT_ID` — the secret exists, but the
> running function was deployed before it did. Set every secret first, **then**
> (re)deploy `xero-start` and `xero-callback`. Rotating `TOKEN_ENC_KEY` or the client
> secret later? Redeploy again. (This rule is Edge-Function-specific — see the
> stack note above if you're not on Edge Functions.)

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

**Read vs write — choose deliberately; you can't change it later without
re-consent.** The `.read` scopes are GET-only. If your app **creates or modifies**
anything (invoices, contacts, payments), you must request the **write** (non-`.read`)
scope for each. Shipping a read-only token and then calling `POST /Invoices` fails at
runtime with a 403 / insufficient-scope error — and because scopes **cannot be
widened on an existing token**, the only fix is to send the user back through the
whole consent flow. So decide up front: read, write, or both.

A **read-only dashboard** (reports + listing) wants something like:
```
openid profile email offline_access accounting.contacts.read accounting.invoices.read accounting.reports.profitandloss.read
```

A **write / invoicing app** (creates invoices, upserts contacts) needs the write
scopes:
```
openid profile email offline_access accounting.contacts accounting.invoices
```
(`accounting.invoices` already covers reading invoices, so you don't also add
`accounting.invoices.read`.)

Store the chosen string in the `XERO_SCOPES` secret (space-separated). **Make the
`XERO_SCOPES` fallback in `_shared/xero.ts` match your intent** — the all-`.read`
default shown there will silently hand you a read-only token if the secret is missing
or the function wasn't redeployed after you set it, and your first write will 403.

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

> **Gate this function — and don't skip it just because you have no auth yet.**
> Anyone who can reach `xero-start` can run a connect and, via the callback,
> **overwrite your single token row with a different Xero org's tokens**, pointing
> your whole app at someone else's books. So `xero-start` must never be openly
> reachable. Pick the gate that fits where you are:
>
> - **You have user auth already** (an admin login): keep `verify_jwt = true`, call
>   `getUser`, check the caller is an admin. That's the version coded below.
> - **No auth implemented yet** (common for a single-merchant store or early
>   prototype): you can't check a JWT — but do **not** just deploy it public and move
>   on, that's the gap. Use a **shared-secret interim** (snippet after the code):
>   require a secret header only you know, set `verify_jwt = false`, and trigger the
>   one-time connect out-of-band (curl) from your own machine. Don't put the secret
>   in browser code.
> - **Before you go live, either way:** replace the interim with real admin auth, or
>   — since connecting is one-time — disable or delete `xero-start` once connected,
>   so there's nothing left to abuse.
>
> `xero-callback` stays `verify_jwt = false` regardless (Xero's browser redirect has
> no JWT); its protection is the `state` row that only a successful `xero-start`
> could have written — and that only holds if `state` is **expiry-checked and
> single-use** (deleted after the exchange, so a captured callback URL can't be
> replayed). The callback code below does both. So what makes a public callback safe
> is locking down `xero-start` *plus* disciplined `state` handling — not either alone.

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

**Shared-secret interim (when you have no user auth yet).** Set `verify_jwt = false`
for `xero-start`, add an `XERO_CONNECT_SECRET` Supabase secret, and gate on a header
instead of a JWT. Two non-negotiables, because `verify_jwt = false` leaves this
endpoint **publicly reachable** — a leaked header is full compromise (anyone can
re-point your app at their own books):

- **Generate it, don't choose it.** Use a high-entropy value, e.g.
  `openssl rand -base64 32` — never a hand-picked or guessable string. Treat it like
  a password: never commit it, never put it in client code, rotate it if exposed.
- **Compare it timing-safe**, not with `===` on the raw string (a plain compare
  leaks the secret to a timing attacker one byte at a time).

Swap the `getUser()` check at the top of the handler for a constant-time compare —
hash both sides and compare fixed-length digests, which stays in Web Crypto (no Node
dependency) and doesn't leak length:

```ts
// xero-start interim gate — replaces the getUser() check above
async function safeEqual(a: string, b: string) {
  const enc = new TextEncoder();
  const [ha, hb] = await Promise.all([
    crypto.subtle.digest("SHA-256", enc.encode(a)),
    crypto.subtle.digest("SHA-256", enc.encode(b)),
  ]);
  const x = new Uint8Array(ha), y = new Uint8Array(hb);
  let diff = 0;
  for (let i = 0; i < x.length; i++) diff |= x[i] ^ y[i];   // constant-time over the digest
  return diff === 0;
}
const provided = req.headers.get("x-connect-secret") ?? "";
if (!(await safeEqual(provided, Deno.env.get("XERO_CONNECT_SECRET") ?? ""))) {
  return new Response("Forbidden", { status: 403 });
}
```

**Adopting this interim removes the in-app connect button — the two are mutually
exclusive.** A secret-gated `xero-start` *cannot* be triggered from the browser
(that's the whole point: the secret must never ship in client code), so on the
interim there is **no in-app "Connect Xero" button**. You connect once, by hand,
from your own machine:

```
curl -i -H "x-connect-secret: <your-secret>" \
  https://<project-ref>.supabase.co/functions/v1/xero-start
```
It returns the consent URL; open that in a normal browser tab to authorise. The
popup/iframe button pattern below applies **only** after you've moved to the
admin-JWT gate. Replace the interim with real admin auth — or delete `xero-start` —
before launch.

**The button pattern below assumes the admin-JWT gate** (`verify_jwt = true`), so the
browser carries the signed-in admin's session when it calls `xero-start`. If you're
on the shared-secret interim, skip this — there is no in-app button; connect with the
curl above, and come back here once you have real auth.

**Lovable preview iframe gotcha.** Lovable's in-editor preview renders your app in
an iframe, and Xero's login refuses to be framed (`frame-ancestors`), so a
same-frame redirect shows a **blank white page**. Detect the iframe and open consent
in a new top-level tab, opening the tab **synchronously in the click handler**
(before the `await`) so it isn't blocked as a popup:

```ts
const inIframe = window.self !== window.top;
const popup = inIframe ? window.open("about:blank", "_blank") : null;
const { data } = await supabase.functions.invoke("xero-start");   // carries the admin's session JWT
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
  if (!flow || new Date(flow.expires_at) < new Date()) return back("?error=bad_state");  // must exist AND be unexpired

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
  await admin.from("xero_oauth_flows").delete().eq("state", state);  // single-use: consume state so a captured callback can't be replayed
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

## 6. Surface connection status (reconnect UX)

A real dashboard has to *show* when the connection has dropped. A failed refresh
flips `status` to `needs_reconnect`, and from then on every Xero call throws until
you reconnect — don't leave that invisible behind a generic error. Expose a tiny
**status** endpoint that returns only non-sensitive metadata (never the token) and
drive a banner from it.

```ts
// supabase/functions/xero-status/index.ts   (gate as your app needs)
import { admin } from "../_shared/admin.ts";
Deno.serve(async () => {
  const { data } = await admin.from("xero_connection")
    .select("tenant_name, status").limit(1).maybeSingle();
  return new Response(JSON.stringify(
    data
      ? { connected: data.status === "active", status: data.status, org: data.tenant_name }
      : { connected: false, status: "not_connected", org: null },
  ), { headers: { "Content-Type": "application/json" } });
});
```

Frontend: poll it (or refetch on focus) and show a reconnect CTA when needed — the
CTA runs the very same connect flow as the first-time hookup.

```tsx
const { data: xero } = useQuery({
  queryKey: ["xero-status"],
  queryFn: async () => (await supabase.functions.invoke("xero-status")).data,
  refetchOnWindowFocus: true,
});

if (xero && !xero.connected) {
  return (
    <Banner tone="warning">
      {xero.status === "needs_reconnect"
        ? `Xero connection for ${xero.org} has expired — reconnect to keep data flowing.`
        : "Connect your Xero organisation to get started."}
      <button onClick={startXeroConnect}>
        {xero.status === "needs_reconnect" ? "Reconnect Xero" : "Connect Xero"}
      </button>
    </Banner>
  );
}
```

The endpoint returns the org name and status, so gate it if even that is sensitive
in your app. And mark `needs_reconnect` *proactively*, not only after a failed
refresh: a refresh token unused for 60 days dies silently, so a low-traffic dashboard
can read as "connected" in the table yet be broken. A scheduled keep-alive refresh
(see gotchas) plus this banner keeps the displayed state honest.

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
- **Redeploy Edge Functions after changing any secret.** They don't hot-reload env;
  `client_id=undefined` in the authorize URL is the tell. (N/A on non-Edge runtimes.)
- **`xero-start` must be gated** — admin JWT, or a shared-secret header as a
  documented interim if you have no auth yet (never leave it openly reachable; §3).
  Unguarded, it lets anyone overwrite your token row with a different org's books.
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
- **`xero-callback` is public (`verify_jwt = false`); `xero-start` is gated** — admin
  JWT, or a shared-secret header as a documented interim when you have no auth yet,
  and removed or locked to real auth before launch.
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
5. Confirm `xero-start` rejects an ungated caller (no admin JWT, or no/incorrect
   `x-connect-secret` on the interim).
6. If your app writes to Xero, confirm a `POST` (e.g. create invoice) succeeds rather
   than returning 403 — proof the token carries write scopes, not just `.read`.
7. Change a secret, redeploy, and confirm the authorize URL no longer shows
   `client_id=undefined` (Edge Functions only).
8. Flip a row to `needs_reconnect` and confirm the banner appears and its CTA re-runs
   the connect flow.
