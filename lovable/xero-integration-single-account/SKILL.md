---
name: xero-integration-single-account
description: Use when YOU (the app builder) connect ONE Xero organisation that you own, and the whole app reads/writes that single set of books — internal dashboards, a single-merchant storefront, a back-office tool, or a prototype. Covers the OAuth2 lifecycle in TanStack Start server functions, one encrypted token set with RLS, refresh/rotation, scopes, and Xero API gotchas (granular scope migration, date/money/tax normalisation, idempotent writes, rate limits). Not for apps where each end user connects their OWN Xero org — use xero-integration-multi-account for that.
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
> connect/callback/status become three Edge Functions, secrets are read via
> `Deno.env.get` (and you must redeploy after changing one), the publicly-reachable
> callback needs `verify_jwt = false`, and crypto becomes Web Crypto with a
> two-part `iv.ciphertext` blob (Web Crypto appends the GCM tag to the ciphertext).
> Everything else — SQL, scopes, idempotency, Xero gotchas — carries over
> unchanged.

## 1. Register your own Xero app

In the Xero developer portal (developer.xero.com) create a **Web app** OAuth2 app
(confidential — it holds a secret, which your server functions can):
- Grab **Client ID** and **Client Secret**.
- Register your redirect URI as your **callback server-route URL**, exactly:
  `https://<your-app>.lovable.app/api/public/xero/callback`
  Register **both** redirect URIs in Xero before first connect — the deployed
  app and the preview host (`https://id-preview--<project-id>.lovable.app/api/public/xero/callback`)
  — so you can test consent end-to-end before publishing. **Tell the user
  explicitly what redirect URIs to register for both environments.** The
  in-editor preview iframe host (`*.lovableproject.com`) is **not** a valid
  redirect URI — don't register it and don't send it (see the iframe gotcha
  in §3 for how to derive `redirect_uri` correctly).

- Decide your scopes (see **Scopes** next). You MUST include **`offline_access`**
  or Xero returns no refresh token.

Store **your app's** Client ID/Secret as Lovable Cloud secrets (`XERO_CLIENT_ID`,
`XERO_CLIENT_SECRET`, `XERO_REDIRECT_URI`, `XERO_SCOPES`), plus a base64-encoded
32-byte **`TOKEN_ENC_KEY`** for encrypting the stored token. `SUPABASE_URL` and
`SUPABASE_SERVICE_ROLE_KEY` are already available to server functions via
`process.env`. The refresh token **rotates on every refresh**, so it must live in
a **mutable** store (the database) — never in a secret.

> Read `process.env.*` **inside** `.handler()`, not at module scope. The Worker
> runtime injects env per-request; a module-level read returns `undefined`.

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
`XERO_SCOPES` fallback in your shared Xero config match your intent** — an
all-`.read` default will silently hand you a read-only token if the secret is
missing, and your first write will 403.

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
-- Only the service role (used inside server functions / server routes) may touch this.
-- Do NOT grant anon/authenticated — the browser must never read token_set_enc.
grant all on public.xero_connection to service_role;
alter table public.xero_connection enable row level security;
-- INTENTIONALLY no anon/authenticated policies.

-- xero_oauth_flows: short-lived CSRF/state handshake between connect and callback
create table public.xero_oauth_flows (
  state      text primary key,
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

## 3. Connect (one-time) — start consent

Connecting your org is normally a **one-time admin action**. Use the standard
confidential web-app flow: the client **secret** authenticates the token exchange
and `state` covers CSRF — no PKCE (that's for public clients that can't keep a
secret).

> ### ⚠️ Lock this behind admin auth before you connect
>
> Anyone who can reach the `xeroStart` server function can complete a connect
> and, via the callback, **overwrite your single token row with a different Xero
> org's tokens**, pointing your whole app at someone else's books. So before you
> wire up the button:
>
> 1. **Set up Supabase Auth** in your project (Lovable's built-in integration).
> 2. **Add a `user_roles` table and `has_role()` function** (see the user-roles
>    convention) so you can mark yourself as `admin`.
> 3. Gate `xeroStart` with `.middleware([requireSupabaseAuth])` AND a
>    `has_role(userId, 'admin')` check inside the handler.
>
> If your project has no auth yet, **add auth first** — don't ship an ungated
> connect endpoint. There is no curl/shared-secret workaround in this skill on
> purpose: non-developers shouldn't have to paste commands into a terminal, and
> an unguarded `xeroStart` is a full-takeover bug waiting to happen.

```ts
// src/lib/xero-start.functions.ts
import { createServerFn } from "@tanstack/react-start";
import { requireSupabaseAuth } from "@/integrations/supabase/auth-middleware";
import { XERO, randomState } from "./xero-config";

export const xeroStart = createServerFn({ method: "POST" })
  .middleware([requireSupabaseAuth])
  .handler(async ({ context }) => {
    const { supabaseAdmin } = await import("@/integrations/supabase/client.server");

    // Admin-only — anyone who can hit this can repoint your books.
    const { data: isAdmin } = await context.supabase
      .rpc("has_role", { _user_id: context.userId, _role: "admin" });
    if (!isAdmin) throw new Error("Forbidden");

    const state = randomState();
    await supabaseAdmin.from("xero_oauth_flows").insert({
      state,
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

**Lovable preview iframe gotcha.** Lovable's in-editor preview renders your app
in an iframe, and Xero's login refuses to be framed (`frame-ancestors`), so a
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
redirect works there — handle both.

**Don't derive `redirect_uri` from `window.location.origin`.** The editor
renders your app inside an iframe whose origin is `*.lovableproject.com` —
**not** the registered `id-preview--*.lovable.app` host. Sending the iframe
origin to Xero produces `invalid redirect_uri` every time, because
`*.lovableproject.com` is not (and must not be) registered. Two correct
options, in order of preference:

1. **Read `XERO_REDIRECT_URI` server-side** (preferred). Store one secret per
   environment and use it in both `xeroStart` and the callback. No
   client-side guessing, and it matches what's registered in Xero exactly.
2. **If you must compute it on the client**, map the iframe host back to the
   public preview host before sending to the server — e.g. detect
   `*.lovableproject.com` and translate to the matching
   `id-preview--<project-id>.lovable.app`. Pass that derived origin into
   `xeroStart`; never pass `window.location.origin` raw.



## 4. Callback — exchange, store your org

Xero redirects the **browser** here with no app session. The handler lives at a
server route under `src/routes/api/public/` so it bypasses Lovable's published-site
auth gate; trust comes from the `state` row written by `xeroStart`.

```ts
// src/routes/api/public/xero.callback.ts
import { createFileRoute, redirect } from "@tanstack/react-router";
import { XERO, basicAuth } from "@/lib/xero-config";

export const Route = createFileRoute("/api/public/xero/callback")({
  server: {
    handlers: {
      GET: async ({ request }) => {
        const { supabaseAdmin } = await import("@/integrations/supabase/client.server");
        const { encrypt } = await import("@/lib/server/crypto.server");

        const APP_URL = process.env.APP_URL!;
        const back = (q: string) =>
          new Response(null, { status: 302, headers: { Location: `${APP_URL}/settings/xero${q}` } });

        const u = new URL(request.url);
        const code = u.searchParams.get("code");
        const state = u.searchParams.get("state");
        if (!code || !state) return back("?error=missing_params");

        const { data: flow } = await supabaseAdmin
          .from("xero_oauth_flows").select("*").eq("state", state).single();
        if (!flow || new Date(flow.expires_at) < new Date()) return back("?error=bad_state");

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

        const conns = await fetch(XERO.connections, {
          headers: { Authorization: `Bearer ${tokenSet.access_token}` },
        }).then((r) => r.json());

        const enc = encrypt(JSON.stringify(tokenSet));
        const expiresAt = new Date(Date.now() + tokenSet.expires_in * 1000).toISOString();
        for (const c of conns) {
          await supabaseAdmin.from("xero_connection").upsert(
            { tenant_id: c.tenantId, tenant_name: c.tenantName, token_set_enc: enc, status: "active", expires_at: expiresAt },
            { onConflict: "tenant_id" },
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

Load the row, refresh slightly early, and persist the rotated refresh token. With a
single org this is just a one-row update. Put this in a `.server.ts` helper, and
call it from inside server-function handlers.

```ts
// src/lib/server/xero-token.server.ts
import { XERO, basicAuth } from "@/lib/xero-config";
import { encrypt, decrypt } from "./crypto.server";

export async function getAccessToken(tenantId?: string) {
  const { supabaseAdmin } = await import("@/integrations/supabase/client.server");

  const q = supabaseAdmin.from("xero_connection").select("*");
  const { data: conn } = tenantId
    ? await q.eq("tenant_id", tenantId).single()
    : await q.limit(1).single();
  if (!conn) throw new Error("Xero is not connected yet");

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
      await supabaseAdmin.from("xero_connection")
        .update({ status: "needs_reconnect" }).eq("tenant_id", conn.tenant_id);
      throw new Error("Xero refresh failed — reconnect required");
    }
    tokenSet = await res.json();                          // refresh_token ROTATES — must persist
    await supabaseAdmin.from("xero_connection").update({
      token_set_enc: encrypt(JSON.stringify(tokenSet)),
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
**status** server function that returns only non-sensitive metadata (never the
token) and drive a banner from it.

```ts
// src/lib/xero-status.functions.ts
import { createServerFn } from "@tanstack/react-start";

export const xeroStatus = createServerFn({ method: "GET" }).handler(async () => {
  const { supabaseAdmin } = await import("@/integrations/supabase/client.server");
  const { data } = await supabaseAdmin.from("xero_connection")
    .select("tenant_name, status").limit(1).maybeSingle();
  return data
    ? { connected: data.status === "active", status: data.status, org: data.tenant_name }
    : { connected: false, status: "not_connected", org: null };
});
```

Frontend: read it (or refetch on focus) and show a reconnect CTA when needed —
the CTA runs the very same connect flow as the first-time hookup.

```tsx
const { data: xero } = useQuery({
  queryKey: ["xero-status"],
  queryFn: useServerFn(xeroStatus),
  refetchOnWindowFocus: true,
});

if (xero && !xero.connected) {
  return (
    <Banner tone="warning">
      {xero.status === "needs_reconnect"
        ? `Xero connection for ${xero.org} has expired — reconnect to keep data flowing.`
        : "Connect your Xero organisation to get started."}
      <button onClick={onConnect}>
        {xero.status === "needs_reconnect" ? "Reconnect Xero" : "Connect Xero"}
      </button>
    </Banner>
  );
}
```

Gate this endpoint too if even the org name is sensitive in your app. And mark
`needs_reconnect` *proactively*, not only after a failed refresh: a refresh token
unused for 60 days dies silently, so a low-traffic dashboard can read as
"connected" in the table yet be broken. A scheduled keep-alive refresh (see
gotchas) plus this banner keeps the displayed state honest.

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

### Lovable / TanStack Start gotchas

- **The callback MUST live under `src/routes/api/public/`.** That prefix bypasses
  Lovable's published-site auth gate; anywhere else, Xero's redirect lands on a
  login wall and the connect silently fails.
- **`xeroStart` must be gated by real admin auth** (Supabase Auth +
  `has_role(userId, 'admin')`). If your project has no auth yet, **add auth first**
  — don't ship the connect endpoint until it's gated. An unguarded `xeroStart`
  lets anyone overwrite the single token row with a different org's books.
- **The `state` value MUST be persisted.** The server runtime is stateless — connect
  and callback are separate invocations with no shared memory.
- **Never import `@/integrations/supabase/client.server` at module scope of a
  `*.functions.ts` or route file.** Top-level code in those files ships to the
  client bundle (only handler bodies are stripped). Load it inside the handler.
- **`process.env.*` is server-only and must be read inside the handler**, not at
  module scope. A module-level read returns `undefined` under the Worker runtime.
- **Service-role key never goes to the browser.** Surface only derived data
  (reports, invoices) through your own server functions; the token table has RLS
  with no anon/authenticated policies and no grants to those roles.
- **`npm:xero-node`** for read/write API calls is optional and must bundle cleanly
  for the Worker runtime — the `fetch`-based OAuth flow above avoids the dependency.

## Security checklist (do not skip)

- **Encrypt** the stored token set at rest; the key (`TOKEN_ENC_KEY`) lives in
  Lovable Cloud secrets, the rotating token in the DB.
- **Verify `state`** on every callback against the persisted flow row (CSRF), and
  **delete it after use** so a captured callback URL can't be replayed.
- **`xeroStart` is admin-gated** via `requireSupabaseAuth` + `has_role`. No auth
  yet? Add auth before exposing this function — there is no curl/shared-secret
  fallback in this skill on purpose.
- **The callback route is the only public endpoint**, and it only acts on a valid,
  unexpired, single-use `state`.
- **RLS on the token table with no user policies** — only the service role touches
  it. No `GRANT` to `anon` or `authenticated`.
- **Never expose the token or service-role key to the browser** — surface only
  derived data through your own server functions.
- On a failed refresh, **mark `needs_reconnect` and prompt re-connect** — don't loop.
- Keep all Xero access in **one place** (`xero-config.ts` + `xero-token.server.ts`).

## Verify

1. Connect your org once; confirm a single `xero_connection` row and that a read
   (e.g. Profit & Loss) returns your books.
2. Force a near-expiry token and confirm transparent refresh **and** that the rotated
   refresh token is persisted.
3. Revoke the app in Xero and confirm the app surfaces a reconnect state
   (`needs_reconnect`) instead of looping on failed refreshes.
4. Hit the callback route directly in a browser and confirm it isn't blocked by a
   login wall (i.e. it actually lives under `/api/public/`).
5. Confirm `xeroStart` rejects a non-admin caller (signed-out → 401 from
   middleware; signed-in non-admin → "Forbidden").
6. If your app writes to Xero, confirm a `POST` (e.g. create invoice) succeeds
   rather than returning 403 — proof the token carries write scopes, not just `.read`.
7. Flip a row to `needs_reconnect` and confirm the banner appears and its CTA
   re-runs the connect flow.
