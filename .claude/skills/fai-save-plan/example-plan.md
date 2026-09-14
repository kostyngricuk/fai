# Example plan

A worked example of the required structure. Fictional ticket, fictional codebase — copy the
*shape*, not the content. Note especially: the explicit scope boundary, the `path:line` citations,
the "does NOT exist" facts, the per-step agent tags, and the execution log.

---8<--- EXAMPLE BEGINS ---8<---

---
ticket: TID-1234
feature-agents: [fai-checkout, fai-pricing]
status: in-progress
created: 2026-09-10
updated: 2026-09-14
---

# TID-1234 — Stop non-stackable promos from compounding (cart total only)

## Context

Two promos flagged `isStackable: false` currently compound instead of the larger one winning. A
customer applying a 30% and a 40% promo gets 58% off rather than 40%. Finance flagged it after
three days of over-discounted orders.

The fix belongs in the cart total calculation. The promo *payload* already carries the flag from the
pricing service — `GET /v2/pricing/promos` returns `isStackable` per promo (confirmed against the
staging response), so no backend change is needed.

**Scope boundary.** Cart total calculation only. Explicitly **not** included: the promo picker UI
(which should also grey out incompatible promos — deferred to TID-1251), the order-summary display,
and any retroactive correction of already-placed orders (owned by the finance team, TID-1248).

## Current-state facts (verified)

- `src/cart/promo.ts` — `applyPromo` (L44-91) folds every promo in `order.promos` unconditionally.
  It reads `promo.percentage` and ignores `promo.isStackable` entirely.
- `src/cart/total.ts` — `computeTotal` (L20-38) calls `applyPromo` once per promo in array order.
- `src/types/pricing.ts` — `Promo` (L12-19) **already declares** `isStackable: boolean`. No type
  change is needed.
- `src/cart/rules/` — **does NOT exist.** There is no rules module in the cart today; guard logic
  currently lives inline in `promo.ts`.
- `__tests__/cart/promo.ts` (L1-60) covers single-promo cases only. There is **no** multi-promo
  test at all.
- `fai/features/checkout.md` §5 states the invariant "at most one non-stackable promo applies per
  order" — it is documented but was never enforced in code.

## Implementation steps

### 1. Partition promos before folding  🟣 fai-checkout

`src/cart/promo.ts` — add a partition step at the top of `applyPromo`'s caller path:

```ts
export function selectApplicablePromos(promos: Promo[]): Promo[] {
  const stackable = promos.filter((p) => p.isStackable);
  const exclusive = promos.filter((p) => !p.isStackable);
  if (exclusive.length === 0) return stackable;
  const best = exclusive.reduce((a, b) => (b.percentage > a.percentage ? b : a));
  return [best, ...stackable];
}
```

Rationale: the invariant is "at most one non-stackable promo", not "drop all but the first" — order
in `order.promos` is arrival order, not preference, so picking the largest is the customer-favorable
reading finance confirmed.

### 2. Route `computeTotal` through the partition  🟣 fai-checkout

`src/cart/total.ts:24` — replace `order.promos.forEach(...)` with
`selectApplicablePromos(order.promos).forEach(...)`. No signature change, so no call-site churn.

### 3. Keep the pricing payload honest  🟢 fai-pricing

`src/pricing/mapPromo.ts:31` — `isStackable` is currently defaulted to `true` when the field is
absent. Flip the default to `false`: a promo whose stackability the service did not state must not
silently compound. Add the missing-field case to `__tests__/pricing/mapPromo.ts`.

### 4. Cover the multi-promo cases  🟣 fai-checkout

`__tests__/cart/promo.ts` — add: two exclusives (larger wins), one exclusive plus two stackables
(all three apply), and the 30%+40% regression from the ticket asserting a 40% total.

## Verification

```sh
yarn lint
yarn typecheck
yarn jest __tests__/cart __tests__/pricing
```

Expect all green. `__tests__/legacy/checkout.ts` has one pre-existing failure
(`renders deprecated banner`) unrelated to this change — ignore it.

Manual: apply the two promos from the ticket in the staging cart and confirm a 40% total.

## Out of scope / dependencies

- **TID-1251** — promo picker should grey out incompatible promos. Deferred; this fix is
  defence-in-depth behind it.
- **TID-1248** — retroactive correction of already-placed orders. Owned by the finance team.
- Order-summary display still recomputes its own subtotal at `src/order/summary.ts:77`. Out of
  scope here, but flagged: it will disagree with the cart until it uses `computeTotal`.

## Execution log

- [x] 1. Partition promos before folding — done 2026-09-12, `a1b2c3d`
- [x] 2. Route `computeTotal` through the partition — done 2026-09-12, `a1b2c3d`
- [ ] 3. Flip the `isStackable` default in the pricing mapper
- [ ] 4. Cover the multi-promo cases
