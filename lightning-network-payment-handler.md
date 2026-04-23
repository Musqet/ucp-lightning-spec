<!--
   Copyright 2026 UCP Authors

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->

# Lightning Network Payment Handlers

* **Handler Family:** `network.lightning.*`
* **Version:** `2026-04-22` (Draft)
* **Target UCP Version:** `2026-01-11`
* **Status:** Community Draft — open for review
* **Authors:** [Musqet](https://musqet.com)

## Introduction

This specification defines how an autonomous buyer (the **Platform**, acting
on behalf of a shopper or agent) pays a **Business** using the Bitcoin
Lightning Network in a UCP Checkout. It is payment-provider-neutral: a
Business can implement it using any Lightning stack (self-hosted node,
managed service, custodial or non-custodial) as long as they satisfy one of
the profiles defined below.

The handler is split into three sibling profiles that differ only in **how
the Platform obtains a BOLT11 invoice** to pay:

| Handler ID | Profile | Best for |
|:-----------|:--------|:---------|
| `network.lightning.bolt12`      | BOLT12 offers (no HTTP surface, no shared secret)                      | Non-custodial Businesses whose Lightning node already supports BOLT12. Agents request invoices directly from the node over the Lightning peer-to-peer network. |
| `network.lightning.lnurl-pay`   | LNURL-pay / Lightning Address                                          | Businesses with existing LNURL-pay infrastructure (mobile wallet apps, Alby, BTCPayServer, etc.). Works with any Lightning node. Sats-only. |
| `network.lightning.invoice-api` | Provider-mediated HTTP API for invoice issuance and (optional) verification | Businesses that want fiat-denominated invoices with FX locking or provider-managed invoice lifecycle. |

All three profiles produce the **same payment credential**: a SHA-256
preimage proving HTLC settlement. A Business can advertise one or many
profiles in a single `payment.handlers` array; the Platform selects
whichever profile it can execute.

### Why Lightning

- **Settlement finality in seconds**, not days.
- **Low fees** for both small and large values.
- **Cryptographic proof of payment**: a 32-byte preimage that any party
  (shopper, agent, auditor) can verify against the payment hash. No acquirer
  dispute window.
- **Global reach**: no issuer geography, no chargeback risk.

### Out of Scope

- On-chain Bitcoin payments (a separate handler).
- Lightning *routing* (handled by the Lightning Network itself).
- Key management and custody models at the Business — that is the Business's
  choice and outside UCP. This spec does not require custody; it only requires
  that the Business (or a service acting on their behalf) can produce BOLT11
  invoices and observe settlement.

---

## Participants

| Participant | Role | Prerequisites |
|:------------|:-----|:--------------|
| **Business** | Advertises a Lightning handler in their UCP Checkout profile; verifies preimages before marking orders paid. | Access to a Lightning node that can (a) issue BOLT11 invoices and (b) observe HTLC settlement. |
| **Platform** | Discovers the handler, executes the profile-specific flow to obtain a BOLT11, pays it, captures the preimage, and returns it as the payment credential. | A Lightning wallet capable of paying BOLT11 invoices. |
| **Invoice Provider** *(invoice-api profile only)* | HTTP service that issues BOLT11 invoices on behalf of the Business and optionally verifies preimages. MAY be the Business itself or a third party. | Owns or has delegated access to the Business's Lightning node. |

> **Note on Terminology:** Throughout this document, "Business" is the UCP
> participant term. The Lightning ecosystem typically uses "merchant"; both
> refer to the same actor. Schema field names retain `merchant_*` where that
> matches common industry nomenclature (e.g., `merchant_id`).

### Flow (profile-agnostic)

```text
┌──────────┐         ┌──────────┐    ┌────────────────────────┐    ┌──────────────┐
│ Platform │         │ Business │    │ Invoice Source         │    │ LN Network   │
│ (Agent)  │         │          │    │ (node / LNURL / API)   │    │              │
└────┬─────┘         └────┬─────┘    └──────────┬─────────────┘    └──────┬───────┘
     │  1. Create checkout│                     │                         │
     │───────────────────>│                     │                         │
     │  checkout_id       │                     │                         │
     │<───────────────────│                     │                         │
     │                    │                     │                         │
     │  2. Acquire BOLT11 (per profile — see profile sections)            │
     │──────────────────────────────────────────>│                        │
     │  bolt11, payment_hash                                              │
     │<──────────────────────────────────────────│                        │
     │                                                                    │
     │  3. Pay invoice                                                    │
     │────────────────────────────────────────────────────────────────────>│
     │  preimage                                                          │
     │<────────────────────────────────────────────────────────────────────│
     │                    │                     │                         │
     │  4. Complete checkout with preimage credential                     │
     │───────────────────>│                     │                         │
     │                    │                     │                         │
     │                    │  5. Verify credential (see §Verification — Option A local or Option B /verify)
     │                    │─────────────────────────────────────────>     │
     │                    │                                               │
     │  payment_status    │                                               │
     │<───────────────────│                                               │
```

---

## Shared Concepts

### Payment Instrument

**Schema:** `https://ucp.dev/schemas/network/lightning/instrument.json`

```json
{
  "id": "ln_1a2b3c4d",
  "handler_id": "bolt12",
  "type": "network.lightning.preimage",
  "credential": { ... }
}
```

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `id` | string | Yes | Platform-assigned instrument identifier. |
| `handler_id` | string | Yes | Business-assigned handler instance id. MUST match the `id` of an entry in the Business's `payment.handlers[]` whose `name` is one of `network.lightning.bolt12`, `network.lightning.lnurl-pay`, or `network.lightning.invoice-api`. |
| `type` | const `network.lightning.preimage` | Yes | Credential type. Same across all profiles. |
| `credential` | object | Yes | See [Payment Credential](#payment-credential). |

### Payment Credential

**Schema:** `https://ucp.dev/schemas/network/lightning/credential.json`

```json
{
  "type": "network.lightning.preimage",
  "preimage": "c3a1...d9f2",
  "checkout_id": "chk_01HXYZ..."
}
```

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `type` | const `network.lightning.preimage` | Yes | Credential type. |
| `preimage` | 64-char hex | Yes | The 32-byte preimage revealed during HTLC settlement. |
| `checkout_id` | string | Yes | The UCP checkout this payment settles. Carried in-credential so binding data is covered by checkout integrity, not transport headers. |

The credential is intentionally minimal. The `payment_hash` is derived by
the Business as `SHA256(preimage)`; the BOLT11 the Platform paid is already
held by the Business (keyed by `payment_hash`), so re-submitting it would
add nothing a forger couldn't also provide.

### Verification

A Business verifying a preimage credential MUST perform all of the
following, in order:

1. **Compute and locate** — compute `payment_hash := SHA256(preimage)`,
   then look up `payment_hash` in the Business's own invoice index. If no
   record exists, reject with `invoice_not_found`.
2. **Binding** — the invoice record's `checkout_id` (bound at
   invoice-creation time) MUST equal `credential.checkout_id`. This is
   the check that prevents a preimage — which is revealed to every
   routing hop at settlement and is therefore not a Platform-exclusive
   secret — from being replayed against an unrelated open checkout of
   the same Business. Per profile, the bound `checkout_id` is recovered
   from:
   - **BOLT12:** the `payer_note` recorded on the settled invoice by the
     Business's node.
   - **LNURL-pay:** the LUD-12 `comment` recorded by the Business's LNURL
     service, or — if comments are unavailable — the `checkout_id`
     embedded in the per-checkout dynamic URL.
   - **Invoice API:** the `checkout_id` persisted by the Invoice Provider
     at invoice creation.
3. **Settlement** — the Business's Lightning node MUST report `payment_hash`
   as *settled* (not merely issued). Observed locally, never trusted from
   the Platform's claim.
4. **Amount match** — the sats amount locked into the invoice at issuance
   MUST equal the Business's expected sats for this `checkout_id`. For
   fiat-denominated orders, expected sats is the FX-locked amount
   computed at invoice creation.

A Business that does not operate its own Lightning infrastructure MAY
delegate any of these checks to its Invoice Provider via the `invoice-api`
profile's `/verify` endpoint.

Reuse of a single `(preimage, checkout_id)` pair is handled by the UCP
checkout layer's own idempotency, not repeated here: a second completion
attempt for an already-paid checkout returns the previous result.

---

## Profile: BOLT12 (`network.lightning.bolt12`)

### When to use

A Business whose Lightning node supports BOLT12 offers (e.g., Core
Lightning with `experimental-offers`, or LND with a BOLT12 plugin). No HTTP
endpoint is required on the Business side; everything happens over the
Lightning peer-to-peer network.

### Handler Configuration

**Schema:** `https://ucp.dev/schemas/network/lightning/bolt12.config.json`

```json
{
  "payment": {
    "handlers": [
      {
        "id": "bolt12",
        "name": "network.lightning.bolt12",
        "version": "2026-04-22",
        "spec": "https://ucp.dev/specification/lightning-network-payment-handler/",
        "config_schema": "https://ucp.dev/schemas/network/lightning/bolt12.config.json",
        "instrument_schemas": [
          "https://ucp.dev/schemas/network/lightning/instrument.json"
        ],
        "config": {
          "offer": "lno1pgx9getnwss8vetrw3hhyuckyypnx7u30fshxcmjd9c8g7tsv9e8g",
          "display_name": "Pay with Lightning (BOLT12)",
          "supported_currencies": ["SAT"]
        }
      }
    ]
  }
}
```

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `offer` | BOLT12 offer string (`lno1...`) | Yes | The Business's static BOLT12 offer. The offer MAY be amountless (Platform specifies amount in `invoice_request`) or amount-fixed. |
| `display_name` | string | No | Human-readable label for the Platform's payment-method UI. |
| `supported_currencies` | array of ISO-4217 or `SAT` | No | If the offer is denominated in fiat (`iso4217` in BOLT12), list the supported codes here. Defaults to `["SAT"]`. |

### Platform Integration

#### Prerequisites

1. A Lightning wallet that can send BOLT12 `invoice_request` messages.
2. A BOLT11-capable payment path (BOLT12 invoices are still BOLT11 under the
   hood at settlement time; Platforms pay them like any other invoice).

#### Step 1: Decode the Offer

The Platform decodes the offer in `config.offer`. Relevant fields include
the issuer node id and any amount / currency constraints.

#### Step 2: Send `invoice_request`

The Platform sends a BOLT12 `invoice_request` to the issuer node, including:

- `payer_note`: MUST be set to the UCP `checkout_id`. This is the binding
  mechanism — the Business reads `payer_note` from the settled invoice to
  correlate payment to order.
- `invoice_request_amount`: the amount to pay, if the offer is amountless.

#### Step 3: Receive BOLT12 Invoice & Pay

The issuer node returns a BOLT12 invoice. The Platform pays it and captures
the 32-byte preimage revealed during settlement.

#### Step 4: Submit Credential

The Platform completes the checkout with:

```json
POST /checkout-sessions/{checkout_id}/complete

{
  "payment_data": {
    "id": "ln_bolt12_01HXYZ",
    "handler_id": "bolt12",
    "type": "network.lightning.preimage",
    "credential": {
      "type": "network.lightning.preimage",
      "preimage": "c3a1...d9f2",
      "checkout_id": "chk_01HXYZ"
    }
  }
}
```

### Business Integration

1. Advertise the offer in `config.offer`.
2. On receiving a payment credential, compute `payment_hash = SHA256(preimage)`,
   look it up in the node's settled invoice index, read the recorded
   `payer_note`, and match it to `credential.checkout_id`.
3. Apply the four verification checks from [Verification](#verification).

### Limitations

- BOLT12 adoption is still partial; not every Lightning node supports it.
- Fiat-denominated offers rely on the Business's node performing FX at
  invoice-request time. There is no separate `fx_rate` field in the credential
  — the amount in the returned invoice is the final locked sats amount.

---

## Profile: LNURL-pay (`network.lightning.lnurl-pay`)

### When to use

A Business that already exposes an LNURL-pay endpoint or a Lightning
Address (the overwhelming majority of existing Lightning merchant setups).
HTTP-based, widely supported by wallets, but sats-only.

### Handler Configuration

**Schema:** `https://ucp.dev/schemas/network/lightning/lnurl-pay.config.json`

```json
{
  "payment": {
    "handlers": [
      {
        "id": "lnurl-pay",
        "name": "network.lightning.lnurl-pay",
        "version": "2026-04-22",
        "spec": "https://ucp.dev/specification/lightning-network-payment-handler/",
        "config_schema": "https://ucp.dev/schemas/network/lightning/lnurl-pay.config.json",
        "instrument_schemas": [
          "https://ucp.dev/schemas/network/lightning/instrument.json"
        ],
        "config": {
          "lightning_address": "alice@example.com",
          "display_name": "Pay with Lightning",
          "comment_supported": true
        }
      }
    ]
  }
}
```

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `lightning_address` | string | One of `lightning_address` or `lnurl` | Lightning Address (`user@host`). Resolves to `https://{host}/.well-known/lnurlp/{user}`. |
| `lnurl` | string (bech32 `lnurl1...` or `https://...`) | One of `lightning_address` or `lnurl` | Direct LNURL-pay endpoint. |
| `display_name` | string | No | Human-readable label. |
| `comment_supported` | boolean | No | Advertises whether the endpoint accepts a `comment` parameter (LUD-12). If `false`, Platforms MUST NOT rely on the `comment` field for `checkout_id` binding; see [Binding without comment support](#binding-without-comment-support). |

### Platform Integration

#### Step 1: Resolve LNURL

If `lightning_address` is used, fetch
`https://{host}/.well-known/lnurlp/{user}`. If `lnurl` is bech32-encoded,
decode to HTTPS URL per LUD-01.

Response (LUD-06):

```json
{
  "tag": "payRequest",
  "callback": "https://example.com/.well-known/lnurlp/alice/callback",
  "minSendable": 1000,
  "maxSendable": 100000000,
  "metadata": "[[\"text/plain\",\"Pay alice\"]]",
  "commentAllowed": 280
}
```

#### Step 2: Request BOLT11

```http
GET {callback}?amount={msats}&comment={checkout_id}
```

- `amount` MUST be within `[minSendable, maxSendable]` (millisats).
- `comment` MUST be the UCP `checkout_id`, if `comment_supported` is `true`
  and `commentAllowed >= len(checkout_id)`.

Response:

```json
{
  "pr": "lnbc10u1p3xnhl2...",
  "routes": []
}
```

#### Step 3: Pay and Capture Preimage

Platform pays `pr` and captures the preimage.

#### Step 4: Submit Credential

Same shape as [BOLT12 Step 4](#step-4-submit-credential), with
`handler_id: "lnurl-pay"`.

### Business Integration

1. Host an LNURL-pay endpoint (or a Lightning Address that proxies to one).
   Existing tools that provide this out-of-the-box: BTCPayServer,
   LNbits, Alby Hub, Phoenixd, Zeus.
2. On payment, look up `payment_hash` on the node; read the recorded
   `comment` to recover `checkout_id`; match to order.

#### Binding without comment support

If the LNURL endpoint does not support comments (or has a restrictive
`commentAllowed` length), the Business MUST use a per-checkout dynamic
LNURL (generate a fresh LNURL with the `checkout_id` embedded in the URL
path, e.g., `https://example.com/ln/chk_01HXYZ`). In that case, the
`config.lnurl` advertised in the UCP Checkout profile is the dynamic URL,
not a static Lightning Address.

### Limitations

- **Sats only.** LNURL-pay amounts are millisats. Fiat pricing must be
  converted to sats by the Business before advertising the UCP profile.
- **Binding depends on LUD-12 or dynamic URLs.** Static Lightning
  Addresses without comment support cannot safely carry `checkout_id`.

---

## Profile: Invoice API (`network.lightning.invoice-api`)

### When to use

A Business (or service acting on their behalf — the "Invoice Provider")
wants an HTTP API for invoice creation so that:

- Platforms do not need any Lightning-protocol tooling.
- Invoices can be priced in fiat, with the sats amount locked at
  issuance.
- The Provider can expose a `/verify` endpoint that the Business can use
  to confirm settlement without operating their own node integration.

This profile is the most full-featured and operationally complex.

### Handler Configuration

**Schema:** `https://ucp.dev/schemas/network/lightning/invoice-api.config.json`

```json
{
  "payment": {
    "handlers": [
      {
        "id": "invoice-api",
        "name": "network.lightning.invoice-api",
        "version": "2026-04-22",
        "spec": "https://ucp.dev/specification/lightning-network-payment-handler/",
        "config_schema": "https://ucp.dev/schemas/network/lightning/invoice-api.config.json",
        "instrument_schemas": [
          "https://ucp.dev/schemas/network/lightning/instrument.json"
        ],
        "config": {
          "merchant_id": "biz_abc123",
          "invoice_endpoint": "https://provider.example.com/ucp/lightning/invoices",
          "display_name": "Pay with Lightning",
          "environment": "production",
          "supported_currencies": ["SAT", "USD", "EUR", "GBP"]
        }
      }
    ]
  }
}
```

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `merchant_id` | string | Yes | Public Business identifier at the Invoice Provider. Maps to `PaymentIdentity.access_token`. NOT a secret. |
| `invoice_endpoint` | URI | Yes | Provider endpoint for BOLT11 invoice creation. Called by the Platform. |
| `display_name` | string | No | Human-readable label. |
| `environment` | enum `sandbox` \| `production` | No | Defaults to `production`. Sandbox semantics are Provider-defined. |
| `supported_currencies` | array of 3-char codes | No | Currencies the Provider accepts for this Business (e.g., `SAT`, `USD`, `EUR`). Omitted means `["SAT"]` only. |

Note: a `verify_endpoint` is **not** advertised in the UCP profile because
verification is a Business↔Provider concern, not agent-visible. The
Business learns the verify endpoint during onboarding with the Provider.

### Platform Integration

#### `POST {invoice_endpoint}`

Creates (or retrieves, if idempotent) a BOLT11 invoice.

**Request:**

```json
{
  "merchant_id": "biz_abc123",
  "checkout_id": "chk_01HXYZ",
  "currency": "USD",
  "amount": 2500,
  "description": "Order #4231",
  "expiry_seconds": 600
}
```

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `merchant_id` | string | Yes | MUST match the value advertised in the Business's UCP profile. |
| `checkout_id` | string | Yes | UCP checkout identifier. Idempotency key scoped per `merchant_id`. |
| `currency` | string, 3 chars | Yes | `SAT` for satoshis, or any ISO-4217 code. MUST be in the Provider's `supported_currencies` for this Business. |
| `amount` | integer, ≥ 1 | Yes | Minor units of `currency` (sats for `SAT`, cents for `USD`, etc.). |
| `description` | string | No | Human-readable invoice description. |
| `expiry_seconds` | integer, ≥ 1 | No | Invoice expiry. Defaults to 600 (10 minutes). |

**Response (201 Created or 200 OK on idempotent retry):**

```json
{
  "invoice_id": "inv_01HXYZ",
  "bolt11": "lnbc10u1p3xnhl2...",
  "payment_hash": "8d2a...7e1b",
  "currency": "USD",
  "amount": 2500,
  "amount_sats": 45230,
  "fx_rate": 18.092,
  "expires_at": "2026-04-22T12:34:56Z"
}
```

| Field | Type | Description |
|:------|:-----|:------------|
| `invoice_id` | string | Provider-issued ID. Distinct from `payment_hash`. |
| `bolt11` | string | The BOLT11 invoice to pay. |
| `payment_hash` | 64-char hex | Payment hash of the issued invoice. Informational for the Platform; the Business re-derives it from the credential's `preimage` during verification. |
| `currency`, `amount` | | Echo of the request. |
| `amount_sats` | integer | Lightning amount in sats, locked at issuance. |
| `fx_rate` | number | Sats per minor unit of `currency`. Omitted when `currency == "SAT"`. Locked at issuance — does not move with market. |
| `expires_at` | ISO-8601 | Invoice expiry. |

**Idempotency:** repeat requests with the same `(merchant_id, checkout_id)`
that also match `(currency, amount)` MUST return the same invoice. If
`(currency, amount)` differs from the existing invoice, the Provider MUST
return `409 amount_mismatch`.

**Authentication (Platform → Provider):** no per-agent authentication is
required. The `checkout_id` (opaque, unguessable, Business-issued) acts as
the capability token. Providers MUST rate-limit per
`(merchant_id, client-ip)` to mitigate abuse.

### Business Integration

#### Option A: Local verification

The Business operates the Lightning node itself and verifies preimages
locally using the four checks in [Verification](#verification). No call to
the Provider is needed.

#### Option B: Provider-mediated verification

The Business calls the Provider's `/verify` endpoint (private — discovered
during onboarding, authenticated with a Business API key) to confirm
settlement. This is useful when the Business does not operate the node
itself or when the Provider holds additional state such as FX lock and
settlement timestamps.

##### `POST {verify_endpoint}`

**Request:**

```json
{
  "preimage": "c3a1...d9f2",
  "checkout_id": "chk_01HXYZ"
}
```

**Response (200 OK):**

```json
{
  "valid": true,
  "settled": true,
  "invoice_id": "inv_01HXYZ",
  "payment_hash": "8d2a...7e1b",
  "currency": "USD",
  "amount": 2500,
  "amount_sats": 45230,
  "fx_rate": 18.092,
  "settled_at": "2026-04-22T12:33:01Z"
}
```

Binding: the Provider MUST reject the request with `403 binding_mismatch`
if the preimage's invoice is bound to a different `checkout_id`, or with
`404 invoice_not_found` if the preimage's invoice belongs to a different
Business than the authenticated caller.

### Limitations

- More operational surface than BOLT12 or LNURL.
- Requires Business↔Provider trust for FX lock and settlement reporting
  (Option B).

### Example Implementation

[Musqet](https://musqet.com) operates a UCP Invoice Provider at
`https://api.musqet.tech/ucp/lightning` supporting `SAT`, `USD`, `EUR`, and
`GBP`, including Option B (`/verify`) support.
Source lives under `apps/@api/src/routes/ucp/` in the Musqet monorepo and
can serve as a reference for other Providers building this profile.

---

## Security Considerations

| Requirement | Description |
|:------------|:------------|
| **TLS everywhere** | All HTTP surfaces (LNURL endpoint, Invoice API) MUST use TLS. |
| **Credential binding** | `checkout_id` MUST be carried in the `credential` payload, not in headers or query strings, so it is covered by checkout integrity. |
| **Issuance-time binding** | The Business (or Invoice Provider) MUST persist `(payment_hash → checkout_id)` at invoice-creation time. Verification rejects any credential whose recomputed `payment_hash` does not resolve to the submitted `checkout_id`. Per profile the bound value lives in `payer_note` (BOLT12), the LUD-12 `comment` or per-checkout dynamic URL (LNURL-pay), or the Provider's invoice record (Invoice API). |
| **Preimage is not Platform-exclusive** | A Lightning preimage is revealed to every routing hop at HTLC settlement, so any intermediary may learn it before the Platform submits the credential. Businesses MUST treat the preimage as a shared secret: it proves *an* HTLC settled, not *who* paid. Authorization of the payment comes from the issuance-time binding above, not from preimage possession. |
| **Settlement observed locally** | The Business MUST confirm settlement by consulting their own Lightning node (Option A) or their Invoice Provider (Option B). The Platform's claim of settlement is never sufficient on its own. |
| **Invoice expiry** | Platforms SHOULD pay invoices well before `expires_at`. Expired invoices produce `invoice_expired` on verification. |
| **Amount tampering (Platform-side)** | Because the Platform (agent) chooses the amount in the `invoice_request` / LNURL callback / Invoice API POST, the Business MUST re-check the paid sats against the expected amount for the order. Expected amount is the sats amount the Business (or its Provider, in `invoice-api`) locked at invoice creation. |
| **Rate limiting** | Providers and LNURL endpoints MUST rate-limit invoice creation per `(merchant_id, client-ip)` to mitigate abuse. |
| **Cross-tenant info leak (invoice-api)** | The `/verify` endpoint MUST NOT reveal whether a preimage belongs to another Business. Return `404 invoice_not_found` uniformly for non-owned and non-existent invoices. |

---

## Error Codes

Responses use JSON `{ "code": "...", "message": "..." }`. Handlers MAY
return additional fields. All HTTP surfaces in this spec (LNURL callback,
Invoice API, `/verify`) SHOULD use these codes where applicable.

| HTTP | Code | Meaning |
|:-----|:-----|:--------|
| 400  | `invalid_request` | Request failed schema validation or had invalid field values. |
| 400  | `unsupported_currency` | `currency` is not in the Business's `supported_currencies`. |
| 400  | `amount_out_of_range` | `amount` violates Provider or Business minimums/maximums. |
| 403  | `binding_mismatch` | Preimage is valid but bound to a different `checkout_id` or Business. |
| 404  | `merchant_not_found` | `merchant_id` unknown to the Provider. |
| 404  | `invoice_not_found` | No matching invoice for the preimage (includes cross-tenant hiding). |
| 409  | `amount_mismatch` | Idempotent retry with different `(currency, amount)` than existing invoice. |
| 409  | `lightning_not_enabled` | Business has not finished onboarding the underlying Lightning service. |
| 410  | `invoice_expired` | Invoice has expired without being paid. |
| 429  | `rate_limited` | Rate limit exceeded. |
| 503  | `provider_unavailable` | Upstream Lightning or Provider temporarily unavailable. Recoverable. |

---

## AP2 Compatibility

This handler is compatible with the `dev.ucp.shopping.ap2_mandate` extension.
The mandate's `payment_method` object MAY reference any of the three handler
IDs. The mandate binds the Platform to a single `checkout_id`; the Lightning
credential the Platform submits MUST carry the same `checkout_id`.

No additional Lightning-specific fields are required in the mandate.

---

## Test Vectors

### Preimage ↔ Payment Hash

```
preimage     = 0000000000000000000000000000000000000000000000000000000000000001
payment_hash = 4bf5122f344554c53bde2ebb8cd2b7e3d1600ad631c385a5d7cce23c7785459a
```

(`SHA256(\x00...\x01)` over the 32 raw bytes.)

---

## Governance & Licensing

This spec was authored by [Musqet](https://musqet.tech) and is open for
community review and contribution. The namespace `network.lightning.*` is
intentionally unowned — no single organization, including Musqet, controls
it. The spec is published under
[Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0).

To propose changes, open an issue or PR against the UCP repository where
this file lives.

---

## References

- **BOLT11:** [Invoice protocol](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md)
- **BOLT12:** [Offers](https://github.com/lightning/bolts/blob/master/12-offer-encoding.md)
- **LNURL-pay (LUD-06):** [Payment request](https://github.com/lnurl/luds/blob/luds/06.md)
- **Lightning Address:** [lightningaddress.com](https://lightningaddress.com/)
- **LUD-12:** [Comments in payRequest](https://github.com/lnurl/luds/blob/luds/12.md)
- **UCP Payment Handler Guide:** [ucp.dev/specification/payment-handler-guide](https://ucp.dev/specification/payment-handler-guide/)
- **UCP AP2 Mandates:** [ucp.dev/specification/ap2-mandates](https://ucp.dev/specification/ap2-mandates/)

---

## Changelog

| Version | Changes |
|:--------|:--------|
| 2026-04-22 | Initial community draft. Multi-profile handler family covering BOLT12, LNURL-pay, and Invoice API. Neutral namespace `network.lightning.*`. Shared preimage credential across all profiles. |
