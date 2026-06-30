---
name: xero-integration-single-account
description: Connect ONE Xero organisation that you own to a Lovable Cloud (TanStack Start + Lovable Cloud Postgres) app, so every visitor sees the same books through your single connection. Covers OAuth 2 (authorize → callback → refresh) with rotating refresh tokens, encrypted token storage, scope selection (granular > broad), idempotent writes, and the Lovable-specific traps around iframe preview, host mapping, and redirect URIs.
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
>   store, a back-office tool, a prototype. **This skill.**
> - **Many — one per customer.** Each end user links *their own* Xero. Stop and
>   use **`xero-integration-multi-account`** instead.

## Where this code runs in Lovable

TanStack Start on a Cloudflare Workers-style serverless runtime, with **Lovable Cloud**
for Postgres, auth, secrets, and deployment. Lovable Cloud runs on Supabase under the
hood, so the auto-generated env variables and imports still say `supabase` (e.g.
`@/integrations/supabase/client.server`). App-internal server logic — including all
Xero OAuth and token handling — lives in **`createServerFn` server functions**;
external callers (Xero's browser redirect, webhooks, cron) hit **server routes** under
`src/routes/api/public/*`. **Do not use Supabase Edge Functions** for any of this.

Two runtime facts shape the code below:

- The server runtime is stateless across invocations, so the OAuth `state` value
  must be persisted to a table between the connect step and the callback.
- Use `fetch` for the OAuth lifecycle (authorize redirect, token exchange, refresh).

**Crypto is Node `crypto`** (the Worker runtime supports it via `nodejs_compat`);
the three-part `iv.tag.enc` ciphertext shape below is correct on this stack.

---

## ⚠️ Pre-flight — do this FIRST (skipping it costs hours)

Three checks. Each one prevents a class of bug that produces a misleading
upstream error from Xero. Do them **before** you wire the Connect button to a
human.

### 1. Assert every secret is set and non-empty

Empty/missing secrets surface as `unauthorized_client` from Xero (because
`client_id=undefined` got sent), which points away from the real cause. Fail fast
with a clear message inside `xeroStart` **before** building the URL:

```ts
function requireEnv(name: string) {
  const v = process.env[name];
  if (!v || v.trim() === "") {
    throw new Error(`${name} is empty — set it in Project Secrets and retry.`);
  }
  return v;
}
```

Call `requireEnv("XERO_CLIENT_ID")`, `XERO_CLIENT_SECRET`, `XERO_SCOPES`, and
`TOKEN_ENC_KEY` at the top of every handler that uses them.

### 2. Log the constructed authorize URL once

You want a server log you can read when something is wrong, without ever
printing the secret value itself:

```ts
console.log("[xero] authorize", {
  client_id_len: process.env.XERO_CLIENT_ID?.length,
  redirect_uri: redirectUri,
  scope: process.env.XERO_SCOPES,
});
```

If the next failure is `unauthorized_client` or `invalid redirect_uri`, this
line tells you *exactly* which value Xero rejected.

### 3. Smoke-test `xeroStart` BEFORE clicking Connect

Call the server function and inspect the returned URL: `client_id` must be a
non-empty hex string, and `redirect_uri` must **exactly** match one of the URIs
registered in your Xero app (including scheme, host, and path). If either is
wrong, fix it here. Do not proceed to the popup — Xero's errors past this point
are far less useful than the URL you can read right now.

### 4. Validate `TOKEN_ENC_KEY` decodes to exactly 32 bytes

AES-256-GCM requires a **32-byte** key. A 32-character random string is NOT 32
bytes once base64-decoded (it's ~24), and `createCipheriv` throws
`Invalid key length` from deep inside `encrypt()` — which surfaces as an
opaque **500 on the callback**, *after* Xero has already consumed the
single-use `code`. The user sees "something went wrong" with no way to retry
without re-consenting, and no breadcrumb in the logs points at the key.

Make `key()` self-validating and tolerant of either shape:

```ts
// src/lib/server/crypto.server.ts
import { createCipheriv, createDecipheriv, createHash, randomBytes } from "node:crypto";
function key() {
  const k = process.env.TOKEN_ENC_KEY;
  if (!k) throw new Error("TOKEN_ENC_KEY is not set");
  const decoded = Buffer.from(k, "base64");
  if (decoded.length === 32) return decoded;                         // preferred shape
  if (k.length >= 32) return createHash("sha256").update(k, "utf8").digest();  // tolerant shim
  throw new Error("TOKEN_ENC_KEY too short — must be base64(32 bytes) or ≥32 chars");
}
```

The SHA-256 branch is a one-way compatibility shim so a mis-generated secret
doesn't crash the app; do not rely on it as the *intended* path. Generation
guidance is in §1.

---



## 1. Register your own Xero app

In the Xero developer portal (developer.xero.com) create a **Web app** OAuth2 app
(confidential — it holds a secret):

- Grab **Client ID** and **Client Secret**.
- **Register BOTH redirect URIs**, exactly:
  - **Published:** `https://project--<project-id>.lovable.app/api/public/xero/callback`
  - **Preview:** `https://id-preview--<project-id>.lovable.app/api/public/xero/callback`

  Tell the user to register both before first connect. The in-editor preview
  iframe host (`*.lovableproject.com`) and the dev-server origin (`localhost:8080`)
  are **NOT** valid redirect URIs — don't register them and don't send them.
  See the resolver in §3.

- Decide your scopes (see **Scopes** next). You MUST include **`offline_access`**
  or Xero returns no refresh token.

Store in Lovable Cloud secrets:
- `XERO_CLIENT_ID`, `XERO_CLIENT_SECRET`
- `XERO_SCOPES` (space-separated string)
- `TOKEN_ENC_KEY` — must base64-decode to exactly **32 bytes**. Call
  `generate_secret` with **`length: 44`** (the base64 length of 32 random
  bytes). The default 32-char length is the common mistake — it decodes to
  ~24 bytes and AES-256-GCM rejects it at runtime (see Pre-flight §4).

You do **not** need a `XERO_REDIRECT_URI` secret — the resolver in §3 derives it
from the request origin. `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are
already available via `process.env`. The refresh token **rotates on every
refresh** and lives in the database (a mutable store), never in a secret.

> Read `process.env.*` **inside** `.handler()`, not at module scope.

## Scopes

Scopes are **additive** — request the minimum and re-run consent to add more later.
You **cannot** widen scope on an existing token; you must re-consent. To get a
refresh token at all you must include `offline_access`.

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
only until **September 2027**. **Always prefer the granular scopes.**

Deprecated broad scope → the granular scopes that replace it:

| Deprecated (broad) | New granular scopes |
| --- | --- |
| `accounting.transactions` | `accounting.invoices`, `accounting.payments`, `accounting.banktransactions`, `accounting.manualjournals` |
| `accounting.transactions.read` | `accounting.invoices.read`, `accounting.payments.read`, `accounting.banktransactions.read`, `accounting.manualjournals.read` |
| `accounting.reports.read` | `accounting.reports.aged.read`, `accounting.reports.balancesheet.read`, `accounting.reports.banksummary.read`, `accounting.reports.budgetsummary.read`, `accounting.reports.executivesummary.read`, `accounting.reports.profitandloss.read`, `accounting.reports.trialbalance.read`, `accounting.reports.taxreports.read` |

What each granular scope covers:

| Scope | Grants |
| --- | --- |
| `accounting.invoices` | Invoices, CreditNotes, Quotes, PurchaseOrders, RepeatingInvoices, LinkedTransactions, Items |
| `accounting.payments` | Payments, BatchPayments, Overpayments, Prepayments |
| `accounting.banktransactions` | BankTransactions, BankTransfers |
| `accounting.manualjournals` | ManualJournals |
| `accounting.reports.*.read` | the matching report only |

### Scopes that were never broad (use as-is)

| Scope | Grants |
| --- | --- |
| `accounting.settings` / `.read` | Accounts, BrandingThemes, Currencies, Items, InvoiceReminders, Organisation, TaxRates, TrackingCategories, Users |
| `accounting.contacts` / `.read` | Contacts, ContactGroups |
| `accounting.attachments` / `.read` | attachments across most resources |
| `accounting.journals.read` | Journals (general ledger) |
| `accounting.budgets.read` | Budgets |

**Read vs write — choose deliberately; you can't change it later without
re-consent.** `.read` scopes are GET-only. Write scopes (no `.read` suffix) are
needed for any POST/PUT. Mismatched scope → 403 at runtime and the only fix is
to send the user through consent again.

Read-only dashboard:
```
openid profile email offline_access accounting.contacts.read accounting.invoices.read accounting.reports.profitandloss.read
```

Write / invoicing app:
```
openid profile email offline_access accounting.contacts accounting.invoices
```

Source: https://developer.xero.com/documentation/guides/oauth2/scopes

## 2. Token store (one encrypted row, RLS-locked)

```sql
create table public.xero_connection (
  tenant_id     text primary key,
  tenant_name   text,
  token_set_enc text not null,                 -- AES-256-GCM ciphertext
  status        text not null default 'active',-- 'active' | 'needs_reconnect'
  expires_at    timestamptz not null,
  updated_at    timestamptz not null default now()
);
grant all on public.xero_connection to service_role;
alter table public.xero_connection enable row level security;
-- INTENTIONALLY no anon/authenticated policies.

create table public.xero_oauth_flows (
  state      text primary key,
  created_at timestamptz not null default now(),
  expires_at timestamptz not null
);
grant all on public.xero_oauth_flows to service_role;
alter table public.xero_oauth_flows enable row level security;
```

AES-256-GCM helpers (Node `crypto`, three-part `iv.tag.ct` base64):

```ts
// src/lib/server/crypto.server.ts
import { createCipheriv, createDecipheriv, createHash, randomBytes } from "node:crypto";
function key() {
  const k = process.env.TOKEN_ENC_KEY;
  if (!k) throw new Error("TOKEN_ENC_KEY is not set");
  const decoded = Buffer.from(k, "base64");
  if (decoded.length === 32) return decoded;
  if (k.length >= 32) return createHash("sha256").update(k, "utf8").digest();  // tolerant shim
  throw new Error("TOKEN_ENC_KEY too short — must be base64(32 bytes) or ≥32 chars");
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

Shared Xero config (no secrets at module scope):

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
    .toString("base64").replace(/\+/g, "-").replace(/\//g, "_").replace(/=+$/, "");
```

---

## 3. The redirect-URI resolver — read carefully

> ### ⚠️ Three traps, all of which produce `invalid redirect_uri` or `unauthorized_client`
>
> **Trap A — `getRequest()` / `request.url` on the server returns
> `http://localhost:8080`.** Under the Worker runtime the inbound request you
> see inside a server function is on the internal dev host, not the public URL.
> Never derive `redirect_uri` from it.
>
> **Trap B — In the Lovable editor, `window.location.origin` is
> `https://<project-id>.lovableproject.com`** (the iframe host), which is NOT
> a registered URI. Sending it raw produces `invalid redirect_uri` every time.
>
> **Trap C — A single `XERO_REDIRECT_URI` secret can't cover both preview and
> published.** If you set it to the published host the preview breaks (and
> vice-versa). Use the resolver below instead.

Pass the client origin into `xeroStart`, then **map** unregistered hosts to the
matching registered URI:

```ts
// src/lib/xero-redirect.ts
const PROJECT_ID = "<your-project-id>";
const PREVIEW   = `https://id-preview--${PROJECT_ID}.lovable.app`;
const PUBLISHED = `https://project--${PROJECT_ID}.lovable.app`;
const CALLBACK  = "/api/public/xero/callback";

export function resolveRedirectUri(clientOrigin: string | undefined): string {
  const origin = (clientOrigin ?? "").trim();
  if (!origin) return `${PREVIEW}${CALLBACK}`;
  let host: string;
  try { host = new URL(origin).hostname; } catch { return `${PREVIEW}${CALLBACK}`; }
  // Editor iframe host — not registered with Xero.
  if (host.endsWith(".lovableproject.com")) return `${PREVIEW}${CALLBACK}`;
  // Local dev — not registered with Xero.
  if (host === "localhost" || host === "127.0.0.1") return `${PREVIEW}${CALLBACK}`;
  // Registered hosts pass through unchanged.
  if (origin.startsWith(PUBLISHED)) return `${PUBLISHED}${CALLBACK}`;
  if (origin.startsWith(PREVIEW))   return `${PREVIEW}${CALLBACK}`;
  // Custom domain or anything else → default to preview, force a published
  // visit if needed. Add an explicit branch here if you ship a custom domain.
  return `${PREVIEW}${CALLBACK}`;
}
```

Use the same resolver in **both** `xeroStart` (with the client-supplied origin)
and the callback route (with `new URL(request.url).origin` — which DOES work
correctly inside a server route because it's the actual public URL Xero hit).

### `xeroStart` (consent kickoff)

> ### ⚠️ Lock this behind admin auth before you connect (or accept the prototype risk)
>
> Anyone who can reach `xeroStart` can complete a connect and **overwrite your
> single token row with a different Xero org's tokens**, pointing your whole app
> at someone else's books. Production apps MUST gate `xeroStart` with
> `requireSupabaseAuth` + `has_role(userId, 'admin')`.

**Prototype escape hatch (v1/internal demos only).** If you're shipping an
unauthenticated dashboard for v1 and explicitly accept the risk, you may leave
`xeroStart` ungated **only if** all of the following hold:

- A visible, persistent warning banner on the dashboard saying "unauthenticated
  admin — do not share this URL".
- A clear plan and tracker entry to add auth before the app handles any traffic
  outside the team.
- You understand and accept these specific risks:
  1. Anyone with the URL can replace your stored Xero connection with theirs.
  2. The dashboard exposes derived revenue / order data to anyone with the URL.
  3. `/dashboard` cannot be indexed or shared publicly without leaking the above.
  4. Removing the warning banner without adding auth is a regression — flag it.

```ts
// src/lib/xero-start.functions.ts
import { createServerFn } from "@tanstack/react-start";
import { XERO, randomState } from "./xero-config";
import { resolveRedirectUri } from "./xero-redirect";

export const xeroStart = createServerFn({ method: "POST" })
  // .middleware([requireSupabaseAuth])  // ENABLE for production
  .inputValidator((d: { origin?: string } | undefined) => ({ origin: d?.origin ?? "" }))
  .handler(async ({ data /*, context */ }) => {
    const { supabaseAdmin } = await import("@/integrations/supabase/client.server");

    // Pre-flight: fail fast with a useful message.
    const clientId = process.env.XERO_CLIENT_ID;
    const scopes   = process.env.XERO_SCOPES;
    if (!clientId) throw new Error("XERO_CLIENT_ID is empty — set it in Project Secrets.");
    if (!scopes)   throw new Error("XERO_SCOPES is empty — set it in Project Secrets.");

    const redirectUri = resolveRedirectUri(data.origin);
    console.log("[xero] authorize", { client_id_len: clientId.length, redirectUri, scopes });

    const state = randomState();
    await supabaseAdmin.from("xero_oauth_flows").insert({
      state, expires_at: new Date(Date.now() + 10 * 60_000).toISOString(),
    });

    const url = new URL(XERO.authorize);
    url.searchParams.set("response_type", "code");
    url.searchParams.set("client_id", clientId);
    url.searchParams.set("redirect_uri", redirectUri);
    url.searchParams.set("scope", scopes);
    url.searchParams.set("state", state);
    return { url: url.toString() };
  });
```

### Click handler — iframe rule

> ### ⚠️ Open consent in a NEW TAB, opened SYNCHRONOUSLY in the click handler
>
> Xero's login refuses to be framed (`frame-ancestors`), so a same-frame
> redirect from the Lovable preview iframe shows a **blank white page** that
> looks like the button is broken. Detect the iframe and open a top-level tab.
> The `window.open()` call MUST happen *before* any `await`, or popup blockers
> will eat it.

```tsx
import { useServerFn } from "@tanstack/react-start";
import { xeroStart } from "@/lib/xero-start.functions";

const start = useServerFn(xeroStart);
async function onConnect() {
  const inIframe = window.self !== window.top;
  const popup = inIframe ? window.open("about:blank", "_blank") : null;  // sync!
  const { url } = await start({ data: { origin: window.location.origin } });
  if (popup) popup.location.href = url;
  else window.location.href = url;
}
```

The published app at `<project>.lovable.app` is not iframed, so the same-tab
redirect works there — handle both.

## 4. Callback — exchange, store your org

Lives at `/api/public/*` to bypass Lovable's published-site auth gate; trust
comes from the `state` row. Use the **same** `resolveRedirectUri` helper here —
the exchange call's `redirect_uri` MUST match the one used to start the flow.

```ts
// src/routes/api/public/xero.callback.ts
import { createFileRoute } from "@tanstack/react-router";
import { XERO, basicAuth } from "@/lib/xero-config";
import { resolveRedirectUri } from "@/lib/xero-redirect";

export const Route = createFileRoute("/api/public/xero/callback")({
  server: {
    handlers: {
      GET: async ({ request }) => {
        const u = new URL(request.url);
        const origin = u.origin;  // safe here — this IS the public URL Xero hit
        const back = (q: string) =>
          new Response(null, { status: 302, headers: { Location: `${origin}/dashboard${q}` } });
        try {
        const { supabaseAdmin } = await import("@/integrations/supabase/client.server");
        const { encrypt } = await import("@/lib/server/crypto.server");

        const redirectUri = resolveRedirectUri(origin);

        const code = u.searchParams.get("code");
        const state = u.searchParams.get("state");
        if (!code || !state) return back("?error=missing_params");

        const { data: flow } = await supabaseAdmin
          .from("xero_oauth_flows").select("*").eq("state", state).maybeSingle();
        if (!flow || new Date(flow.expires_at as string) < new Date())
          return back("?error=bad_state");

        const tokenRes = await fetch(XERO.token, {
          method: "POST",
          headers: {
            Authorization: `Basic ${basicAuth(process.env.XERO_CLIENT_ID!, process.env.XERO_CLIENT_SECRET!)}`,
            "Content-Type": "application/x-www-form-urlencoded",
          },
          body: new URLSearchParams({ grant_type: "authorization_code", code, redirect_uri: redirectUri }),
        });
        if (!tokenRes.ok) {
          console.error("[xero] token exchange failed", await tokenRes.text());
          return back("?error=token_exchange");
        }
        const tokenSet = await tokenRes.json();

        const conns = await fetch(XERO.connections, {
          headers: { Authorization: `Bearer ${tokenSet.access_token}` },
        }).then((r) => r.json()) as Array<{ tenantId: string; tenantName: string }>;

        const enc = encrypt(JSON.stringify(tokenSet));
        const expiresAt = new Date(Date.now() + tokenSet.expires_in * 1000).toISOString();
        for (const c of conns) {
          await supabaseAdmin.from("xero_connection").upsert(
            { tenant_id: c.tenantId, tenant_name: c.tenantName, token_set_enc: enc,
              status: "active", expires_at: expiresAt, updated_at: new Date().toISOString() },
            { onConflict: "tenant_id" },
          );
        }
        await supabaseAdmin.from("xero_oauth_flows").delete().eq("state", state);
        return back("?connected=1");
        } catch (e) {
          // Any throw past token_exchange MUST funnel into the same ?error/&detail
          // redirect — a bare 500 is unrecoverable here because Xero's `code` is
          // single-use and the user has to re-consent before you can read the next log.
          console.error("[xero] callback error", e);
          const detail = encodeURIComponent(e instanceof Error ? e.message : String(e));
          return back(`?error=callback&detail=${detail}`);
        }
      },
    },
  },
});
```

## 5. Use it per request (refresh + persist rotation)

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
      body: new URLSearchParams({ grant_type: "refresh_token", refresh_token: tokenSet.refresh_token }),
    });
    if (!res.ok) {
      await supabaseAdmin.from("xero_connection")
        .update({ status: "needs_reconnect" }).eq("tenant_id", conn.tenant_id);
      throw new Error("Xero refresh failed — reconnect required");
    }
    tokenSet = await res.json();   // refresh_token ROTATES — must persist
    await supabaseAdmin.from("xero_connection").update({
      token_set_enc: encrypt(JSON.stringify(tokenSet)),
      expires_at: new Date(Date.now() + tokenSet.expires_in * 1000).toISOString(),
      updated_at: new Date().toISOString(),
    }).eq("tenant_id", conn.tenant_id);
  }
  return { accessToken: tokenSet.access_token as string, tenantId: conn.tenant_id as string };
}
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
  body: JSON.stringify({ Invoices: [{ /* ... */ }] }),
});
```

## 6. Surface connection status (reconnect UX)

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

Drive a banner from it; refetch on focus. `needs_reconnect` should surface
immediately with a CTA that re-runs the connect flow. Refresh tokens unused for
60 days die silently, so on a low-traffic dashboard add a scheduled keep-alive
refresh, not just reactive detection.

Also read `?error=…&detail=…` from the URL on `/dashboard` and render it in
the connection banner. The callback (§4) redirects every failure with those
params; without surfacing them, a callback crash looks identical to "not
connected yet" and the user has no idea anything happened.



## Date + money + tax normalisation

- **Dates.** REST returns `/Date(1518685950940+0000)/`. Normalise to ISO; send
  dates as `YYYY-MM-DD`.

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

- **Money.** Numbers, not formatted strings. Control rounding with `unitdp` (≤4).
  Postgres `numeric` comes back as strings via the Supabase client — convert before parse.

- **Tax is your org's.** Your org's tax config is applied to invoices, so
  `AmountDue` / `Total` may exceed line amounts (e.g. 15% GST turns 2495 into
  2869.25). The `UnitAmount` you posted is still correct — don't "fix" the
  discrepancy. Set `LineAmountTypes` (`Exclusive`/`Inclusive`/`NoTax`) deliberately.

## Idempotency (avoid duplicate writes)

Two layers.

**Layer 1 — app level:** dedup the logical action (stable order id, unique
constraint) so a double-submit never reaches Xero twice; upsert contacts by email.

**Layer 2 — Xero idempotency:** `Idempotency-Key` header on mutating calls.

Rules that bite:
- Only POST/PUT/PATCH honour the key; ignored on GET.
- Max 128 chars. A `crypto.randomUUID()` (36) is plenty.
- Keys expire after ~6 minutes — protects retries, not long-term dedup.
- Same key + different request → `400`. Persist the key and replay identically.
- Errors are cached too. To recover after an errored keyed request, GET to check
  whether the resource was created, then retry with a **new** key.

## Other Xero gotchas

- **`offline_access` is mandatory for refresh.** Omit it → access token dies in
  ~30 min with no recovery.
- **Token lifetimes:** access ~30 min; refresh ~60 days and **rotates on every
  refresh** — persist or break. Refresh dies if unused for 60 days.
- **Always send `Xero-Tenant-Id`** on every API call — there is no implicit org.
- **Contacts aren't deduplicated by email.** Look up by email/`ContactID` via
  `GET /Contacts` and reuse the id.
- **Invoice status flow:** `DRAFT` → `SUBMITTED` → `AUTHORISED`; only
  `AUTHORISED` are real receivables and expose `OnlineInvoiceUrl`. Use `ACCREC`
  (sales) vs `ACCPAY` (bills) correctly.
- **Rate limits:** ~60/min, 5,000/day, ~5 concurrent. On **429** read
  `Retry-After` and back off.
- **`summarizeErrors=false`** on batch creates returns per-item errors.

### Lovable / TanStack Start gotchas

- **Callback MUST live under `src/routes/api/public/`** — that prefix bypasses
  Lovable's published-site auth gate.
- **`xeroStart` MUST be admin-gated in production** (or accept the prototype
  risks in §3).
- **`state` MUST be persisted** — connect and callback are separate invocations.
- **Never import `@/integrations/supabase/client.server` at module scope** of a
  `*.functions.ts` or route file. Load it inside the handler.
- **`process.env.*` is server-only and must be read inside the handler.**
- **`getRequest()` / `request.url` inside a server function returns
  `http://localhost:8080`** under the Worker runtime — useless for `redirect_uri`.
- **`window.location.origin` in the editor is `*.lovableproject.com`** — also
  not registered. Always run it through `resolveRedirectUri`.
- **Service-role key never goes to the browser.**

## Security checklist

- Encrypt the stored token set at rest (`TOKEN_ENC_KEY` in secrets, rotating
  token in DB).
- Verify `state` on every callback against the persisted flow row; delete after
  use (single-use, replay-proof).
- `xeroStart` admin-gated, or the prototype escape hatch with banner + risks
  acknowledged.
- Callback is the only public endpoint; only acts on a valid, unexpired,
  single-use `state`.
- RLS on the token table with no user policies; no GRANT to anon/authenticated.
- Never expose the token or service-role key to the browser.
- On refresh failure, mark `needs_reconnect` and prompt re-connect.
- Keep all Xero access in one place (`xero-config.ts` + `xero-token.server.ts`
  + `xero-redirect.ts`).

## Verify

0. **Smoke-test before clicking Connect.** Call `xeroStart` and inspect the
   returned URL — `client_id` non-empty, `redirect_uri` exactly matches a URI
   registered in your Xero app. Fix before touching the popup.
0.5. **Force an encryption failure once.** Temporarily set `TOKEN_ENC_KEY` to
   a 16-char string and click Connect. Confirm `/dashboard` shows
   `?error=callback&detail=…` (with a readable message) rather than a blank
   500 — this proves both the callback try/catch (§4) and the dashboard
   banner (§6) actually wire failures back to the user. Restore the real
   key afterwards.
1. Connect your org; confirm one `xero_connection` row and a real read works.
2. Force a near-expiry token; confirm transparent refresh **and** rotated
   refresh token persisted.
3. Revoke the app in Xero; confirm the banner flips to `needs_reconnect`
   instead of looping.
4. Hit the callback route in a browser; confirm it isn't blocked by a login
   wall (proof it's under `/api/public/`).
5. Confirm `xeroStart` rejects non-admin callers (production gating).
6. If your app writes: confirm a `POST` succeeds (proof scopes carry write).
7. Flip a row to `needs_reconnect`; confirm banner CTA re-runs connect.
