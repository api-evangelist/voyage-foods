---
name: Discover the Voyage Foods catalog
description: Find Voyage Foods products and resolve exact variants, prices and availability using the store's UCP/MCP catalog tools, with an anonymous GraphQL fallback.
api: mcp/voyage-foods-mcp.yml
endpoint: https://voyagefoods.com/api/ucp/mcp
operations: [search_catalog, lookup_catalog, get_product]
generated: '2026-08-05'
method: generated
grounded_in:
  - mcp/voyage-foods-ucp-tools-list.json
  - graphql/voyage-foods-storefront.graphql
---

# Discover the Voyage Foods catalog

Voyage Foods sells cocoa-free chocolate, bean-free coffee and nut-free spreads. The public
storefront catalog is a **sample and bulk ordering catalog for CPG buyers** — sample boxes and
bulk requests — not the retail grocery SKUs. Do not promise a buyer a retail product you did not
see in the catalog.

## Before you call anything

Every tool on this endpoint requires an agent identity. Send `meta["ucp-agent"].profile` on
**every** call — a dereferenceable HTTPS URI for your agent profile. The server fetches it. If you
omit it you get JSON-RPC `-32001` / `invalid_profile_url`; if it is unreachable you get `-32001` /
`profile_unreachable`. Tool invocation also requires a bearer JWT
(see https://shopify.dev/docs/agents/get-started/authentication) — a `-32000`
`AuthenticationRequired` means you have identity but no token.

Note the transport quirk: **errors come back with HTTP 200**. Branch on the JSON-RPC `error`
member, never on the status code.

## Steps

1. **Search.** Call `search_catalog` with `catalog.query` (natural language), or
   `catalog.filters`, or both — at least one is required. Pass `catalog.context.address_country`
   and `catalog.context.currency`; the store asks for these explicitly because pricing and
   availability depend on them. Filters available: `categories` (OR logic), `price.min` /
   `price.max` **in minor currency units**, and `available` (defaults to `true`, i.e. sale-ready
   items only). Paginate with `catalog.pagination.cursor` and `catalog.pagination.limit`.

2. **Resolve identifiers in bulk.** When you already hold several IDs, call `lookup_catalog`
   rather than looping `get_product`. It accepts `gid://shopify/Product/...` and
   `gid://shopify/ProductVariant/...` in one request.

3. **Get exact detail before quoting a price.** Call `get_product` for the chosen item. It returns
   a single product with its relevant variants, exact pricing and real-time availability, and
   supports `selected` / `preferences` for narrowing options. Only a **ProductVariant** GID can be
   added to a cart — a Product GID cannot.

4. **Back off politely.** The endpoint is rate-limited per IP. On `429`, back off before retrying.

## Anonymous fallback

If you cannot obtain a JWT, the store still exposes read-only surfaces with no credential at all:

- Storefront GraphQL at `https://voyagefoods.com/api/2026-04/graphql.json` — `products`,
  `productByHandle`, `collection`, `search`, `predictiveSearch`. Introspection is open, so you can
  validate a query before sending it. Watch `extensions.cost.requestedQueryCost` on each response.
- Plain JSON at `/products.json`, `/products/{handle}.json`,
  `/collections/{handle}/products.json`, and `/search?q={query}&type=product`.

## Hand off to a human when stuck

Every UCP error carries `data.continue_url`. When you cannot proceed, give the buyer that URL
rather than retrying blindly.
