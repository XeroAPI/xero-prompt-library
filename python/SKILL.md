---
name: xero-integration-multi-tenant-python
description: Build a MULTI-TENANT Xero integration in PYTHON where EACH end user of the app connects THEIR OWN Xero organisation, with their own access + refresh tokens. Use whenever building a multi-tenant Python SaaS on Xero (Flask, FastAPI, Django, etc.) — every customer authorizes their own Xero and the app acts as that customer. Covers the OAuth2 + PKCE consent leg with an OAuth client (Authlib), the official `xero-python` SDK for API calls and refresh, per-user encrypted token storage, the global token getter/saver trap, refresh/rotation, tenant selection, scopes (including the granular-scope migration), and the Xero API gotchas that bite in production (date/money/tax normalisation, idempotent writes, rate limits). Read this BEFORE writing any multi-tenant Xero code in Python. If instead the app talks to a SINGLE Xero org that the builder owns, do NOT use this flow — see "Do you actually need this?" at the top for the much simpler single-tenant approach. For a JavaScript/TypeScript app, use the `xero-integration-multi-tenant` (xero-node) skill instead.
---

# Multi-tenant Xero in Python (each user connects their own Xero)

This skill is for a **multi-tenant SaaS written in Python**: every customer links
**their own** Xero organisation, and the app reads/writes Xero **as that customer**,
with **their** tokens. That means owning the entire OAuth2 lifecycle yourself.

The official `xero-python` SDK is **not** a full OAuth client. It manages the *token*
(storage hook, refresh, API calls) but does **not** perform the consent redirect or
the code exchange — you bring your own OAuth client (Authlib below) for that leg, then
hand the resulting token to `xero-python`. Internalise that split before you start.

## Do you actually need this?

Multi-tenant is real work — a per-user token vault, refresh rotation, request-scoped
tenant context. Don't take it on unless the app needs it. If the app talks to a
**single** Xero that the *builder* owns — an internal tool, a dashboard over your own
books, a single-merchant storefront, a prototype — **stop and use the single-tenant
approach instead**:

- **Custom Connection** (recommended for single-org, server-to-server): a
  machine-to-machine client-credentials app tied to **one** organisation, no consent
  redirect. `xero-python` exchanges your client id/secret directly:
  ```python
  api_client = ApiClient(Configuration(oauth2_token=OAuth2Token(
      client_id=CLIENT_ID, client_secret=CLIENT_SECRET)))
  api_client.get_client_credentials_token()   # no user-facing redirect
  AccountingApi(api_client).get_organisations("")  # tenant id often "" for custom connections
  ```
- **A single stored refresh token**: register a normal Web App, run the consent flow
  **once** yourself, persist the one resulting token dict, and refresh it on a schedule.

Use this multi-tenant skill **only** when each end user must connect their *own* Xero.

> **Don't confuse Xero multi-org with multi-tenant.** `GET /connections` returns several
> *organisations* under ONE login — still one principal. Multi-tenant means many
> *separate* Xero logins, one per customer.

## Decision check before building

Build the full flow below only if ALL are true:

- Each customer connects their **own** Xero org (not yours).
- The app acts under **each customer's** Xero identity, not a shared one.
- You can securely store and rotate per-user refresh tokens.

If any is false, the single-tenant approach above is enough.

## Libraries

```bash
# Match the project's package manager — check for the lockfile first:
#   uv.lock          -> uv add xero-python authlib cryptography sqlalchemy
#   poetry.lock      -> poetry add xero-python authlib cryptography sqlalchemy
#   Pipfile.lock     -> pipenv install xero-python authlib cryptography sqlalchemy
#   requirements.txt -> pip install xero-python authlib cryptography sqlalchemy
pip install xero-python authlib cryptography sqlalchemy
```

- **`xero-python`** — official SDK: API calls, token refresh, tenant discovery.
- **`authlib`** — the OAuth client for the consent + code-exchange leg, with clean PKCE
  support. (The official Xero samples use `flask-oauthlib`, which is **deprecated** —
  prefer Authlib.) Works framework-agnostically with Flask, FastAPI, or Django.
- **`cryptography`** — AES-GCM encryption for tokens at rest.
- **`sqlalchemy`** — example token store; swap for the project's existing ORM.

Keep **all** Xero access in one module so paths, tenant handling, refresh, the
getter/saver wiring, and the gotchas below live in one place.

## 1. Register your own Xero app

In the Xero developer portal (developer.xero.com) create a **Web App** OAuth2 app:

- Grab the **Client ID** and **Client Secret**.
- Register your redirect URI **exactly** — scheme, host, port, and path must match
  byte-for-byte what your code sends. Register both prod and local dev:
  - prod: `https://your-app.example.com/oauth/xero/callback`
  - local: `http://localhost:8000/oauth/xero/callback`  *(match your dev server port)*

  Xero permits `http://localhost` for development; every other redirect must be `https`.
  **Tell the user the exact redirect URIs to register** rather than guessing.
- Decide your scopes (see **Scopes**). You MUST include **`offline_access`** or Xero
  returns no refresh token.

Store **only your app's** Client ID/Secret as environment variables via your secret
manager — a git-ignored `.env` locally, your platform's secret store in prod. Also add a
32-byte **`TOKEN_ENC_KEY`** (base64) for encrypting stored tokens. **Never commit
secrets, and never put per-user tokens in env** — those go in the database, encrypted.

## Scopes

Scopes are **additive** — request the minimum needed and re-run consent to add more.
You **cannot** widen scope on an existing token; the user must re-consent. To get a
refresh token at all you must include `offline_access`.

### User / OpenID scopes (identity)

| Scope | Description |
| --- | --- |
| `offline_access` | required for a refresh token (offline connection) |
| `openid` | use the user's identity |
| `profile` | first name, last name, full name, Xero user id |
| `email` | email address |

### Accounting API — prefer granular scopes

Xero is replacing **broad** scopes with **granular** ones. Since **March 2026** all new
and existing Web/PKCE apps are assigned granular scopes; custom connections since
**29 April 2026**. Broad scopes keep working for apps that already used them only until
**September 2027**. **Always prefer the granular scopes** — use `accounting.invoices`,
NOT `accounting.transactions`.

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

Source of truth (re-check periodically — dates and assignments change):
https://developer.xero.com/documentation/guides/oauth2/scopes

## 2. Per-user token store (database, encrypted)

One row per (user, Xero tenant). The token is an oauthlib-style dict
(`access_token`, `refresh_token`, `expires_at`, `token_type`, `scope`, `id_token`) —
store it encrypted as JSON, plus the selected `tenant_id`. Example uses SQLAlchemy;
adapt column types to the project's ORM but keep the shape.

```python
# models.py
from sqlalchemy import Column, Integer, String, DateTime, UniqueConstraint, func
from sqlalchemy.orm import declarative_base

Base = declarative_base()

class XeroConnection(Base):
    __tablename__ = "xero_connections"
    id = Column(Integer, primary_key=True)
    user_id = Column(String, nullable=False)        # YOUR app's user
    tenant_id = Column(String, nullable=False)       # Xero org id
    tenant_name = Column(String)
    token_set_enc = Column(String, nullable=False)   # ENCRYPTED JSON of the token dict
    expires_at = Column(DateTime(timezone=True), nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())
    __table_args__ = (UniqueConstraint("user_id", "tenant_id", name="xero_user_tenant"),)
```

AES-256-GCM helpers (key from `TOKEN_ENC_KEY`):

```python
# crypto.py
import os, base64
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

KEY = base64.b64decode(os.environ["TOKEN_ENC_KEY"])  # 32 bytes

def encrypt(plain: str) -> str:
    nonce = os.urandom(12)
    ct = AESGCM(KEY).encrypt(nonce, plain.encode(), None)
    return base64.b64encode(nonce + ct).decode()

def decrypt(blob: str) -> str:
    raw = base64.b64decode(blob)
    return AESGCM(KEY).decrypt(raw[:12], raw[12:], None).decode()
```

## 3. The biggest difference from the JS SDK: token getter/saver are GLOBAL

`xero-python` does not take a token per call. Instead you register two callbacks on the
`ApiClient` — a **getter** it calls to read the current token before each request, and a
**saver** it calls after a refresh. The official sample wires these to the Flask
`session`, i.e. **one user**. In multi-tenant that is a trap: a global session means
every request shares one token. **Wire the getter/saver to request-scoped context** that
resolves *which* connection the current request is acting as. Use a `ContextVar` so it is
correct under threads and async.

```python
# xero.py
import json, contextvars
from dataclasses import dataclass
from xero_python.api_client import ApiClient
from xero_python.api_client.configuration import Configuration
from xero_python.api_client.oauth2 import OAuth2Token
from .crypto import encrypt, decrypt

@dataclass
class ConnCtx:
    user_id: str
    tenant_id: str
    token: dict          # current oauthlib token dict
    token_set_enc: str   # ciphertext of the token set as loaded (for sibling refresh)

current_conn: contextvars.ContextVar[ConnCtx | None] = contextvars.ContextVar(
    "current_conn", default=None)

api_client = ApiClient(
    Configuration(oauth2_token=OAuth2Token(
        client_id=os.environ["XERO_CLIENT_ID"],
        client_secret=os.environ["XERO_CLIENT_SECRET"],
    )),
    pool_threads=1,
)

@api_client.oauth2_token_getter
def _get_token():
    ctx = current_conn.get()
    if ctx is None:
        raise RuntimeError("No Xero connection context set for this request")
    return ctx.token

@api_client.oauth2_token_saver
def _save_token(token: dict):
    ctx = current_conn.get()
    # CRITICAL multi-tenant trap: one consent can authorize several orgs that SHARE one
    # token set. A refresh rotates (and invalidates) the old refresh token, so update
    # EVERY row that shared the previous ciphertext — not just this tenant's row — or the
    # siblings will hold a dead token and wrongly flip to needs_reconnect on next call.
    update_tokens_for_shared_set(ctx.user_id, ctx.token_set_enc, encrypt(json.dumps(token)),
                                 expires_at=token["expires_at"])
    ctx.token = token
```

## 4. Start consent (per user, with state AND code_verifier)

Authlib runs the OAuth leg. PKCE means there is a **`code_verifier`** the callback needs;
because the callback is a *separate* HTTP request, persist both `state` (CSRF + link to
the initiating user) **and** `code_verifier` server-side, keyed to the user. The SDK will
not remember either for you.

```python
from authlib.integrations.requests_client import OAuth2Session

AUTH_URL = "https://login.xero.com/identity/connect/authorize"
TOKEN_URL = "https://identity.xero.com/connect/token"
SCOPES = os.environ.get("XERO_SCOPES",
    "openid profile email offline_access accounting.contacts accounting.invoices")

def build_consent_url(user_id: str) -> str:
    oauth = OAuth2Session(
        client_id=os.environ["XERO_CLIENT_ID"],
        client_secret=os.environ["XERO_CLIENT_SECRET"],
        scope=SCOPES,
        redirect_uri=os.environ["XERO_REDIRECT_URI"],
        code_challenge_method="S256",          # PKCE
    )
    url, state = oauth.create_authorization_url(AUTH_URL)
    save_flow_state(user_id=user_id, state=state,
                    code_verifier=oauth.code_verifier,   # MUST persist for the callback
                    expires_at=now() + timedelta(minutes=10))
    return url
```

### Local-dev gotchas (these bite during testing)

- **Redirect URI must match exactly.** A port or trailing-slash mismatch between
  `XERO_REDIRECT_URI` and the portal registration is the most common local failure
  (`unauthorized_client` / invalid redirect). Run the dev server on the registered port.
- **`http://localhost` only.** Xero accepts `http` solely for `localhost`.
- **`oauthlib` refuses plain http.** For local `http://localhost` testing set
  `os.environ["OAUTHLIB_INSECURE_TRANSPORT"] = "1"` **in dev only** — never in prod.
- **Don't embed the consent page in an iframe.** Xero's login sets `frame-ancestors` and
  refuses framing — a same-frame redirect from an embedded preview shows a blank page.
  Open consent as a top-level navigation / new tab.

## 5. Callback — exchange, discover tenants, store

```python
from xero_python.identity import IdentityApi

def handle_callback(request_url: str, state: str):
    flow = load_flow_state(state)                       # must exist, match session, unexpired
    if not flow:
        raise ValueError("Invalid state")

    oauth = OAuth2Session(
        client_id=os.environ["XERO_CLIENT_ID"],
        client_secret=os.environ["XERO_CLIENT_SECRET"],
        redirect_uri=os.environ["XERO_REDIRECT_URI"],
        code_verifier=flow.code_verifier,               # PKCE: same verifier as consent
    )
    token = oauth.fetch_token(TOKEN_URL, authorization_response=request_url)  # dict

    # Hand the token to the SDK via a one-shot context so IdentityApi can use it.
    ctx = ConnCtx(user_id=flow.user_id, tenant_id="", token=dict(token),
                  token_set_enc=encrypt(json.dumps(dict(token))))
    current_conn.set(ctx)

    # A single consent can authorize MULTIPLE orgs that all SHARE ONE token set.
    # Don't assume connections[0]: store all, or let the user pick. Encrypt ONCE and write
    # the identical ciphertext to every row so refresh can find and update siblings.
    connections = IdentityApi(api_client).get_connections()
    token_set_enc = encrypt(json.dumps(dict(token)))
    for c in connections:                               # c.tenant_id, c.tenant_name, c.tenant_type
        if c.tenant_type != "ORGANISATION":
            continue
        upsert_connection(user_id=flow.user_id, tenant_id=c.tenant_id,
                          tenant_name=c.tenant_name, token_set_enc=token_set_enc,
                          expires_at=token["expires_at"])
    delete_flow_state(state)
```

## 6. Use it per request (set context, refresh, persist rotation)

```python
import time, json
from contextlib import contextmanager
from xero_python.accounting import AccountingApi, Invoice, Invoices, Contact

@contextmanager
def xero_for(user_id: str, tenant_id: str):
    conn = load_connection(user_id, tenant_id)
    if not conn:
        raise LookupError("User has not connected this Xero org")
    token = json.loads(decrypt(conn.token_set_enc))
    ctx = ConnCtx(user_id, tenant_id, token, conn.token_set_enc)
    reset = current_conn.set(ctx)
    try:
        # Proactively refresh if within ~2 min of expiry. refresh_oauth2_token() calls the
        # saver, which rotates ALL sibling rows (see section 3).
        if token.get("expires_at", 0) - time.time() < 120:
            api_client.set_oauth2_token(token)
            api_client.refresh_oauth2_token()           # raises if the refresh token is dead
        yield api_client, tenant_id
    finally:
        current_conn.reset(reset)

# Example write (explicit tenant_id, summarize_errors, unitdp, idempotency_key):
with xero_for(user_id, tenant_id) as (client, tid):
    accounting = AccountingApi(client)
    invoices = Invoices(invoices=[Invoice(...)])   # type, contact, line_items, ...
    accounting.create_invoices(
        tid, invoices,
        summarize_errors=False,                     # get per-item validation errors
        unitdp=4,
        idempotency_key=order.idempotency_key or str(uuid.uuid4()),
    )
```

On a failed refresh (dead/expired refresh token), **mark the connection broken and prompt
re-connect** — do not loop on failed refreshes.

## Date + money + tax normalisation

- **Dates.** `xero-python` parses Xero's Microsoft-JSON date form
  (`/Date(1518685950940+0000)/`) into `datetime` for you, and accepts `datetime`/`date`
  on the way in. If you ever drop to raw REST (e.g. a paged report endpoint),
  `datetime.fromisoformat` **cannot** parse the MS-JSON string — normalise it:

  ```python
  import re
  from datetime import datetime, timezone

  def normalize_xero_date(v):
      if v is None or isinstance(v, datetime):
          return v
      if isinstance(v, str):
          m = re.search(r"/Date\((-?\d+)([+-]\d{4})?\)/", v)
          if m:
              return datetime.fromtimestamp(int(m.group(1)) / 1000, tz=timezone.utc)
          try:
              return datetime.fromisoformat(v.replace("Z", "+00:00"))
          except ValueError:
              return None
      return None
  ```

- **Money.** The SDK uses `float` for amounts. If you need exact arithmetic, convert to
  `Decimal` deliberately (`Decimal(str(amount))`, never `Decimal(float)`). Control rounding
  on writes with `unitdp` (unit decimal places, up to 4). Amounts like `amount_due` come
  back as numbers.

- **Tax is the org's, not yours.** Each connected organisation applies **its own** tax
  config, so `amount_due` / `total` may exceed the sum of your line amounts — e.g. a 15%
  GST org turns a 2495 line into 2869.25. The `unit_amount` you posted is still correct;
  do **not** "fix" the discrepancy. Set `line_amount_types` (`Exclusive`, `Inclusive`,
  `NoTax`) deliberately if you need control. **In multi-tenant this varies per customer** —
  never hard-code a tax assumption; whatever each org returns is correct for that org.

## Idempotency (avoid duplicate writes)

Defend at two layers.

**Layer 1 — app level (cheapest):** don't call Xero twice for the same logical action.
Upsert contacts by email (see gotchas) and dedup the order on a stable identifier (a DB
unique constraint) so a double-submitted form never reaches Xero twice.

**Layer 2 — Xero idempotency (the safety net):** pass `idempotency_key=...` on the
mutating call. The accounting create methods accept it as a keyword argument; the order is
`create_invoices(tenant_id, invoices, summarize_errors, unitdp, idempotency_key)` and
similar for other resources. Xero caches the first response and returns it for any repeat
with the same key, so a network-timeout retry can't create a second invoice.

Rules that bite (from Xero's idempotency guide):

- **Only POST/PUT/PATCH** honour the key; ignored on GET.
- **Max 128 characters.** A single `uuid.uuid4()` (36 chars) is plenty unique — use that as
  the default.
- **Keys expire after ~6 minutes.** They protect transient retries, not long-term dedup.
  Long-term dedup is Layer 1's job.
- **Same key + a *different* request → `400` "used with a different request".** A request
  differs if URL, body, or method changes. Persist the key with the record and reuse the
  *identical* request when retrying.
- **Errors are cached too.** A keyed request that errored internally returns the *same
  cached error* on retry with the same key, even after the underlying problem is fixed. To
  recover, **GET to check whether the resource was actually created**; if not, create again
  with a **new** key.

When a user *re-submits* a previously **failed** order later, mint a **new** key — but
first GET to confirm no invoice was actually created, so you don't duplicate a silent
success.

## Other Xero gotchas (these bite in production)

- **Getter/saver are global — keep context per request.** Restated because it is the #1
  Python multi-tenant bug: never resolve "the current token" from a global/module/session
  variable. Always go through the `ContextVar` set at the start of the request.
- **`offline_access` is mandatory for refresh.** Omit it and the access token dies in
  ~30 min with no way to refresh — the user must re-consent.
- **Token lifetimes:** access token ~30 min; refresh token ~60 days and **rotates on every
  refresh**. You MUST persist the new refresh token each time (the saver does this) or the
  next refresh fails. A refresh token also dies if unused for 60 days.
- **Always pass `tenant_id` explicitly** to every `AccountingApi` call. There is no implicit
  "current org"; the wrong id silently writes to the wrong company's books.
- **Re-fetch connections periodically.** `IdentityApi.get_connections()` can change as users
  add/remove orgs; re-run it, not just at first callback.
- **Contacts are not auto-deduplicated by email.** Posting an invoice with a new contact
  name creates a NEW contact every time. Upsert yourself: look up by email (or `contact_id`)
  via `get_contacts` and reuse the id.
- **Invoice status flow:** `DRAFT` → `SUBMITTED` → `AUTHORISED`. Only `AUTHORISED` invoices
  are real receivables and expose an online payment URL (`get_online_invoice`). Use `ACCREC`
  (sales) vs `ACCPAY` (bills) correctly.
- **Rate limits (per tenant):** ~60 calls/minute, 5,000/day, plus an app-wide per-minute cap
  and ~5 concurrent requests. On **429** read the `Retry-After` header and back off —
  multi-tenant apps hit this fast when looping over orgs. Queue/space calls per tenant.
- **`summarize_errors=False`** on batch creates to get per-item validation errors instead of
  one opaque failure.

## Security checklist (do not skip)

- **Encrypt** the stored token set at rest; the key (`TOKEN_ENC_KEY`) lives in your secret
  manager / env and is never committed.
- **Verify `state`** on every callback and bind it to the session (CSRF); persist and check
  the PKCE `code_verifier` too.
- Keep PKCE on (`code_challenge_method="S256"`) — don't disable it.
- **Never expose tokens to the browser** or any client-readable storage.
- Set `OAUTHLIB_INSECURE_TRANSPORT` **only** in local dev, never in production.
- On a failed refresh, **mark the connection broken and prompt re-connect** — don't retry
  forever.
- Keep all Xero access in **one module**; resolve tenant context per request.
- Add `.env` (and any local token dumps) to `.gitignore`.

## Don't want to hand-roll it?

Managed token-vault platforms (**Nango**, **Composio**, etc.) implement per-user OAuth +
refresh + encryption for Xero and many providers. For a multi-provider SaaS this is often
the pragmatic choice — you still call the Xero API the same way, but the vault and rotation
are managed for you.

## Verify

1. Two app users each connect a **different** Xero org; confirm two distinct token rows and
   that each user's writes land in their own org (check `tenant_id`), with no context bleed
   between concurrent requests.
2. Force a near-expiry token and confirm transparent refresh **and** that the rotated
   refresh token is persisted to all sibling rows.
3. Revoke the app in one user's Xero and confirm your app surfaces a "reconnect" state
   instead of looping on failed refreshes.
4. Post the same invoice twice with the same `idempotency_key` and confirm only one invoice
   is created.
