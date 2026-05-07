# pack-stripe — usage examples

The vendored OpenAPI spec exposes namespace `stripe`. All write operations use
`application/x-www-form-urlencoded` (Stripe explicitly rejects JSON); the
gateway flattens nested objects/lists into Stripe's bracket syntax automatically.

## Why this pack exists

Stripe's official OpenAPI dump is ~600 operations and 100k+ lines — overwhelming
for an LLM tool surface, and most of it (Connect platform admin, Issuing,
Capital, Climate) is irrelevant to billing-Q&A workflows. This pack hand-curates
the ~22 operations that handle the chat-driven 95%: customer health, MRR,
dunning, refunds, payouts, balance, payment links, ad-hoc invoicing.

## Common patterns

### Customer health snapshot

```javascript
const r = await gateway.stripe.customersSearch({
  query: `email:'${email}'`,
  limit: 1,
});
const c = r.data[0];
if (!c) { console.log(`No customer for ${email}`); return; }

const subs = await gateway.stripe.subscriptionsList({
  customer: c.id, status: "all", limit: 50,
});
const live = subs.data.filter(s => s.status !== "canceled" && s.status !== "incomplete_expired");

console.log({
  customer: c.id,
  name: c.name ?? c.email,
  delinquent: c.delinquent,
  balance: c.balance,
  liveSubs: live.length,
  byStatus: live.reduce((a, s) => ({ ...a, [s.status]: (a[s.status] ?? 0) + 1 }), {}),
});
```

### Top accounts by MRR

```javascript
const subs = [];
let starting_after;
for (let p = 0; p < 20; p++) {
  const r = await gateway.stripe.subscriptionsList({
    status: "active", limit: 100, starting_after,
  });
  subs.push(...r.data);
  if (!r.has_more) break;
  starting_after = r.data[r.data.length - 1].id;
}

const monthlyByCustomer = {};
for (const s of subs) {
  for (const item of s.items.data) {
    const p = item.price;
    if (p.type !== "recurring" || p.unit_amount == null) continue;
    const months = p.recurring.interval === "year"  ? 12 * p.recurring.interval_count
                 : p.recurring.interval === "month" ? p.recurring.interval_count
                 : p.recurring.interval === "week"  ? p.recurring.interval_count / 4.345
                 :                                    p.recurring.interval_count / 30.4;
    const cents = (p.unit_amount * item.quantity) / months;
    monthlyByCustomer[s.customer] = (monthlyByCustomer[s.customer] ?? 0) + cents;
  }
}
const top = Object.entries(monthlyByCustomer)
  .sort((a, b) => b[1] - a[1])
  .slice(0, 10);
console.log(top.map(([id, c]) => ({ customer: id, mrr: `$${(c/100).toFixed(2)}` })));
```

### Dunning queue — past_due subs and the failure reason

```javascript
const overdue = await gateway.stripe.subscriptionsList({ status: "past_due", limit: 100 });
for (const s of overdue.data) {
  if (!s.latest_invoice) continue;
  const inv = await gateway.stripe.invoicesGet({ invoice: s.latest_invoice });
  console.log(`${s.customer}: invoice ${inv.number} — ${inv.amount_remaining/100} ${inv.currency}, attempt ${inv.attempt_count}`);
}
```

### Cards expiring in the next 30 days

Stripe doesn't expose this as a single op — search by metadata or paginate
through customers and read `default_source` / payment methods. For most chat
workflows the simpler answer is "ask the customer to update via Stripe-hosted
billing portal" and skip the introspection.

### Refunding the latest charge for a customer

```javascript
const charges = await gateway.stripe.chargesList({ customer: "cus_xxx", limit: 1 });
const ch = charges.data[0];
if (!ch || ch.status !== "succeeded") { console.log("No refundable charge"); return; }
console.log(`About to refund ${ch.amount/100} ${ch.currency} on ${ch.id}`);
// → user confirms before this line:
await gateway.stripe.refundsCreate({ charge: ch.id, reason: "requested_by_customer" });
```

### Generating a $5k one-off invoice

```javascript
await gateway.stripe.invoiceItemsCreate({
  customer: "cus_xxx",
  amount: 500000,
  currency: "usd",
  description: "Q2 services",
});
const inv = await gateway.stripe.invoicesCreate({
  customer: "cus_xxx",
  collection_method: "send_invoice",
  days_until_due: 30,
  auto_advance: true,
});
await gateway.stripe.invoicesSend({ invoice: inv.id });
console.log(inv.hosted_invoice_url);
```

### Quick payment link for an existing Price

```javascript
const link = await gateway.stripe.paymentLinksCreate({
  line_items: [{ price: "price_xxx", quantity: 1 }],
});
console.log(link.url);
```
