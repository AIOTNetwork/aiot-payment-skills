# AIOT Payment Skills

OpenClaw skills for the AIOT Network payment platform. These skills enable AI agents to manage accounts, complete KYC verification, create virtual cards, send payments, handle crypto, and manage decentralized identities.

## Skills

| Skill | Slug | Description |
|-------|------|-------------|
| [Account & Authentication](account-auth/) | `aiotnetwork-account-auth` | Signup, login, OTP, token refresh, password reset, wallet linking |
| [KYC & Identity](kyc-identity/) | `aiotnetwork-kyc-identity` | Identity verification via MasterPay Global, document upload |
| [Card Management](card-management/) | `aiotnetwork-card-management` | Virtual card creation, lock/unlock/cancel, card details |
| [Payments & Banking](payments-banking/) | `aiotnetwork-payments-banking` | Wallet top-up, transfers, remittances, currency conversion |
| [Crypto Wallet](crypto-wallet/) | `aiotnetwork-crypto-wallet` | Crypto deposit addresses, withdrawals |
| [Blockchain & DID](blockchain-did/) | `aiotnetwork-blockchain-did` | Decentralized identity, on-chain KYC, membership tiers |
| [x402 Merchant Authentication](x402-merchant-auth/) | `aiotnetwork-x402-merchant-auth` | Merchant registration, login, OTP, token refresh, session management |
| [x402 Merchant Payments](x402-merchant-payments/) | `aiotnetwork-x402-merchant-payments` | x402 payment pre-request, off-chain verify, on-chain settle, facilitator capabilities |
| [AIOT Network](aiotnetwork/) | `aiotnetwork` | Meta-skill that routes requests to the correct sub-skill |

## Installation

### Install all skills at once

```bash
bash aiotnetwork/scripts/install.sh
```

### Install individual skills via ClawHub

```bash
clawhub install aiotnetwork-account-auth
clawhub install aiotnetwork-kyc-identity
clawhub install aiotnetwork-card-management
clawhub install aiotnetwork-payments-banking
clawhub install aiotnetwork-crypto-wallet
clawhub install aiotnetwork-blockchain-did
clawhub install aiotnetwork-x402-merchant-auth
clawhub install aiotnetwork-x402-merchant-payments
```

### Install the meta-skill (includes routing index)

```bash
clawhub install aiotnetwork
```

## Configuration

All skills use the AIOT Payment API. Set the base URL via environment variable:

```bash
export AIOT_API_BASE_URL="https://payment-api-dev.aiotnetwork.io"
```

If `AIOT_API_BASE_URL` is not set, skills default to `https://payment-api-dev.aiotnetwork.io`.

## Skill Dependencies

```
Account & Auth ─┬─> KYC & Identity ──> Card Management
                ├─> Payments & Banking
                ├─> Crypto Wallet
                └─> Blockchain & DID

x402 Merchant Auth ──> x402 Merchant Payments
```

- **KYC must be approved** before cards can be created
- **Wallet KYC** must be submitted after KYC approval, before card creation
- **Authentication** is required for all operations except signup and login

## Development

Skills are generated from the [AIOT Payment Backend](https://github.com/AIOTNetwork/AIOTPaymentBackend) source:

```bash
# In the backend repo
go run ./cmd/generate-skills    # Regenerate SKILL.md files
go test ./internal/skills/...   # Run tests
bash scripts/publish-skills.sh 1.0.1  # Publish to ClawHub
```

## License

Copyright AIOT Network. All rights reserved.
