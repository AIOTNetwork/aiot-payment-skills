---
name: x402 Merchant Authentication
description: Merchant registration, login, OTP verification, token refresh, and session management for the x402 payment protocol.
version: 1.0.0
metadata:
  openclaw:
    requires:
      env:
        - AIOT_API_BASE_URL
    primaryEnv: AIOT_API_BASE_URL
---

# x402 Merchant Authentication

Use this skill when the user needs to register as an x402 merchant, log in, manage merchant sessions, or complete OTP verification for merchant registration.

## Configuration

The default API base URL is `https://payment-api-dev.aiotnetwork.io`. All endpoints are relative to this URL.

To override (e.g. for local development):

```bash
export AIOT_API_BASE_URL="http://localhost:8080"
```

If `AIOT_API_BASE_URL` is not set, use `https://payment-api-dev.aiotnetwork.io` as the base for all requests.

## Available Tools

- `merchant_send_otp` — Send a one-time password to a merchant email address | `POST /api/v1/x402/auth/otp/send`
- `merchant_verify_otp` — Verify an OTP code and receive a verification token (expires in 5 minutes) | `POST /api/v1/x402/auth/otp/verify`
- `merchant_register` — Register a new merchant account with first product and API key | `POST /api/v1/x402/auth/register`
- `merchant_login` — Login with email and password, receive JWT access and refresh tokens | `POST /api/v1/x402/auth/login`
- `merchant_refresh_token` — Refresh an expired access token using a refresh token | `POST /api/v1/x402/auth/refresh`
- `merchant_logout` — Invalidate all sessions for the merchant account | `POST /api/v1/x402/auth/logout` | Requires auth

## Recommended Flows

### Register a Merchant

1. Send OTP: POST /api/v1/x402/auth/otp/send with {email}
2. Verify OTP: POST /api/v1/x402/auth/otp/verify with {email, code} — returns verificationToken (expires in 5 minutes)
3. Register: POST /api/v1/x402/auth/register with {name, email, password, productName, payToAddress, verificationToken} — returns merchantAccount, tokens, and firstProduct (including apiKey shown once and proxyBaseUrl)

Important: The API key in the registration response is shown only once — save it immediately. The payToAddress must be a valid Ethereum address (0x followed by 40 hex characters). Password must be 8-128 characters.

### Login

1. Login: POST /api/v1/x402/auth/login with {email, password} — returns merchantAccount {id, name, email, status} and tokens {accessToken, refreshToken}
2. Use accessToken as Bearer token in Authorization header for authenticated requests
3. When accessToken expires (1 hour), refresh: POST /api/v1/x402/auth/refresh with {refreshToken}

## Rules

- OTP is required for merchant registration — always send then verify before registering
- Verification tokens expire after 5 minutes and can only be used once
- Access tokens expire after 1 hour — use refreshToken to get a new one
- Refresh tokens expire after 7 days
- Registration creates the merchant account, first product, and API key in one atomic operation
- The API key is returned only during registration — it cannot be retrieved again
- payToAddress must be a valid Ethereum address (0x + 40 hex characters)
- Password must be 8-128 characters
- Logout invalidates all sessions for the merchant

## Agent Guidance

Follow these instructions when executing this skill:

- Always follow the documented flow order. Do not skip steps.
- If a tool requires authentication, verify the session has a valid bearer token before calling it.
- If a tool requires a transaction PIN, ask the user for it fresh each time. Never cache or log PINs.
- Never expose, log, or persist secrets (passwords, tokens, full card numbers, CVVs).
- If the user requests an operation outside this skill's scope, decline and suggest the appropriate skill.
- If a step fails, check the error and follow the recovery guidance below before retrying.

- To register a new merchant: first call `merchant_send_otp`, then `merchant_verify_otp`, then `merchant_register` with the verification token. Never skip OTP verification.
- Registration requires: name, email, password (8-128 chars), productName, payToAddress (0x + 40 hex), verificationToken.
- The API key in the registration response (firstProduct.apiKey) is shown only once. Instruct the user to save it immediately. This key is needed for the x402-merchant-payments skill (X-API-Key header).
- The proxyBaseUrl in the registration response is the base URL for x402 proxy endpoints.
- When the access token expires, call `merchant_refresh_token` with the refresh token. Do not ask the user to log in again.
- Logout invalidates all active sessions. The user must log in again after logout.
- OTP send errors: COOLDOWN_ACTIVE (wait before resending), RATE_LIMIT_EXCEEDED (too many requests), EMAIL_SUPPRESSED (email cannot receive messages — use a different address).
- Never log, store, or repeat the user's password or API key back to them.
