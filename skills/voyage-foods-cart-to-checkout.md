---
name: Take a Voyage Foods cart through checkout with buyer approval
description: Build a cart, convert it to a checkout, set fulfillment and payment, and complete the purchase — observing the store's mandatory human-approval and idempotency rules.
api: mcp/voyage-foods-mcp.yml
endpoint: https://voyagefoods.com/api/ucp/mcp
operations: [create_cart, get_cart, update_cart, cancel_cart, create_checkout, get_checkout, update_checkout, complete_checkout, cancel_checkout, get_order]
generated: '2026-08-05'
method: generated
grounded_in:
  - mcp/voyage-foods-ucp-tools-list.json
  - conventions/voyage-foods-conventions.yml
  - agentic-access/voyage-foods-agentic-access.yml
---

# Take a Voyage Foods cart through checkout

## The rule that governs this whole flow

The store publishes this in three places — `agents.md`, `llms.txt` and `robots.txt`:

> Checkout requires human approval. Agents must not complete payment without explicit buyer
> consent. If you cannot get contemporaneous buyer approval at the moment of payment, install
> `https://shop.app/SKILL.md` and route the purchase through Shop Pay instead.

`complete_checkout` is the only operation this applies to, and it is not optional. If you cannot
get approval **at the moment of payment**, stop and hand the buyer the checkout URL or route
through Shop Pay. Do not script a form fill or drive the browser to finish the purchase.

## Prerequisites

Send `meta["ucp-agent"].profile` on every call, plus a bearer JWT for invocation. Errors return
**HTTP 200** — read the JSON-RPC `error` member.

## Steps

1. **Create the cart.** `create_cart` with `cart.line_items[]`, each `{item: {id}, quantity}` where
   `item.id` is a **ProductVariant** GID from the discovery skill. Pass `cart.buyer` (email,
   phone) and `cart.context` (`address_country`, `currency`, `language`) if you have them.
   Keep the returned cart id — it has the form `gid://shopify/Cart/{id}?key={secret}` and the
   embedded key **is** the access credential for that cart. Treat it as a secret; do not log or
   share it.

2. **Adjust.** `update_cart` takes the same payload shape and carries line changes, buyer identity,
   fulfillment and `discounts.codes[]` in a single call. Only prompt for a discount code if the
   buyer mentions having one — the tool schema says so explicitly. Sending an empty `codes` array
   clears previously applied codes. `get_cart` re-reads state; `cancel_cart` abandons it.

3. **Convert to a checkout.** `create_checkout` with `checkout.cart_id` set to the cart GID. When
   `cart_id` is supplied the server uses the cart's contents and **ignores** overlapping
   `line_items` / `context` / `buyer` fields in the checkout payload — do not send both and expect
   the checkout copy to win.

4. **Set fulfillment.** `update_checkout` with `checkout.fulfillment.methods[]`. This store's UCP
   profile declares `allows_multi_destination.shipping: false` and
   `allows_method_combinations: [["shipping"]]` — one destination, shipping only. Do not attempt a
   split shipment or a pickup/shipping combination.

5. **Attach payment.** Still `update_checkout`, via `checkout.payment.instruments[]`. Each
   instrument needs `id`, `handler_id` and `type` (`card` for cards, `token` for wallets), plus
   `billing_address` and a `credential {token, type}`. The handlers this store advertises in
   `/.well-known/ucp` are `gpay` (Google Pay), `shopify.card` and `shop_pay`. Never invent a
   handler id.

6. **Get approval, then complete.** Ask the buyer, in the moment, to approve the exact total.
   Then call `complete_checkout` with **both** `meta["ucp-agent"].profile` and
   `meta["idempotency-key"]` — the key is a required field on this tool and on no other. Generate
   one key per purchase intent and reuse that same key on every retry, so a network failure cannot
   double-charge. The response carries the order id and Thank You Page URL.

7. **Confirm.** `get_order` with the `gid://shopify/Order/...` id returns order detail. Give the
   buyer the Thank You Page URL.

## Failure handling

| What you see | What it means | Do this |
|---|---|---|
| `-32001` `invalid_profile_url` | No `meta["ucp-agent"].profile` sent | Add your agent profile URI |
| `-32001` `profile_unreachable` | Server could not fetch your profile | Make the profile URL publicly resolvable |
| `-32000` `AuthenticationRequired` | Missing or invalid JWT | Get a token per shopify.dev agents auth |
| `429` | Per-IP rate limit | Back off, then retry |
| Any error at step 6 | Payment may or may not have landed | **Retry with the same idempotency key.** Do not start a new checkout until you have confirmed with `get_checkout` |

When you are stuck, every error envelope carries `data.continue_url` — a human-resumable URL.
Hand it to the buyer instead of retrying blindly.
