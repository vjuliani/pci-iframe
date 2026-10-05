# PCI Checkout POC v7.4

POC of a Stripe-like PCI checkout using an opaque, short-lived, single-use token.

## Canonical architecture

### Checkout Session / token issuance

```text
Merchant Frontend
    -> Merchant Backend
    -> API Gateway /checkout-session
    -> Checkout Session
    -> Ory Hydra
       -> opaque ephemeral token
```

The Merchant Backend RS256 JWT is used only server-to-server as the RFC 7523 assertion. Hydra validates its public JWK and emits an opaque access token with `payment.execute` scope.

### Payment

```text
Merchant Frontend
    -> PCI Hosted Fields
    -> API Gateway /payment
       -> TOKEN Lambda Authorizer / Ephemeral Token Guard
          -> Hydra introspection
          -> DynamoDB ACTIVE -> USED
       -> Payment Checkout
```

**The payment request does not pass through Merchant Backend.** PAN/CVC stay inside PCI Hosted Fields; only a simulated `payment_method_token` is sent to the payment API.

## Security invariants

- ephemeral token is opaque and short-lived;
- only Token Guard may consume it;
- authorizer cache TTL is zero;
- DynamoDB conditional update provides atomic single-use enforcement;
- Token Guard retrieves the trusted transaction id from the ledger;
- PCI iframe sends `X-Transaction-Id` and Payment Checkout compares it with authorizer context;
- Payment Checkout has no Hydra/replay-store responsibility;
- Hydra revocation is defense-in-depth; DynamoDB is authoritative for replay prevention.

## Local ports

- Merchant Frontend: `http://localhost:8880`
- PCI Hosted Fields: `http://localhost:8881`
- Merchant Backend: `http://localhost:3002`
- LocalStack: `http://localhost:4566`
- Hydra public: `http://localhost:4444`
- Hydra admin: `http://localhost:4445`

## Run

```bash
export LOCALSTACK_AUTH_TOKEN='<your token>'
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
unset AWS_SESSION_TOKEN AWS_PROFILE AWS_DEFAULT_PROFILE

make up
make clean-tf
make deploy
make diagnose
make test-api
make demo-up
```

Open `http://localhost:8880`.

## Tests

`make test-api` validates the canonical API Gateway path directly.

`make test-local` creates the Checkout Session through Merchant Backend, then simulates the PCI iframe by submitting payment directly to the public API Gateway URL.

Before debugging an end-to-end authorization failure, run `make diagnose` to verify the deployed Gateway/Authorizer wiring.

See `docs/FLOWS.md`, `docs/PAYMENT-AUTHORIZATION.md`, and `docs/SECURITY.md`.

## v7.5 local authorization modes

For a deterministic local UI/demo, use:

```bash
make demo-up
# or
make test-local
```

This starts a development-only adapter on `http://localhost:8890`. It executes the **same** `EphemeralTokenGuard` core as the Lambda Authorizer; it does not duplicate token-consumption logic.

To exercise the intended AWS path through LocalStack API Gateway:

```bash
make demo-up-api
make test-api
```

`make test-api` reports the known LocalStack Authorizer skip as `WARN`; use `make test-api-strict` when you want that incompatibility to fail CI. Production architecture remains `PCI Hosted Fields -> API Gateway /payment -> TOKEN Lambda Authorizer -> Token Guard -> Payment Checkout`.
