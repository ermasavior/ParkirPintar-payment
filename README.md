# payment-service

[![Golang CI/CD](https://github.com/ermasavior/parkirpintar-payment/actions/workflows/cicd.yml/badge.svg)](https://github.com/ermasavior/parkirpintar-payment/actions/workflows/cicd.yml)

Manages the QRIS payment lifecycle. Bridges internal services with the external payment gateway via gRPC (outbound) and HTTP webhook (inbound).

## Responsibilities

- `CreatePayment` — registers a payment with the gateway, returns a QRIS code URL immediately. Idempotent via `idempotency_key`.
- `GetPaymentStatus` — returns current payment state
- `POST /webhook/payment/callback` — receives async payment results from the gateway, verifies HMAC-SHA256 signature, updates payment status, publishes to NATS JetStream

## Ports

| Port | Protocol | Purpose |
|---|---|---|
| 8086 | gRPC | Internal API (`CreatePayment`, `GetPaymentStatus`) |
| 8087 | HTTP | Inbound webhook from payment gateway |

## NATS events published

| Subject | Trigger |
|---|---|
| `payment.booking.done` | Webhook callback for a `BOOKING_FEE` payment |
| `payment.parking.done` | Webhook callback for a `PARKING_FEE` payment |

Payload: `{ "reference_id": "<uuid>", "status": "SUCCESS|FAILED|EXPIRED" }`

## Webhook security

Callbacks must include `X-Webhook-Signature: <HMAC-SHA256 hex>` computed over the raw JSON body using the shared `WEBHOOK_SECRET`. Requests with invalid signatures are rejected with `401`.

## gRPC API

```
service PaymentService {
  rpc CreatePayment    (CreatePaymentRequest)    returns (CreatePaymentResponse);
  rpc GetPaymentStatus (GetPaymentStatusRequest) returns (GetPaymentStatusResponse);
}
```

Proto: [`proto/payment/v1/payment.proto`](proto/payment/v1/payment.proto)

## Dependencies

| Dependency | Purpose |
|---|---|
| PostgreSQL | Payment records |
| NATS JetStream | Publish `payment.*.done` events |
| Payment Gateway (HTTP) | Create QRIS codes, receive callbacks |

## Configuration

```bash
cp .env.example .env
```

Key variables: `POSTGRES_DSN`, `NATS_URL`, `WEBHOOK_PORT`, `WEBHOOK_SECRET`, `GATEWAY_STUB_URL`

## Development

```bash
make run              # run locally
make build            # compile binary → bin/payment
make test             # all tests
make test-unit        # unit tests only
make unit-test-coverage
make proto            # regenerate gRPC code from .proto
make mock             # regenerate mocks
```
