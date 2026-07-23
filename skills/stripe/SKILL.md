---
name: stripe
description: Stripe billing — customers, subscriptions, MRR, invoices, refunds, payment links, payouts, balance. Activate BEFORE any Stripe call; it returns the namespace and rules to follow.
---

# Stripe Workflows

## Namespace

Calls go through `gateway.stripe.<method>(args)` from inside `execute_javascript`
or `execute_python`. Never call them as top-level tools.

If a call returns `gateway.stripe.foo is not a world tool`, the error lists every
valid method — pick from it. Never re-send the same call.

## Cardinal rules

1. **Money is integer minor units.** Always. `$25.00` → `2500`. `€1.50` → `150`. Never send `25` or `25.00` for "twenty-five dollars". Always pass `currency` alongside (`"usd"`, `"eur"`, lowercase).
2. **IDs come from tool results, never invented.** Customer ids `cus_*`, subscriptions `sub_*`, invoices `in_*`, charges `ch_*`, prices `price_*`, etc.
3. **Confirm money-moving writes with the user.** `refundsCreate`, `subscriptionsCancel`, `invoicesPay`, `paymentLinksCreate` — confirm the charge id / subscription id / amount before firing.
4. **Pagination is cursor-based.** `limit` (≤100) + `starting_after` (the id of the last item from the previous page). Cap loops — don't paginate forever.
5. **Search vs list.** `*List` ops accept simple equality filters (`customer`, `status`). For anything else (`amount >= 10000`, `created >= timestamp`, free-text) use the matching `*Search` op with Stripe's search DSL.

## Customer health — "is ACME up to date?"

The headline field is `customer.delinquent` (true if the most recent invoice
attempt failed and hasn't resolved). Combine with subscription status for the
full picture:

```javascript
// Find ACME by email
const r = await gateway.stripe.customersSearch({
  query: `email:'billing@acme.com'`,
  limit: 1,
});
const cust = r.data[0];
if (!cust) { console.log("No customer"); return; }

// Active subscriptions for this customer
const subs = await gateway.stripe.subscriptionsList({
  customer: cust.id,
  status: "all",
  limit: 10,
});
const active = subs.data.filter(s => ["active", "trialing", "past_due", "unpaid", "paused"].includes(s.status));

console.log(`${cust.name ?? cust.email}: delinquent=${cust.delinquent}`);
for (const s of active) {
  console.log(`  sub ${s.id}: ${s.status}, renews ${new Date(s.current_period_end*1000).toISOString()}`);
}
```

Status meanings:
- `active` / `trialing` — happy path
- `past_due` — last invoice payment failed, dunning in progress (still has access in most app patterns)
- `unpaid` — gave up after the dunning window; access usually revoked
- `paused` — collection paused (you or Stripe Smart Retries)
- `canceled` — terminated
- `incomplete` / `incomplete_expired` — initial payment never succeeded

## Customer monthly spend — "what does X pay us?"

MRR for one customer is the sum across their `active` / `trialing` subscriptions
of `unit_amount × quantity` normalised to monthly:

```javascript
const subs = await gateway.stripe.subscriptionsList({
  customer: "cus_xxx",
  status: "active",
  limit: 100,
});
let monthlyCents = 0;
for (const s of subs.data) {
  for (const item of s.items.data) {
    const p = item.price;
    if (p.type !== "recurring" || p.unit_amount == null) continue;
    const months = p.recurring.interval === "year" ? 12 * p.recurring.interval_count
      : p.recurring.interval === "month" ? p.recurring.interval_count
      : p.recurring.interval === "week" ? p.recurring.interval_count / 4.345
      : p.recurring.interval_count / 30.4;   // day
    monthlyCents += (p.unit_amount * item.quantity) / months;
  }
}
console.log(`MRR: $${(monthlyCents / 100).toFixed(2)}`);
```

For lifetime / past spend use `chargesList` with `customer:` and sum `amount` where
`status === "succeeded"` and subtract `amount_refunded`.

## At-risk customers — "who's about to churn?"

```javascript
const overdue = await gateway.stripe.subscriptionsList({ status: "past_due", limit: 100 });
for (const s of overdue.data) {
  console.log(`${s.customer}: past_due since ${new Date(s.current_period_start*1000).toDateString()}`);
}
```

For "trial ending soon" search subscriptions by `trial_end` proximity (use the search
DSL via `subscriptionsList` filtered on `status: "trialing"` then sort client-side
on `trial_end`).

## Failed charges this week

```javascript
const weekAgo = Math.floor(Date.now()/1000) - 7*86400;
const r = await gateway.stripe.chargesSearch({
  query: `status:'failed' AND created>=${weekAgo}`,
  limit: 100,
});
const byReason = {};
for (const c of r.data) {
  const k = c.failure_code ?? "unknown";
  byReason[k] = (byReason[k] ?? 0) + 1;
  console.log(`${c.id} ${c.amount/100} ${c.currency} — ${c.failure_code}: ${c.failure_message}`);
}
console.log("By reason:", byReason);
```

## Refunding a charge

**Confirm with the user before calling.** Show the charge id, amount, and customer.

```javascript
// Full refund
await gateway.stripe.refundsCreate({ charge: "ch_xxx" });

// Partial refund (in minor units — $50 of a $200 charge)
await gateway.stripe.refundsCreate({
  charge: "ch_xxx",
  amount: 5000,
  reason: "requested_by_customer",
});
```

## Cancelling a subscription

Two flavours — pick deliberately:

```javascript
// Graceful — bills until period_end, then stops
await gateway.stripe.subscriptionsUpdate({
  subscription_exposed_id: "sub_xxx",
  cancel_at_period_end: true,
});

// Hard — stop now
await gateway.stripe.subscriptionsCancel({ subscription_exposed_id: "sub_xxx" });
```

Default to graceful unless the user explicitly says "now" / "immediately".

## Sending a one-shot invoice

Three steps for ad-hoc billing (e.g. "send ACME a $5,000 invoice for Q2 services"):

```javascript
// 1. Create the line item against the customer
await gateway.stripe.invoiceItemsCreate({
  customer: "cus_xxx",
  amount: 500000,                    // $5,000 in cents
  currency: "usd",
  description: "Q2 2026 professional services",
});

// 2. Wrap the pending items into an invoice; auto_advance=true tells Stripe
//    to finalise and attempt collection
const inv = await gateway.stripe.invoicesCreate({
  customer: "cus_xxx",
  collection_method: "send_invoice",
  days_until_due: 30,
  auto_advance: true,
});

// 3. Email the hosted invoice link
await gateway.stripe.invoicesSend({ invoice: inv.id });
console.log(`Sent: ${inv.hosted_invoice_url}`);
```

For collection from an on-file card use `collection_method: "charge_automatically"`
and skip the `invoicesSend` step.

## Generating a payment link

For ad-hoc "click to pay" links you can share via email/Slack:

```javascript
const link = await gateway.stripe.paymentLinksCreate({
  line_items: [{ price: "price_xxx", quantity: 1 }],
  metadata: { source: "assistant" },
});
console.log(`Pay here: ${link.url}`);
```

You need an existing Price (`price_*`) — look one up via `pricesList` first.

## Retrying a failed invoice

```javascript
const result = await gateway.stripe.invoicesPay({
  invoice: "in_xxx",
  off_session: true,            // unattended retry
});
console.log(`status=${result.status} attempts=${result.attempt_count} paid=${result.paid}`);
```

## Next payout

```javascript
const r = await gateway.stripe.payoutsList({ status: "in_transit", limit: 5 });
for (const p of r.data) {
  const arrival = new Date(p.arrival_date * 1000).toDateString();
  console.log(`${p.amount/100} ${p.currency} arriving ${arrival}`);
}
```

Or `status: "pending"` for ones not yet on the way.

## Platform balance

```javascript
const b = await gateway.stripe.balanceGet({});
for (const a of b.available) console.log(`Available: ${a.amount/100} ${a.currency}`);
for (const a of b.pending)   console.log(`Pending:   ${a.amount/100} ${a.currency}`);
```

## Pitfalls

- **Money is integer minor units, always.** `$25` → `2500`. The most common LLM error.
- **`metadata` deletes via empty string.** Pass `metadata: {key: ""}` to remove a key, NOT `{key: null}`.
- **Subscription status `"all"` includes `canceled`** — don't be surprised by old terminated subs in the list when you pass it. Filter client-side if you want active-only.
- **Search has a separate cursor** — `next_page` (string), not `starting_after` (id). `*Search` ops use page tokens; `*List` ops use object-id cursors.
- **Lists don't return `total`.** Use `*Search` with `expand[]=total_count` if you genuinely need a count, otherwise paginate-and-sum.
- **Customer search is eventually consistent** — a customer created via `customersCreate` may not appear in `customersSearch` for ~1 second. Don't search-then-create-if-missing in tight loops.
- **Currency is always lowercase.** `"usd"`, not `"USD"`.
- **`price.unit_amount` can be null** for tiered or per-unit metered prices — handle the null branch when summing.
- **Never create raw `Charge` objects.** Modern Stripe flows go through Invoices, Subscriptions, or PaymentIntents. This realm only exposes Charges as read.
- **`subscription_exposed_id` is the path param name** for the Subscription endpoints — Stripe's spec quirk. Pass `{subscription_exposed_id: "sub_xxx"}`, not `{subscription: ...}`, when calling `subscriptionsGet/Update/Cancel`.
