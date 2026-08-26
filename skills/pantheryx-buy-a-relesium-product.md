---
name: buy-a-relesium-product
description: >-
  Search the Relesium storefront catalog, build a cart, and take a buyer through an approved
  checkout using the live Universal Commerce Protocol MCP endpoint on relesium.com.
api: Relesium Agentic Commerce (UCP / MCP)
endpoint: https://relesium.com/api/ucp/mcp
transport: MCP (JSON-RPC 2.0)
operations:
  - search_catalog
  - lookup_catalog
  - get_product
  - create_cart
  - update_cart
  - get_cart
  - cancel_cart
  - create_checkout
  - update_checkout
  - get_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
generated: '2026-08-26'
method: generated
source: >-
  Tool names, required inputs and semantics taken verbatim from the live tools/list response at
  https://relesium.com/api/ucp/mcp (HTTP 200, 2026-08-26) and the store's published agent
  instructions at https://relesium.com/llms.txt (HTTP 200). No operation named here was invented.
---

# Buy a Relesium product

Relesium is a PanTheryx consumer brand. Its storefront implements the Universal Commerce Protocol
over MCP, so you can transact without scraping the site.

## Before you start

- Endpoint: `POST https://relesium.com/api/ucp/mcp`, `Content-Type: application/json`,
  `Accept: application/json, text/event-stream`.
- No credential is needed to discover tools or read the catalog.
- **Every tool requires `meta.ucp-agent.profile`** — a URI identifying your agent profile. Calls
  without it fail schema validation.
- Confirm capabilities first with `GET https://relesium.com/.well-known/ucp`. It declares the
  protocol version (currently `2026-04-08`, with `2026-01-23` still supported) and the payment
  handlers the store accepts: Google Pay, Shopify card, and Shop Pay.
- The catalog is small. At last probe it carried a single published product,
  "Relesium GLP-1 Digestive Support".

## Steps

1. **Find the product.** Call `search_catalog` with the buyer's intent. Use `lookup_catalog` or
   `get_product` when you already have an identifier. Pass `context.address_country` and
   `context.currency` so pricing and availability come back correct for the buyer.
2. **Read prices correctly.** Every amount is an integer in ISO 4217 *minor* units paired with a
   currency code: `{"amount": 2500, "currency": "USD"}` is $25.00. Divide by 100 before you quote a
   USD or EUR price to a person. Zero-decimal currencies such as JPY are already whole units.
   Quoting minor units to a buyer is the most likely mistake on this surface.
3. **Build the cart.** `create_cart`, then `update_cart` to adjust quantities. `get_cart` re-reads
   state. If the buyer changes their mind, `cancel_cart` — this is safe and reversible.
4. **Open the checkout.** `create_checkout` returns line items, totals, and any discounts or taxes.
   Checkout IDs are Shopify global IDs in the form `gid://shopify/Checkout/abc123`.
5. **Set shipping.** `update_checkout` carries the shipping address and method. The store's UCP
   profile declares single-destination shipping only (`allows_multi_destination.shipping: false`),
   so do not attempt to split an order across addresses.
6. **Get explicit approval, then complete.** `complete_checkout` returns the order ID and a thank-you
   page URL. The store's own instructions are unambiguous: *"Agents must not complete payment
   without explicit buyer consent."* If you cannot get contemporaneous approval at the moment of
   payment, do not complete — route the purchase through the Shop skill at
   `https://shop.app/SKILL.md` instead.
7. **Confirm.** `get_order` reads the resulting order.

## Reversal — know this before step 6

`cancel_cart` and `cancel_checkout` will unwind anything you have built up to that point.

**Nothing in this API reverses `complete_checkout`.** There is no refund, void or reverse tool. The
only published path back is out of band: the refund policy at
`https://relesium.com/policies/refund-policy` says to email `info@relesium.com` to start a return.
It states **no return window and no eligibility conditions**, so you cannot promise a buyer how long
they have to change their mind. Do not invent one. Say that returns are handled by email and that
the store does not publish a deadline.

## Errors and pacing

- Errors arrive as JSON-RPC 2.0 error objects. There is no published error-code reference, and
  `complete_checkout` is documented only as returning "any errors encountered" — surface the raw
  error to the buyer rather than guessing at its meaning.
- The endpoint is rate-limited per IP. Back off on `429`. No numeric limit and no `Retry-After`
  header is published, so use exponential backoff.

## The same skill on the sibling storefront

Life's First Naturals (`https://www.lifesfirstnaturals.com/api/ucp/mcp`) is the same PanTheryx
brand surface with an identical 13-tool set. At last probe its catalog was **empty** — the tools
answer, but `products.json` returned zero products, so `search_catalog` has nothing to return.
