# pack-stripe

Stripe billing via a hand-curated, vendored OpenAPI 3 spec — the chat-driven
95% of Stripe's API (customers, subscriptions, invoices, charges, refunds,
payouts, balance, payment links) typed end-to-end so the LLM never guesses.

> Pack authoring reference: see
> [`docs/pack-format.md`](https://github.com/embabel/assistant/blob/main/docs/pack-format.md)
> in the assistant repo for the full pack format spec — vendored
> OpenAPI specs, OAuth2, identity introspection, admin OAuth app
> registry, and per-workspace overrides are all documented there.

## Why

Stripe's official OpenAPI dump is ~600 ops and 100k+ lines. Most of it
(Issuing, Capital, Connect platform admin, Climate) is irrelevant to billing
Q&A. Worse: feeding it to an LLM as tool definitions wastes thousands of
tokens per turn. This pack vendors a curated mini-spec covering the operations
that matter for "is ACME up to date?", "what's their MRR?", "refund this
charge", "send a $5k invoice", "when's our next payout?" — same data, ~3% of
the prompt.

A second consideration: Stripe is the only major API that accepts ONLY
`application/x-www-form-urlencoded` request bodies (it explicitly rejects
JSON). The assistant's OpenAPI runtime detects this from the spec's
`requestBody.content` and switches encoding automatically — including
flattening nested objects and lists into Stripe's bracket form syntax
(`metadata[key]=v`, `line_items[0][price]=p`). End-to-end you write
JavaScript-shaped objects; the wire format Just Works.

## Namespace

Methods land under `gateway.stripe`. E.g.:

- `gateway.stripe.customersSearch({ query: "email:'a@b.com'" })`
- `gateway.stripe.subscriptionsList({ customer: "cus_xxx", status: "active" })`
- `gateway.stripe.invoicesPay({ invoice: "in_xxx", off_session: true })`
- `gateway.stripe.refundsCreate({ charge: "ch_xxx", amount: 5000 })`
- `gateway.stripe.payoutsList({ status: "in_transit" })`
- `gateway.stripe.balanceGet({})`

See `prompts/examples.md` for end-to-end patterns and `skills/stripe/SKILL.md`
for the workflow guidance the LLM activates before any Stripe call.

## Auth — OAuth2 via Stripe Connect

End users **never** paste API keys, never see `sk_live_...` strings, and
never set environment variables. They click **Authorize** in
Settings → Connected Services. That's it.

This works because the assistant deployment is registered as a Stripe
Connect platform — every end user connects their own Stripe account against
that single platform, the same way every "Connect with Stripe" button on
the web works.

### For end users

1. Open **Settings → Connected Services**.
2. Click **Authorize** on the `stripe` row.
3. Consent on Stripe's page (it'll show your platform's name + logo). Done — `gateway.stripe.*` is live in chat.

If the row shows **"Not configured"**, the deployment operator hasn't
registered a Stripe Connect platform yet — show them the next section.

### For installation admins (one-time setup)

Done once per installation. Every workspace inherits — end users just
click Authorize.

1. **Register a Stripe Connect platform.**
   - Go to <https://dashboard.stripe.com/settings/applications> (use the
     Stripe account you want users connecting **to** — for "users connect
     their own Stripe", this is your platform account, not a destination
     account).
   - Click **Get started with Connect** if you haven't already.
   - Under **OAuth settings**, configure:

2. **Redirect URI**

   Set to your assistant's public callback URL:

   ```
   https://your-host/api/v1/auth/oauth2/callback
   ```

   For local dev: `http://localhost:8042/api/v1/auth/oauth2/callback`.

   Stripe lets you register multiple redirect URIs — add both prod and dev
   so the same Connect platform serves both environments.

3. **Choose the integration type**

   Use **Standard Connect** (the default). The OAuth `access_token`
   returned from Stripe's `/oauth/token` endpoint **is** the connected
   account's secret key — there's no separate token to mint. The assistant
   stores it and sends it as `Authorization: Bearer <token>` on every
   call.

   - "Express" and "Custom" Connect are intended for marketplace flows
     where the platform fully controls the connected account. For
     "users bring their own Stripe account" — which is what this pack
     does — Standard is the only correct choice.

4. **Scopes**

   Standard Connect uses one of two scopes — pass the one you want at
   the authorization URL (the framework reads it from `apis/apis.yml`):

   - `read_write` (default in this pack) — full API access. Required for
     refunds, payment links, sending invoices, cancelling subscriptions.
   - `read_only` — read endpoints only. Use this if the installation
     should only browse Stripe data, never write. To switch, edit the
     `scopes:` value in `apis/apis.yml`.

5. **Grab the two halves of the OAuth credential pair.**

   Stripe Connect splits these across two different dashboard pages,
   which trips everyone up the first time:

   - **`client_id`** — comes from
     <https://dashboard.stripe.com/settings/connect> (the page from step 1).
     Look for "Test mode client ID" / "Live mode client ID" — it's
     prefixed `ca_xxxxxxxxxxxxxxxxxxxxxxxxxx`. There's a separate one
     for each mode; copy the one matching the environment you're
     setting up.
   - **`client_secret`** — there is **no separate "OAuth client secret"
     field anywhere in the Connect UI.** Stripe reuses your account's
     **regular secret API key** as the OAuth client secret. Get it from
     <https://dashboard.stripe.com/apikeys> — "Secret key", prefixed
     `sk_test_...` (test mode) or `sk_live_...` (live mode). Use the
     **standard** key, not a restricted key.

   Yes, this means your platform's full-power secret key is also the
   OAuth client secret. That's how Stripe Connect works — it's
   documented [here](https://docs.stripe.com/connect/oauth-reference#post-token)
   ("Use your live or test secret API key as the `client_secret`").

6. **Add them to** `{workspaceBase}/admin/oauth-apps.yml` (the same admin
   directory that holds `pack-sources.yml`, `themes/`, `hints/`, etc.):

   ```yaml
   apps:
     stripe:
       client-id: ca_xxxxxxxxxxxxxxxxxxxxxxxxxx
       client-secret: sk_test_xxxxxxxxxxxxxxxxxxxxxxxxxx
   ```

   Hot-reloaded — no restart needed. Every workspace will see "Authorize"
   appear in Settings → Connected Services.

   **Match the modes.** Test-mode `ca_*` pairs with `sk_test_*`. Live-mode
   `ca_*` pairs with `sk_live_*`. Crossing them will fail at the token
   exchange step. Simplest pattern: one assistant deployment per mode,
   each pointing at the matching Stripe environment.

7. **Test the flow.**
   - Restart any client browsers, open Settings → Connected Services, click
     Authorize on the stripe row.
   - Stripe will prompt the user to either select an existing Stripe
     account or sign up a new one.
   - On consent, the assistant exchanges the code for an access token and
     stores it. The row should flip to "Connected — <account email>".
   - Run a test call: ask "what's my Stripe balance?" — you should see
     the LLM call `balanceGet`.

A specific workspace can opt out of the installation default and point at
its own Stripe Connect platform by writing the same shape to
`<workspace>/config/oauth-apps.yml` — useful if a team needs a different
brand on Stripe's consent screen.

### Token lifetime

Stripe Standard Connect access tokens **do not expire**. The user can
revoke at any time from their Stripe dashboard
(`https://dashboard.stripe.com/settings/applications`) — the assistant will
get 401s on the next call and surface a "reconnect" prompt.

There is no refresh token; if the user revokes, they re-Authorize from
Settings → Connected Services.

### Webhooks (optional)

This pack covers request/response API calls only. If you want event-driven
workflows (notify on `invoice.payment_failed`, ping on `customer.subscription.deleted`),
configure a Stripe webhook endpoint pointing at your assistant's webhook
ingest URL — that's outside this pack's scope, but a separate pack or
the assistant's built-in webhook tools can subscribe.

## Object types covered

| Resource     | List | Get | Create | Update | Delete | Search | Notes                              |
|--------------|------|-----|--------|--------|--------|--------|------------------------------------|
| Customers    | ✓    | ✓   | ✓      | ✓      |        | ✓      | Headline `delinquent` field        |
| Subscriptions| ✓    | ✓   |        | ✓      | ✓      |        | Cancel via DELETE or `cancel_at_period_end` |
| Invoices     | ✓    | ✓   | ✓      |        |        |        | + `pay`, `send`, `invoiceItemsCreate` |
| Charges      | ✓    | ✓   |        |        |        | ✓      | Read-only — use Invoices/PaymentLinks to charge |
| Refunds      |      |     | ✓      |        |        |        |                                    |
| Payouts      | ✓    |     |        |        |        |        | "Next payout"                       |
| Balance      | ✓    |     |        |        |        |        | Single endpoint                     |
| Products     | ✓    |     |        |        |        |        | Read-only — manage via dashboard    |
| Prices       | ✓    |     |        |        |        |        | Read-only                           |
| PaymentLinks |      |     | ✓      |        |        |        | Generate a hosted URL               |

## Sample data

Stripe test mode is the canonical sandbox — every Stripe account has a
test mode toggle in the dashboard. Connecting against your platform's test
key gives you a fully populated environment with cards (`4242 4242 4242 4242`
for success, `4000 0000 0000 9995` for `insufficient_funds`, etc.) and zero
real-money risk. See <https://stripe.com/docs/testing> for the full deck.

## What's NOT in this pack

To keep the spec small, these are out of scope:

- **Connect platform admin** — onboarding accounts, managed accounts,
  capability requests; intended for marketplace flows, separate spec
- **Issuing** — virtual/physical card issuing
- **Capital / Treasury** — financing, bank-account-as-a-service
- **Identity / Climate / Tax** — separate APIs, add as needed
- **Webhook subscription management** — out-of-band; configure in dashboard
- **Files API** — uploading dispute evidence etc; rarely needed in chat
- **Direct `Charge` creation** — modern flows go through Invoices,
  Subscriptions, or PaymentIntents, all of which this pack covers
- **Payment Methods CRUD** — customers manage these via Stripe-hosted
  billing portal; the pack reads them indirectly via Customer / Charge
