---
name: Read the Voyage Foods storefront with anonymous GraphQL
description: Browse products, collections, blog content and shop policies from the Voyage Foods Storefront GraphQL API without any credential, including how to validate a query against the open schema first.
api: graphql/voyage-foods-storefront.graphql
endpoint: https://voyagefoods.com/api/2026-04/graphql.json
operations: [products, productByHandle, product, collection, collections, search, predictiveSearch, productRecommendations, blog, articles, page, shop, localization]
generated: '2026-08-05'
method: generated
grounded_in:
  - graphql/voyage-foods-storefront.graphql
  - data-model/voyage-foods-data-model.yml
---

# Read the Voyage Foods storefront with anonymous GraphQL

Use this when you need **read-only** catalog or content data and do not want to obtain a JWT for
the UCP/MCP endpoint. This surface answers unauthenticated, including full introspection.

`POST https://voyagefoods.com/api/2026-04/graphql.json` with `Content-Type: application/json` and
a body of `{"query": "..."}`. No token header is required for public reads.

## Validate before you send

Introspection is open on this endpoint, and the captured SDL is at
`graphql/voyage-foods-storefront.graphql` (416 types, 35 query fields, 41 mutations). Check a
field exists before querying it — an unknown field returns `extensions.code: undefinedField` and
costs you a round trip.

## What is readable anonymously

| Need | Fields |
|---|---|
| Catalog listing | `products`, `collections`, `collection`, `collectionByHandle` |
| One product | `product`, `productByHandle`, `productRecommendations` |
| Search | `search`, `predictiveSearch` |
| Facet vocabulary | `productTags`, `productTypes` |
| Content | `blog`, `blogByHandle`, `blogs`, `article`, `articles`, `page`, `pageByHandle`, `pages`, `menu` |
| Store config | `shop`, `localization`, `locations`, `paymentSettings`, `publicApiVersions` |
| Cart state | `cart` (needs the cart GID, whose `?key=` is the credential) |

`customer` and order history are **not** anonymous — they need a customer access token from
`customerAccessTokenCreate` or the OIDC flow at `/.well-known/openid-configuration`.

## Pagination

Every list field is a Relay connection. Use `first` / `after` (or `last` / `before`) and read
`pageInfo { hasNextPage endCursor }`. Do not assume an offset parameter exists; there is none.

## Watch the cost meter

Every response carries `extensions.cost.requestedQueryCost`. This is the rate-limit signal on this
surface — there are no rate-limit headers. Keep selection sets tight; ask for the variants and
price fields you need rather than the whole `Product`.

## Errors

Field-level errors return **HTTP 200** with an `errors` array carrying `message`, `locations`,
`path` and `extensions.code`. An unsupported API version in the path returns HTTP 404 with
`extensions.code: NOT_FOUND` — `2026-04` was confirmed live on 2026-08-05; read
`publicApiVersions` to check the current set.

Mutations behave differently: domain failures come back inside `data` as typed `userErrors`
(`CartUserError`, `CustomerUserError`) on an otherwise successful response. Inspect them even when
there is no top-level `errors` array.

## Even simpler

If you only need a product list, the store also publishes plain JSON with no GraphQL at all:
`/products.json`, `/products/{handle}.json`, `/collections/{handle}/products.json`.
