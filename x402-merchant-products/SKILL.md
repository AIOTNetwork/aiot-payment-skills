---
name: x402 Merchant Products
description: Create and manage x402 merchant products, rotate API keys, view payment dashboards, and list settlement history.
version: 1.0.0
metadata:
  openclaw:
    requires:
      env:
        - AIOT_API_BASE_URL
    primaryEnv: AIOT_API_BASE_URL
---

# x402 Merchant Products

Use this skill when a merchant needs to create products, manage API keys, view payment dashboards, or browse settlement history.

All product endpoints require a Bearer JWT obtained from the x402 Merchant Auth skill.

## Configuration

The default API base URL is `https://payment-api-dev.aiotnetwork.io`. All endpoints are relative to this URL.

To override (e.g. for local development):

```bash
export AIOT_API_BASE_URL="http://localhost:8080"
```

If `AIOT_API_BASE_URL` is not set, use `https://payment-api-dev.aiotnetwork.io` as the base for all requests.

## Available Tools

- `create_product` — Create a new product and receive an API key | `POST /api/v1/x402/products` | Requires auth
- `list_products` — List all products for the current merchant | `GET /api/v1/x402/products` | Requires auth
- `get_product` — Get details of a specific product | `GET /api/v1/x402/products/:id` | Requires auth
- `get_product_dashboard` — Get payment stats and analytics for a product | `GET /api/v1/x402/products/:id/dashboard` | Requires auth
- `rotate_api_key` — Rotate a product's API key | `POST /api/v1/x402/products/:id/api-key/rotate` | Requires auth
- `list_settlements` — List settlement records for a product | `GET /api/v1/x402/products/:id/settlements` | Requires auth

## Recommended Flows

### Set Up a New Product

1. Authenticate via x402 Merchant Auth skill (login or register)
2. Create product: POST /api/v1/x402/products with {name, pay_to_address} — returns {product, api_key, proxy_base_url}. Save the api_key immediately — it is shown only once.
3. Use the api_key as X-API-Key header in x402 Merchant Payments skill

### Monitor Payments

1. View dashboard: GET /api/v1/x402/products/:id/dashboard with optional query params start_date, end_date — returns {merchant, stats: {token_stats, total_tx_count, failed_count}}
2. Browse settlements: GET /api/v1/x402/products/:id/settlements with optional query params limit, offset, search, start_date, end_date — returns {settlements, total}

### Rotate a Compromised Key

1. Rotate: POST /api/v1/x402/products/:id/api-key/rotate — returns {api_key, api_key_prefix}. Save the new api_key immediately.
2. Update all integrations with the new key. The old key is immediately invalid.

## Rules

- All endpoints require a Bearer JWT from the x402 Merchant Auth skill
- API keys are shown only once on create and rotate — they cannot be retrieved later
- pay_to_address must be a valid Ethereum address (0x + 40 hex characters)
- create_product requires: name, pay_to_address
- get_product_dashboard accepts optional query params: start_date, end_date
- list_settlements accepts optional query params: limit (default 20, max 100), offset, search, start_date, end_date
- Rotating an API key immediately invalidates the old key

## Agent Guidance

Follow these instructions when executing this skill:

- Always follow the documented flow order. Do not skip steps.
- If a tool requires authentication, verify the session has a valid Bearer token before calling it.
- Never expose, log, or persist secrets (passwords, tokens, API keys).
- If the user requests an operation outside this skill's scope, decline and suggest the appropriate skill.
- If a step fails, check the error and follow the recovery guidance below before retrying.

- When creating a product, always remind the merchant to save the api_key — it is shown only once and cannot be retrieved later.
- When rotating an API key, warn the merchant that the old key will be immediately invalidated and all integrations using it will stop working.
- Use `list_products` to help the merchant find a product ID before calling product-specific endpoints.
- For dashboard queries, suggest date ranges (start_date, end_date) to help the merchant narrow results.
- For settlement lists, use search to find specific transactions by tx hash, payer address, or order ID.
- The api_key returned by `create_product` or `rotate_api_key` is needed as the X-API-Key header for the x402 Merchant Payments skill.
- If the merchant needs to use the API key for payments, direct them to the x402 Merchant Payments skill.
