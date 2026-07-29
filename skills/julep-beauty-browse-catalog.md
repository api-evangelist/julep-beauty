---
name: julep-browse-catalog
description: Browse the Julep beauty catalog over the public storefront JSON API — list products, page a collection, fetch one product, search — with no authentication.
api: Julep Storefront Read-Only JSON API
base_url: https://www.julep.com
operations:
- listProducts
- listCollectionProducts
- getProduct
- searchStorefront
- getCart
generated: '2026-07-19'
method: generated
source: openapi/julep-beauty-storefront-openapi.yml
---

# Browse the Julep catalog

Julep is a direct-to-consumer beauty brand (makeup, skincare, nail) running on Shopify.
Its public catalog is readable as JSON with no credentials. The provider documents this
surface itself at <https://www.julep.com/agents.md> under "Read-Only Browsing (No
Authentication Required)".

Use this skill for discovery, price checks, availability, and product research. Do **not**
use it to transact — see the checkout rule at the bottom.

## Before you start

- Base URL: `https://www.julep.com`
- No API key, no token, no header. Anonymous `GET` only.
- Send a real `User-Agent`. Requests are rate limited per IP.

## Steps

### 1. List products — `listProducts`

```
GET /products.json?limit=50&page=1
```

Returns `{ "products": [ ... ] }`. Page by incrementing `page` until the array comes back
empty — there is no total-count field, no cursor, and no `Link` header.

Each product carries `id`, `title`, `handle`, `body_html`, `vendor`, `product_type`,
`tags[]`, `variants[]`, `images[]`, and `options[]`.

### 2. Narrow to a collection — `listCollectionProducts`

```
GET /collections/{handle}/products.json?limit=50
```

`all` is a valid handle for the full catalog. Other handles come from
`/sitemap_collections_1.xml` via the sitemap index at `/sitemap.xml`.

### 3. Fetch one product — `getProduct`

```
GET /products/{handle}.json
```

Returns `{ "product": { ... } }`. Read price and availability off `variants[]`, not off
the product: `price` and `compare_at_price` are strings, and `available` is the boolean
that tells you whether it can actually be bought. `sku` and the `option1..3` fields
identify the specific variant.

### 4. Search — `searchStorefront`

```
GET /search?q={query}&type=product
```

This one returns **HTML**, not JSON. Prefer filtering the JSON listings when you can;
fall back to search only for free-text intent.

### 5. Inspect the session cart — `getCart`

```
GET /cart.js
```

Returns the caller's cart: `token`, `item_count`, `items[]`, `total_price`,
`items_subtotal_price`, `currency`, `discount_codes[]`. Monetary values are integers in
the currency's minor unit.

## Buyer context

Julep serves 60+ country/region selectors across USD, CAD, and GBP. Pricing and
availability are region-dependent, so carry the buyer's country and currency through your
reasoning rather than assuming US/USD.

## Errors and rate limiting

There is no RFC 9457 problem document here — failures come back as bare HTTP statuses with
an HTML body.

| Status | Meaning | What to do |
|---|---|---|
| 404 | Unknown product or collection handle | Re-resolve the handle from the sitemap |
| 429 | Per-IP rate limit exceeded | Honor `Retry-After` (60s observed), then back off exponentially |
| 503 | Storefront renderer temporarily unavailable | Retry with backoff; transient in practice |

The provider's own instruction is explicit: *"The MCP endpoint is rate-limited per IP.
Back off on 429 responses."* Treat the storefront JSON the same way — 429 and 503 were
both observed during normal probing.

There are no idempotency keys and no request-id header. Every operation in this skill is a
`GET`, so retries are safe on that basis alone.

## Do not transact

Checkout, payment, and order placement are out of scope for this skill. Julep's
`robots.txt` and `agents.md` both state that agents must not complete payment without
explicit, contemporaneous human approval. If the user wants to buy, route them to the
UCP/MCP commerce endpoint (`mcp/julep-beauty-mcp.yml`), which requires a UCP agent profile
URI, or to the cross-store Shop skill at <https://shop.app/SKILL.md>.
