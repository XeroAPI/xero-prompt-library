---
name: xero-integration-multi-account
description: Use when building a MULTI-TENANT Xero integration in Lovable where each end user connects THEIR OWN Xero organisation, and the app reads/writes Xero as that user with their own access + refresh tokens. Covers the full OAuth2 lifecycle in TanStack Start server functions, per-user encrypted token storage with RLS, refresh/rotation, tenant selection, scopes, and Xero API gotchas (granular scope migration, date/money/tax normalisation, idempotent writes, rate limits). Not for single-org apps where you (the builder) connect one Xero you own — that needs only one stored token set, not per-user keying.
---

# Multi-account / multi-tenant Xero (each user connects their own Xero)

This skill is for a **multi-tenant SaaS built in Lovable**: every customer links
**their own** Xero organisation, and your app reads/writes Xero **as that
customer**, with **their** tokens.

> **First, make sure this is the right variant.** Lovable has no managed
> single-principal Xero connector, so you own the OAuth lifecycle either way.
> The deciding question is *how many Xero logins exist*:
>
> - **One — yours.** Internal dashboard, single-merchant store, back-office tool,
>   prototype. Stop and use **`xero-integration-single-account`** instead — one
>   stored token set, no per-user keying.
> - **Many — one per customer.** Each end user links *their own* Xero, and the
>   app acts as that customer. **This skill.** Per-user token rows, tenant
>   selection, multi-tenant refresh rotation.
>
> Don't confuse Xero **multi-org** with **multi-tenant**. `GET /connections`
> returns several *organisations* under ONE login — that's still one principal,
> still single-account. Multi-tenant = many *separate* Xero logins, one per
> customer.

## Where this code runs in Lovable

Lovable's current default stack is **TanStack Start** on a Cloudflare Workers-style
serverless runtime, with **Supabase** for Postgres and **Lovable Cloud** for
deployment and secrets. App-internal server logic — including all Xero OAuth and
token handling — lives in **`createServerFn` server functions**; external callers
(Xero's browser redirect, webhooks, cron) hit **server routes** under
`src/routes/api/public/*`. **Do not use Supabase Edge Functions** for any of this.

Two runtime facts shape the code below:

- The server runtime is stateless across invocations, so the OAuth `state` value
  must be persisted to a table between the connect step and the callback.
- Use `fetch` for the OAuth lifecycle (authorize redirect, token exchange, refresh)
  — three calls, no SDK needed. You *may* use `npm:xero-node` for read/write API
  calls if it bundles cleanly, but keep the OAuth flow `fetch`-based.

**Crypto is Node `crypto`** (the Worker runtime supports it via `nodejs_compat`);
the three-part `iv.tag.enc` ciphertext shape below is correct on this stack.

> **If you're working in a legacy Lovable project that still uses Supabase Edge
> Functions (Deno) as its server layer**, port these handlers to Edge Functions:
> start/callback/getToken become Edge Functions, secrets are read via
> `Deno.env.get` (and you must redeploy after changing one), the publicly-reachable
> callback needs `verify_jwt = false`, and crypto becomes Web Crypto with a
> two-part `iv.ciphertext` blob (Web Crypto appends the GCM tag to the ciphertext).
> Everything else — SQL, scopes, idempotency, Xero gotchas — carries over
> unchanged.

## Decision check before building

Build the multi-tenant version only if ALL are true:
- Each customer connects their **own** Xero org (not yours).
- The app acts under **each customer's** Xero identity, not a shared one.
- You can securely store and rotate per-user refresh tokens (Supabase + RLS, below).

If any is false, use `xero-integration-single-account` (one stored token set).

## 1. Register your own Xero app

In the Xero developer portal (developer.xero.com) create a **Web app** OAuth2 app
(confidential — it holds a secret, which your server functions can):
- Grab **Client ID** and **Client Secret**.
- Register your redirect URI as your **callback server-route URL**, exactly:
  `https://<your-app>.lovable.app/api/public/xero/callback`
  Register your preview URL too (e.g. `https://<preview-host>.lovable.app/...`) so
  you can test consent end-to-end before publishing. **Tell the user explicitly
  what redirect URIs to register for both their deployed and preview environments.**
- Decide your scopes (see **Scopes** next). You MUST include **`offline_access`**
  or Xero returns no refresh token.

Store **your app's** Client ID/Secret as Lovable Cloud secrets (`XERO_CLIENT_ID`,
`XERO_CLIENT_SECRET`, `XERO_REDIRECT_URI`, `XERO_SCOPES`), plus a base64-encoded
32-byte **`TOKEN_ENC_KEY`** for encrypting stored tokens. `SUPABASE_URL` and
`SUPABASE_SERVICE_ROLE_KEY` are already available to server functions via
`process.env`. **Never** put per-user tokens in secrets — they go in the database,
encrypted. The refresh token **rotates on every refresh**, so it must live in a
**mutable** store (the database).

> Read `process.env.*` **inside** `.handler()`, not at module scope. The Worker
> runtime injects env per-request; a module-level read returns `undefined`.

## Scopes

Scopes are **additive** — request the minimum and re-run consent to add more later.
You **cannot** widen scope on an existing token; the user must re-consent. To get
a refresh token at all you must include `offline_access`.

(This table is identical to the single-account skill — keep the two in sync if
Xero's assignments change.)

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
scope for each. Shipping a read-only token and then calling `POST /Invoices` fails
at runtime with a 403 / insufficient-scope error — and because scopes **cannot be
widened on an existing token**, the only fix is to send every connected customer
back through the whole consent flow. So decide up front: read, write, or both.

A typical invoicing SaaS therefore wants:
```
openid profile email offline_access accounting.contacts accounting.invoices
```

Store this string in the `XERO_SCOPES` secret (space-separated). **Make the
`XERO_SCOPES` fallback in your shared Xero config match your intent** — an
all-`.read` default will silently hand every customer a read-only token if the
secret is missing, and your first write will 403.

Source of truth (re-check periodically — dates and assignments change):
https://developer.xero.com/documentation/guides/oauth2/scopes

## 2. Per-user token store (encrypted, RLS-locked)

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
-- Only the service role (used inside server functions / server routes) may touch this.
-- Do NOT grant anon/authenticated — the browser must never read token_set_enc.
grant all on public.xero_connections to service_role;
alter table public.xero_connections enable row level security;
-- INTENTIONALLY no anon/authenticated policies. NEVER add a policy that lets the
-- logged-in user SELECT token_set_enc — tokens must never reach the browser.

-- xero_oauth_flows: short-lived CSRF/state handshake between start and callback
create table public.xero_oauth_flows (
  state      text primary key,
  user_id    uuid not null references auth.users(id) on delete cascade,
  created_at timestamptz not null default now(),
  expires_at timestamptz not null
);
grant all on public.xero_oauth_flows to service_role;
alter table public.xero_oauth_flows enable row level security;  -- service-role only
```

AES-256-GCM helpers using **Node `crypto`**. The blob is three parts
(`iv.tag.ciphertext`, base64) — this is correct for `node:crypto` GCM, where the
auth tag is exposed separately. Put helpers in a `.server.ts` file so they're
blocked from the client bundle.

```ts
// src/lib/server/crypto.server.ts
import { createCipheriv, createDecipheriv, randomBytes } from "node:crypto";

function key() {
  return Buffer.from(process.env.TOKEN_ENC_KEY!, "base64"); // 32 bytes
}

export function encrypt(plain: string) {
  const iv = randomBytes(12);
  const c = createCipheriv("aes-256-gcm", key(), iv);
  const ct = Buffer.concat([c.update(plain, "utf8"), c.final()]);
  const tag = c.getAuthTag();
  return `${iv.toString("base64")}.${tag.toString("base64")}.${ct.toString("base64")}`;
}

export function decrypt(blob: string) {
  const [iv, tag, ct] = blob.split(".").map((s) => Buffer.from(s, "base64"));
  const d = createDecipheriv("aes-256-gcm", key(), iv);
  d.setAuthTag(tag);
  return Buffer.concat([d.update(ct), d.final()]).toString("utf8");
}
```

Shared Xero config (client-safe constants only — secrets are read inside handlers):

```ts
// src/lib/xero-config.ts
export const XERO = {
  authorize:   "https://login.xero.com/identity/connect/authorize",
  token:       "https://identity.xero.com/connect/token",
  connections: "https://api.xero.com/connections",
  api:         "https://api.xero.com/api.xro/2.0",
};
export const basicAuth = (id: string, secret: string) =>
  Buffer.from(`${id}:${secret}`).toString("base64");
export const randomState = () =>
  Buffer.from(crypto.getRandomValues(new Uint8Array(16)))
    .toString("base64")
    .replace(/\+/g, "-").replace(/\//g, "_").replace(/=+$/, "");
```

## 3. Start consent (server function, knows the user)

`state` is your CSRF defense and your link back to the initiating user. Persist
it server-side (the `xero_oauth_flows` row) — never trust it blind on return.
Use `requireSupabaseAuth` so the handler always has the calling user's id; this
is what scopes every stored token row to the right customer.

```ts
// src/lib/xero-start.functions.ts
import { createServerFn } from "@tanstack/react-start";
import { requireSupabaseAuth } from "@/integrations/supabase/auth-middleware";
import { XERO, randomState } from "./xero-config";

export const xeroStart = createServerFn({ method: "POST" })
  .middleware([requireSupabaseAuth])
  .handler(async ({ context }) => {
    const { supabaseAdmin } = await import("@/integrations/supabase/client.server");

    const state = randomState();
    await supabaseAdmin.from("xero_oauth_flows").insert({
      state,
      user_id: context.userId,
      expires_at: new Date(Date.now() + 10 * 60_000).toISOString(),
    });

    const url = new URL(XERO.authorize);
    url.searchParams.set("response_type", "code");
    url.searchParams.set("client_id", process.env.XERO_CLIENT_ID!);
    url.searchParams.set("redirect_uri", process.env.XERO_REDIRECT_URI!);
    url.searchParams.set("scope", process.env.XERO_SCOPES!);
    url.searchParams.set("state", state);
    return { url: url.toString() };
  });
```

> **Why no PKCE here.** Your server functions can hold the client secret, so the
> standard confidential web-app flow is correct: the **secret** authenticates the
> token exchange and `state` covers CSRF. PKCE exists for public clients that
> *can't* keep a secret (mobile/SPA); bolting it onto a confidential server flow
> adds a `code_verifier` you'd have to persist across the two stateless
> invocations for no security gain. Don't use PKCE for this — secret + `state`.

**Lovable preview iframe gotcha.** Lovable's in-editor preview renders your app in
an iframe, and Xero's login page refuses to be framed (`frame-ancestors`), so a
same-frame redirect shows a **blank white page**. Detect the iframe and open
consent in a new top-level tab, opening the tab **synchronously in the click
handler** (before the `await`) so it isn't blocked as a popup:

```tsx
import { useServerFn } from "@tanstack/react-start";
import { xeroStart } from "@/lib/xero-start.functions";

const start = useServerFn(xeroStart);
async function onConnect() {
  const inIframe = window.self !== window.top;
  const popup = inIframe ? window.open("about:blank", "_blank") : null;
  const { url } = await start();
  if (popup) popup.location.href = url;
  else window.location.href = url;
}
```

The published app at `<your-app>.lovable.app` is not iframed, so the same-tab
redirect works there — handle both. After OAuth, let TanStack Query's
refetch-on-focus refresh the connection list in the original tab.

## 4. Callback — exchange, pick tenant(s), store

Xero redirects the **browser** here with no app session. The handler lives at a
server route under `src/routes/api/public/` so it bypasses Lovable's published-site
auth gate; trust comes from the `state` row written by `xeroStart`, which carries
the originating `user_id`.

```ts
// src/routes/api/public/xero.callback.ts
import { createFileRoute } from "@tanstack/react-router";
import { XERO, basicAuth } from "@/lib/xero-config";

export const Route = createFileRoute("/api/public/xero/callback")({
  server: {
    handlers: {
      GET: async ({ request }) => {
        const { supabaseAdmin } = await import("@/integrations/supabase/client.server");
        const { encrypt } = await import("@/lib/server/crypto.server");

        const APP_URL = process.env.APP_URL!;
        const back = (q: string) =>
          new Response(null, { status: 302, headers: { Location: `${APP_URL}/xero${q}` } });

        const u = new URL(request.url);
        const code = u.searchParams.get("code");
        const state = u.searchParams.get("state");
        if (!code || !state) return back("?error=missing_params");

        const { data: flow } = await supabaseAdmin
          .from("xero_oauth_flows").select("*").eq("state", state).single();
        if (!flow || new Date(flow.expires_at) < new Date()) return back("?error=bad_state");

        // confidential flow: Basic auth with the secret
        const tokenRes = await fetch(XERO.token, {
          method: "POST",
          headers: {
            Authorization: `Basic ${basicAuth(process.env.XERO_CLIENT_ID!, process.env.XERO_CLIENT_SECRET!)}`,
            "Content-Type": "application/x-www-form-urlencoded",
          },
          body: new URLSearchParams({
            grant_type: "authorization_code",
            code,
            redirect_uri: process.env.XERO_REDIRECT_URI!,
          }),
        });
        if (!tokenRes.ok) return back("?error=token_exchange");
        const tokenSet = await tokenRes.json();

        // which orgs did this single consent authorize?
        const conns = await fetch(XERO.connections, {
          headers: { Authorization: `Bearer ${tokenSet.access_token}` },
        }).then((r) => r.json());

        // A single consent can authorize MULTIPLE orgs that all SHARE ONE token set.
        // Don't assume conns[0]. Encrypt ONCE and write the IDENTICAL ciphertext to every
        // row — this lets refresh (§5) find and update all sibling rows together.
        const enc = encrypt(JSON.stringify(tokenSet));
        const expiresAt = new Date(Date.now() + tokenSet.expires_in * 1000).toISOString();
        for (const c of conns) {
          await supabaseAdmin.from("xero_connections").upsert(
            { user_id: flow.user_id, tenant_id: c.tenantId, tenant_name: c.tenantName,
              token_set_enc: enc, status: "active", expires_at: expiresAt },
            { onConflict: "user_id,tenant_id" },
          );
        }
        // single-use: consume state so a captured callback URL can't be replayed
        await supabaseAdmin.from("xero_oauth_flows").delete().eq("state", state);
        return back("?connected=1");
      },
    },
  },
});
```

## 5. Use it per request (refresh + persist rotation)

The server runtime instantiates fresh each call, so "never cache a client" is
automatic. Refresh slightly early, rotate the refresh token, and update **all
sibling rows** that shared the old token set. Put this in a `.server.ts` helper
and call it from inside authenticated server-function handlers.

```ts
// src/lib/server/xero-token.server.ts
import { XERO, basicAuth } from "@/lib/xero-config";
import { encrypt, decrypt } from "./crypto.server";

export async function getAccessToken(userId: string, tenantId: string) {
  const { supabaseAdmin } = await import("@/integrations/supabase/client.server");

  const { data: conn } = await supabaseAdmin.from("xero_connections")
    .select("*").eq("user_id", userId).eq("tenant_id", tenantId).single();
  if (!conn) throw new Error("User has not connected this Xero org");

  let tokenSet = JSON.parse(decrypt(conn.token_set_enc));

  if (new Date(conn.expires_at) <= new Date(Date.now() + 60_000)) {
    const res = await fetch(XERO.token, {
      method: "POST",
      headers: {
        Authorization: `Basic ${basicAuth(process.env.XERO_CLIENT_ID!, process.env.XERO_CLIENT_SECRET!)}`,
        "Content-Type": "application/x-www-form-urlencoded",
      },
      body: new URLSearchParams({
        grant_type: "refresh_token",
        refresh_token: tokenSet.refresh_token,
      }),
    });
    if (!res.ok) {
      // refresh token is dead → surface a reconnect state, don't loop forever
      await supabaseAdmin.from("xero_connections")
        .update({ status: "needs_reconnect" })
        .eq("user_id", userId).eq("token_set_enc", conn.token_set_enc);
      throw new Error("Xero refresh failed — prompt reconnect");
    }
    tokenSet = await res.json();                         // refresh_token ROTATES — must persist
    const enc = encrypt(JSON.stringify(tokenSet));
    const expiresAt = new Date(Date.now() + tokenSet.expires_in * 1000).toISOString();
    // CRITICAL multi-tenant trap: orgs that shared ONE token set all hold the SAME
    // (now-rotated, invalidated) refresh token. Update EVERY sibling row, matched on
    // the OLD ciphertext, or the others flip to needs_reconnect on their next call.
    await supabaseAdmin.from("xero_connections")
      .update({ token_set_enc: enc, expires_at: expiresAt, updated_at: new Date().toISOString() })
      .eq("user_id", userId).eq("token_set_enc", conn.token_set_enc);
  }
  return tokenSet.access_token as string;
}
```

Example write inside an authenticated server function — note the explicit
`Xero-Tenant-Id` header (there is no implicit "current org"), `summarizeErrors=false`,
`unitdp=4`, and an idempotency key:

```ts
// src/lib/xero-invoices.functions.ts
import { createServerFn } from "@tanstack/react-start";
import { requireSupabaseAuth } from "@/integrations/supabase/auth-middleware";
import { z } from "zod";
import { XERO } from "./xero-config";

export const createInvoice = createServerFn({ method: "POST" })
  .middleware([requireSupabaseAuth])
  .inputValidator((d) => z.object({ tenantId: z.string(), invoice: z.unknown() }).parse(d))
  .handler(async ({ data, context }) => {
    const { getAccessToken } = await import("./server/xero-token.server");
    const accessToken = await getAccessToken(context.userId, data.tenantId);

    const res = await fetch(`${XERO.api}/Invoices?unitdp=4&summarizeErrors=false`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${accessToken}`,
        "Xero-Tenant-Id": data.tenantId,                       // REQUIRED — wrong id writes to wrong books
        "Idempotency-Key": crypto.randomUUID(),
        "Content-Type": "application/json",
        Accept: "application/json",
      },
      body: JSON.stringify({ Invoices: [data.invoice] }),
    });
    if (!res.ok) throw new Error(`Xero invoice failed: ${res.status} ${await res.text()}`);
    return res.json();
  });
```

## 6. Surface connection status (reconnect UX)

A real dashboard has to *show* when a customer's connection has dropped. A failed
refresh flips `status` to `needs_reconnect`, and from then on every Xero call for
that customer throws until they reconnect — don't leave that invisible behind a
generic error. Expose a tiny **status** server function that returns only
non-sensitive metadata (never the token) and drive a banner from it.

```ts
// src/lib/xero-status.functions.ts
import { createServerFn } from "@tanstack/react-start";
import { requireSupabaseAuth } from "@/integrations/supabase/auth-middleware";

export const xeroConnections = createServerFn({ method: "GET" })
  .middleware([requireSupabaseAuth])
  .handler(async ({ context }) => {
    const { supabaseAdmin } = await import("@/integrations/supabase/client.server");
    const { data } = await supabaseAdmin.from("xero_connections")
      .select("tenant_id, tenant_name, status")
      .eq("user_id", context.userId);
    return data ?? [];
  });
```

Frontend: read it (or refetch on focus) and show a reconnect CTA per row when
needed — the CTA runs the very same connect flow as the first-time hookup.

Mark `needs_reconnect` *proactively*, not only after a failed refresh: a refresh
token unused for 60 days dies silently, so a low-traffic customer can read as
"connected" yet be broken. A scheduled keep-alive refresh (see gotchas) plus this
banner keeps the displayed state honest.

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

- **Money.** Send numbers, not formatted currency strings. Control rounding with
  the `unitdp` query param (up to 4). Postgres `numeric` comes back as strings via
  Supabase — convert to numbers before any client-side parse. `AmountDue` is a
  number.

- **Tax is the org's, not yours.** Each connected organisation applies **its own**
  tax config, so `AmountDue` / `Total` may exceed the sum of your line amounts
  (e.g. 15% GST turns a 2495 line into 2869.25). The `UnitAmount` you posted is
  still correct — don't "fix" the discrepancy. Set `LineAmountTypes` (`Exclusive`,
  `Inclusive`, `NoTax`) deliberately if you need control. **In multi-tenant this
  varies per customer** — never hard-code a tax assumption; whatever each org
  returns is correct for that org.

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
  on retry; to recover, GET to check whether the resource was actually created,
  and if not, retry with a **new** key.

When a user *re-submits* a previously **failed** order later, mint a **new** key —
but first GET to confirm no invoice was actually created, so you don't duplicate
one that silently succeeded.

## Other Xero gotchas (these bite in production)

- **`offline_access` is mandatory for refresh.** Omit it and the access token dies
  in ~30 min with no way to refresh — the user must re-consent.
- **Token lifetimes:** access token ~30 min; refresh token ~60 days and **rotates
  on every refresh**. Persist the new refresh token each time or the next refresh
  fails. A refresh token also dies if unused for 60 days — low-traffic customers
  hit this; consider a scheduled keep-alive refresh per connection.
- **Always send `Xero-Tenant-Id`** on every API call. There is no implicit "current
  org." With multiple connected orgs, the wrong id silently writes to the wrong
  company's books.
- **Re-fetch `/connections` periodically**, not just at callback — the connected
  org list can change as users add/remove orgs.
- **Contacts aren't deduplicated by email.** Posting an invoice with a new contact
  name creates a NEW contact each time. Look up by email/`ContactID` via
  `GET /Contacts` and reuse the id; otherwise you litter the org with duplicates.
- **Invoice status flow:** `DRAFT` → `SUBMITTED` → `AUTHORISED`; only `AUTHORISED`
  are real receivables and expose an online payment URL. Use `ACCREC` (sales) vs
  `ACCPAY` (bills) correctly.
- **Rate limits (per tenant):** ~60 calls/minute, 5,000/day, plus an app-wide
  per-minute cap and ~5 concurrent requests. On **429** read `Retry-After` and
  back off — multi-tenant apps hit this fast when looping over orgs. Spread/queue
  calls per tenant.
- **`summarizeErrors=false`** on batch creates returns per-item validation errors.

### Lovable / TanStack Start gotchas

- **The callback MUST live under `src/routes/api/public/`.** That prefix bypasses
  Lovable's published-site auth gate; anywhere else, Xero's redirect lands on a
  login wall and the connect silently fails.
- **`xeroStart` must be gated by `requireSupabaseAuth`.** It's the middleware that
  binds the OAuth flow to the calling user; an unauthenticated `xeroStart` would
  store tokens against no one (or worse, the wrong customer).
- **The `state` value MUST be persisted.** The server runtime is stateless —
  start and callback are separate invocations with no shared memory.
- **Never import `@/integrations/supabase/client.server` at module scope of a
  `*.functions.ts` or route file.** Top-level code in those files ships to the
  client bundle (only handler bodies are stripped). Load it inside the handler.
- **`process.env.*` is server-only and must be read inside the handler**, not at
  module scope. A module-level read returns `undefined` under the Worker runtime.
- **Service-role key never goes to the browser.** Surface only derived data
  (connection metadata, invoices) through your own server functions; the token
  tables have RLS with no anon/authenticated policies and no grants to those roles.
- **`npm:xero-node`** for read/write API calls is optional and must bundle cleanly
  for the Worker runtime — the `fetch`-based OAuth flow above avoids the dependency.

## Security checklist (do not skip)

- **Encrypt** the stored token set at rest; the key (`TOKEN_ENC_KEY`) lives in
  Lovable Cloud secrets, the rotating tokens in the DB.
- **Verify `state`** on every callback against the persisted flow row (CSRF), and
  **delete it after use** so a captured callback URL can't be replayed.
- **`xeroStart` and every other Xero server function require the user's session**
  via `requireSupabaseAuth`. Only the callback server route is public, and it only
  acts on a valid, unexpired, single-use `state`.
- **RLS on token tables with no user policies** — only the service role touches
  them. No `GRANT` to `anon` or `authenticated`.
- **Never expose tokens or the service-role key to the browser** or any
  client-readable storage.
- On a failed refresh, **mark the connection `needs_reconnect` and prompt
  re-connect** — don't silently retry forever.
- Keep all Xero access in **one place** (`xero-config.ts` + `xero-token.server.ts`)
  so paths, tenant handling, and gotchas live together.

## Don't want to hand-roll it?

Managed token-vault platforms (**Nango**, **Composio**, etc.) implement per-user
OAuth + refresh + encryption for Xero and many other providers, and you call out
to them from a server function. For a multi-provider SaaS this is often the
pragmatic choice.

## Verify

1. Two app users each connect a **different** Xero org; confirm two distinct token
   rows and that each user's writes land in their own org (check `Xero-Tenant-Id`).
2. Force a near-expiry token and confirm transparent refresh **and** that the
   rotated refresh token is persisted to every sibling row.
3. Revoke the app in one user's Xero and confirm your app surfaces a reconnect
   state (`needs_reconnect`) instead of looping on failed refreshes.
4. Post the same invoice twice with the same `Idempotency-Key` and confirm only
   one invoice is created.
5. Hit the callback route directly in a browser and confirm it isn't blocked by a
   login wall (i.e. it actually lives under `/api/public/`).
6. Confirm `xeroStart` rejects an unauthenticated caller (401 from middleware).
7. If your app writes to Xero, confirm a `POST` (e.g. create invoice) succeeds
   rather than 403 — proof tokens carry write scopes, not just `.read`.
8. Flip a connection to `needs_reconnect` and confirm the per-row banner appears
   and its CTA re-runs the connect flow.
