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

* **Handler Family:** `com.musqet.*`
* **Version:** `2026-04-23`
* **Target UCP Version:** `2026-04-08`
* **Authors:** [Musqet](https://musqet.com)

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY" in this
document are to be interpreted as described in [BCP 14](https://www.rfc-editor.org/info/bcp14)
([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119), [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)).

## Introduction

This specification defines how a **Platform** (autonomous buyer) pays a
**Business** via the Bitcoin Lightning Network in a UCP Checkout. It is
payment-provider-neutral: any Lightning stack works as long as the Business
satisfies one of the profiles below.

Three sibling profiles differ only in **how the Platform obtains a BOLT11
invoice**:

| Handler ID | Profile | Best for |
|:-----------|:--------|:---------|
| `com.musqet.bolt12`      | BOLT12 offers | Non-custodial nodes with BOLT12 support. No HTTP surface needed. |
| `com.musqet.lnurl-pay`   | LNURL-pay / Lightning Address | Existing LNURL-pay setups (BTCPayServer, Alby, etc.). Sats-only. |
| `com.musqet.invoice-api` | Provider-mediated HTTP API | Fiat-denominated invoices with FX locking, or provider-managed lifecycle. |

All three produce the **same credential**: a SHA-256 preimage proving HTLC
settlement. A Business can advertise multiple profiles; the Platform selects
whichever it can execute.

> **Terminology:** "Business" is UCP's term for what Lightning calls
> "merchant". Schema fields retain `merchant_*` where that matches existing
> industry nomenclature.

On-chain Bitcoin payments, Lightning routing, and Business key
management / custody models are out of scope.

---

## Participants

| Participant | Role |
|:------------|:-----|
| **Business** | Advertises a Lightning handler in their UCP Checkout profile; verifies preimages before marking orders paid. Requires a Lightning node that can issue BOLT11 invoices and observe settlement. |
| **Platform** | Discovers the handler, obtains a BOLT11 via the profile-specific flow, pays it, and returns the preimage as credential. Requires a Lightning wallet capable of paying BOLT11 invoices. |
| **Invoice Provider** *(invoice-api only)* | HTTP service that issues BOLT11 invoices on behalf of the Business and optionally verifies preimages. MAY be the Business itself or a third party. |

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
     │                    │  5. Verify (§Verification — Option A or B)    │
     │                    │─────────────────────────────────────────>     │
     │                    │                                               │
     │  completion response│                                               │
     │<───────────────────│                                               │
```

---

## Shared Concepts

### Handler Declaration

Businesses advertise payment handlers in their UCP profile at
`/.well-known/ucp`. In the `payment_handlers` object, the **family key**
(e.g., `com.musqet.bolt12`) identifies the handler type. Each handler
instance beneath it has a Business-assigned **instance `id`** that can be
any string. When a Business has a single instance per family, the `id` often
mirrors the family key, but a Business with multiple instances (e.g., two
Invoice API Providers) MUST use distinct ids.

Full example (BOLT12):

```json
{
  "payment_handlers": {
    "com.musqet.bolt12": [
      {
        "id": "bolt12_main",
        "version": "2026-04-23",
        "spec": "https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/tags/v2026-04-23/lightning-network-payment-handler.md",
        "schema": "https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/tags/v2026-04-23/lightning/bolt12.config.json",
        "available_instruments": [
          { "type": "com.musqet.preimage" }
        ],
        "config": { ... }
      }
    ]
  }
}
```

The `version`, `spec`, `schema`, and `available_instruments` fields follow
the same pattern for all three profiles — only `schema` and `config` differ.
Profile sections below show only the `config` object.

> **No `platform_config` or `response_config`.** None of the three profiles
> require Platform-side onboarding or return additional runtime state beyond
> the static handler config. There are no separate `platform_config` or
> `response_config` schemas.

### Payment Instrument

**Schema:** [`lightning/instrument.json`](https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/tags/v2026-04-23/lightning/instrument.json)
— extends UCP `payment_instrument.json` via `allOf`. This spec introduces
the instrument type `com.musqet.preimage`, shared across all three profiles.

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `id` | string | Yes | Platform-assigned instrument identifier. |
| `handler_id` | string | Yes | MUST match the `id` of a handler instance under one of the `com.musqet.*` families in the Business's `payment_handlers`. |
| `type` | const `com.musqet.preimage` | Yes | Credential type. Same across all profiles. |
| `credential` | object | Yes | See [Payment Credential](#payment-credential). |
| `billing_address` | object | No | Not used for payment verification (no AVS). MAY be provided for order fulfillment. |
| `display` | object | No | Optional display metadata (`label`, `icon_url`) for Platform UIs. |

### Payment Credential

**Schema:** [`lightning/credential.json`](https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/tags/v2026-04-23/lightning/credential.json)
— extends UCP `payment_credential.json` via `allOf`.

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `type` | const `com.musqet.preimage` | Yes | Credential type. |
| `preimage` | 64-char lowercase hex | Yes | 32-byte preimage from HTLC settlement. MUST be lowercase (`[0-9a-f]{64}`). |
| `checkout_id` | string | Yes | UCP checkout this payment settles. Carried in-credential so binding is covered by checkout integrity. |

The credential is intentionally minimal. `payment_hash` is derived as
`SHA256(preimage)`; the BOLT11 is already held by the Business, so
re-submitting it adds nothing a forger couldn't also provide.

> **Credential expiry exemption.** A Lightning preimage is a one-time proof
> of a specific HTLC settlement, not a reusable token. Its validity is
> bounded by the BOLT11 invoice's `expires_at`: once expired unpaid, no
> preimage can be produced; once paid, the preimage is permanently valid.
> No separate `expiry` field is included. Implementations needing
> time-bounded acceptance SHOULD enforce the invoice's `expires_at` during
> verification.

### Verification

A Business verifying a preimage credential MUST perform all of the
following, in order:

1. **Compute and locate** — `payment_hash := SHA256(preimage)`, look up in
   the Business's invoice index. If absent, reject with `invoice_not_found`.
2. **Binding** — the invoice record's `checkout_id` MUST equal
   `credential.checkout_id`. Per profile, the bound value lives in:
   - **BOLT12:** `payer_note` on the settled invoice.
   - **LNURL-pay:** LUD-12 `comment` recorded by the LNURL service.
   - **Invoice API:** Provider's invoice record.
3. **Settlement** — the Business's Lightning node MUST report `payment_hash`
   as *settled*. Never trusted from the Platform's claim alone.
4. **Amount match** — settled sats MUST exactly equal the sats amount
   locked into the invoice at issuance. For fiat-denominated orders, this
   is the FX-converted amount set at invoice creation. Neither over- nor
   under-payment is accepted.

If the Business has already verified and accepted a credential for a given
`(checkout_id, payment_hash)`, it MUST return the existing result rather
than re-fulfilling the order.

A Business without its own node MAY delegate these checks to its Invoice
Provider via the `/verify` endpoint (see [Invoice API — Option B](#option-b-provider-mediated-verification)).

### Retry on Invoice Expiry

If a BOLT11 invoice expires before the Platform completes payment, the
Platform MAY repeat the profile-specific acquisition flow with the same
`checkout_id` to obtain a fresh invoice. The binding mechanism carries the
same `checkout_id` into the new invoice. The Platform MUST NOT reuse a
preimage from a previously expired invoice.

### Completing the Checkout

The Platform submits the preimage credential in the UCP
`payment.instruments` array:

```json
POST /checkout-sessions/{checkout_id}/complete

{
  "payment": {
    "instruments": [
      {
        "id": "ln_bolt12_01HXYZ",
        "handler_id": "bolt12_main",
        "type": "com.musqet.preimage",
        "credential": {
          "type": "com.musqet.preimage",
          "preimage": "c3a1...d9f2",
          "checkout_id": "chk_01HXYZ"
        }
      }
    ]
  }
}
```

Set `handler_id` to the Business-assigned instance `id` of the handler used.

> **Note:** This example shows only the Lightning-specific payload.
> UCP-required fields (`Idempotency-Key` header, `signals`, `selected` on
> the instrument) are omitted for brevity. See the
> [UCP Checkout REST spec](https://ucp.dev/specification/checkout-rest/)
> for the full request shape.

---

## Profile: BOLT12 (`com.musqet.bolt12`)

### Handler Configuration

**Schema:** [`lightning/bolt12.config.json`](https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/tags/v2026-04-23/lightning/bolt12.config.json)

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `offer` | BOLT12 offer string (`lno1...`) | Yes | Static BOLT12 offer. MAY be amountless or amount-fixed. |
| `display_name` | string | No | Human-readable label for payment-method UI. |
| `supported_currencies` | array of ISO-4217 or `SAT` | No | Defaults to `["SAT"]`. |

### Platform Integration

**Prerequisites:** A Lightning wallet that can send BOLT12 `invoice_request`
messages and pay the resulting invoices.

#### Step 1: Decode the Offer

Decode `config.offer`. Relevant fields: issuer node id, amount/currency
constraints.

#### Step 2: Send `invoice_request`

Send a BOLT12 `invoice_request` to the issuer node:

- `payer_note`: MUST be set to the UCP `checkout_id`. The Business's node
  MUST accept `payer_note` values long enough to hold the `checkout_id`.
- `invoice_request_amount`: the amount to pay, if the offer is amountless.

#### Step 3: Pay and Capture Preimage

The issuer returns a BOLT12 invoice. Pay it and capture the 32-byte
preimage. MUST be normalised to lowercase hex before submission.

Then [complete the checkout](#completing-the-checkout) with the preimage
credential.

### Business Integration

1. Advertise the offer in `config.offer`.
2. The node MUST accept `payer_note` values long enough to hold the
   `checkout_id` and persist them alongside the issued invoice.
3. On credential receipt: compute `payment_hash = SHA256(preimage)`, look up
   in settled index, read `payer_note`, match to `credential.checkout_id`.
4. Apply the four [Verification](#verification) checks.

### Limitations

- BOLT12 adoption is still partial; not every Lightning node supports it.
- Fiat-denominated offers rely on the node's FX at invoice-request time.

---

## Profile: LNURL-pay (`com.musqet.lnurl-pay`)

### Handler Configuration

**Schema:** [`lightning/lnurl-pay.config.json`](https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/tags/v2026-04-23/lightning/lnurl-pay.config.json)

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `lightning_address` | string | One of `lightning_address` or `lnurl` | `user@host` → `https://{host}/.well-known/lnurlp/{user}`. Endpoint MUST support LUD-12 comments with `commentAllowed` large enough to hold the `checkout_id`. |
| `lnurl` | string | One of `lightning_address` or `lnurl` | Bech32 `lnurl1...` or direct `https://` URL. MUST support LUD-12 comments with `commentAllowed` large enough to hold the `checkout_id`. |
| `display_name` | string | No | Human-readable label. |

### Platform Integration

**Prerequisites:**
1. A Lightning wallet capable of paying BOLT11 invoices.
2. MUST validate TLS certificates on all HTTPS requests. Certificate errors
   MUST abort the flow.

#### Step 1: Resolve LNURL

If `lightning_address`, fetch `https://{host}/.well-known/lnurlp/{user}`.
If `lnurl` is bech32, decode to HTTPS URL per LUD-01.

The Platform MUST verify:
- `tag` equals `"payRequest"`.
- `commentAllowed` is present and `>= len(checkout_id)`.
  If not met, abort — the endpoint cannot carry the `checkout_id`.
- Retain `metadata` for the description-hash check in Step 2.

#### Step 2: Request BOLT11

```http
GET {callback}?amount={msats}&comment={checkout_id}
```

- `amount`: checkout total in millisats (`sats × 1000`). MUST be within
  `[minSendable, maxSendable]`.
- `comment`: MUST be set to the UCP `checkout_id`.

**Metadata hash verification (LUD-06).** Decode the returned BOLT11 and
verify `description_hash == SHA256(metadata)` from Step 1. If mismatch,
MUST NOT pay (prevents invoice-substitution attacks).

#### Step 3: Pay and Capture Preimage

Pay the BOLT11 and capture the preimage. MUST be normalised to lowercase hex.

Then [complete the checkout](#completing-the-checkout) with the preimage
credential.

### Business Integration

1. Host an LNURL-pay endpoint (or Lightning Address). The endpoint MUST
   support LUD-12 comments with `commentAllowed` large enough to hold the
   `checkout_id`.
2. Persist `(payment_hash, comment)` atomically at invoice creation.
3. On credential receipt: look up `payment_hash`, read `comment` to recover
   `checkout_id`, match to order.
4. Apply the four [Verification](#verification) checks.

> Endpoints without LUD-12 comment support cannot use this profile;
> use `com.musqet.invoice-api` instead.

### Limitations

- **Sats only.** Amounts are millisats. The Business MUST price orders in
  sats; fiat conversion must happen before checkout creation.

---

## Profile: Invoice API (`com.musqet.invoice-api`)

### Handler Configuration

**Schema:** [`lightning/invoice-api.config.json`](https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/tags/v2026-04-23/lightning/invoice-api.config.json)

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `merchant_id` | string | Yes | Public Business identifier at the Provider. NOT a secret. |
| `invoice_endpoint` | URI (`https://`) | Yes | Provider endpoint for BOLT11 invoice creation. |
| `display_name` | string | No | Human-readable label. |
| `supported_currencies` | array of 3-char codes | No | E.g. `["SAT", "USD", "EUR"]`. Omitted means `["SAT"]`. |

The `verify_endpoint` is **not** in the UCP profile — verification is a
Business↔Provider concern. The Business learns it during onboarding.

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
| `merchant_id` | string | Yes | MUST match the Business's UCP profile. |
| `checkout_id` | string | Yes | Idempotency key scoped per `merchant_id`. |
| `currency` | string, 3 chars | Yes | `SAT` or ISO-4217 code. MUST be in `supported_currencies`. |
| `amount` | integer, ≥ 1 | Yes | Minor units of `currency`. |
| `description` | string | No | Human-readable invoice description. |
| `expiry_seconds` | integer, ≥ 1 | No | Defaults to 600 (10 min). |

**Response (201 Created / 200 OK on idempotent retry):**

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
| `invoice_id` | string | Provider-issued ID. |
| `bolt11` | string | BOLT11 invoice to pay. |
| `payment_hash` | 64-char hex | Informational; Business re-derives from preimage. |
| `currency`, `amount` | | Echo of request. |
| `amount_sats` | integer | Sats amount locked at issuance. |
| `fx_rate` | number | `amount_sats / amount`. Omitted when `currency == "SAT"`. Locked at issuance. |
| `expires_at` | ISO-8601 | Invoice expiry. |

**Idempotency:** same `(merchant_id, checkout_id)` with matching
`(currency, amount)` returns the same invoice. Mismatched `(currency, amount)`
→ `409 amount_mismatch`. Expired invoice → new invoice with `201`.

**Authentication:** no per-agent auth. The `checkout_id` (opaque,
unguessable, Business-issued) acts as capability token. Providers MUST
rate-limit per `(merchant_id, client-ip)`.

#### Pay and Capture Preimage

The Platform pays the `bolt11` from the response and captures the 32-byte
preimage. MUST be normalised to lowercase hex.

Then [complete the checkout](#completing-the-checkout) with the preimage
credential.

### Business Integration

#### Option A: Local verification

The Business verifies preimages locally using the four
[Verification](#verification) checks.

#### Option B: Provider-mediated verification

The Business calls the Provider's `/verify` endpoint (private, authentication
is between the Business and Provider).

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

The Provider MUST return `403 binding_mismatch` if the preimage's invoice is
bound to a different `checkout_id`, or `404 invoice_not_found` if it belongs
to another Business.

### Limitations

- More operational surface than BOLT12 or LNURL.
- Requires Business↔Provider trust for FX lock and settlement (Option B).

---

## Security Considerations

| Requirement | Description |
|:------------|:------------|
| **TLS everywhere** | All HTTP surfaces MUST use TLS. Platforms MUST validate certificates and MUST NOT fall back to HTTP. |
| **Credential binding** | `checkout_id` MUST be in the `credential` payload (not headers/query) so it is covered by checkout integrity. |
| **Issuance-time binding** | `(payment_hash → checkout_id)` MUST be persisted at invoice creation. Source: `payer_note` (BOLT12), LUD-12 `comment` (LNURL-pay), or Provider record (Invoice API). |
| **Preimage is not Platform-exclusive** | Revealed to every routing hop at settlement. Businesses MUST treat it as a shared secret — authorization comes from issuance-time binding, not preimage possession. |
| **Settlement observed locally** | Business MUST confirm settlement via own node (Option A) or Provider (Option B). Platform's claim alone is never sufficient. |
| **Invoice expiry** | Pay well before `expires_at`. Retry with same `checkout_id` if expired. |
| **Amount tampering** | Business MUST verify paid sats against expected amount locked at invoice creation. |
| **`checkout_id` entropy** | The `checkout_id` is the binding key across all profiles. Businesses MUST ensure checkout identifiers are unguessable (e.g., CSPRNG-generated, ≥128 bits of entropy). |
| **Rate limiting** | Providers and LNURL endpoints MUST rate-limit per `(merchant_id, client-ip)`. |
| **Cross-tenant info leak** | `/verify` MUST return `404 invoice_not_found` uniformly for non-owned and non-existent invoices. |
| **LNURL metadata hash** | Platform MUST verify `description_hash == SHA256(metadata)` before paying (LUD-06). Prevents invoice-substitution attacks. |
| **Comment integrity** | LUD-12 `comment` is over TLS. Business MUST persist comment atomically with invoice creation. Third-party LNURL services MUST be verified to persist comments. |
| **No keysend / spontaneous payments** | Credentials derived from keysend or AMP spontaneous payments MUST be rejected. These payments carry no binding (`payer_note`, `comment`, or Provider record), so verification step 2 always fails. |
| **Hex normalisation** | Preimage MUST be lowercase hex (`[0-9a-f]{64}`) before storage, comparison, or hashing. |

---

## Error Codes

Responses use JSON `{ "code": "...", "message": "..." }`. These codes apply
to the Invoice API endpoints (`invoice_endpoint`, `verify_endpoint`) and to
UCP-layer error mapping. LNURL endpoints follow their own LUD-06 error
conventions (`{ "status": "ERROR", "reason": "..." }`).

| HTTP | Code | UCP Category | Meaning | Retryable |
|:-----|:-----|:-------------|:--------|:----------|
| 400 | `invalid_request` | `validation_error` | Schema validation or invalid field values. | No |
| 400 | `unsupported_currency` | `validation_error` | Currency not in `supported_currencies`. | No |
| 400 | `amount_out_of_range` | `validation_error` | Amount violates min/max. | No |
| 403 | `binding_mismatch` | `payment_declined` | Preimage bound to a different `checkout_id` (same Business). | No |
| 404 | `merchant_not_found` | `configuration_error` | Unknown `merchant_id`. | No |
| 404 | `invoice_not_found` | `payment_declined` | No matching invoice (includes cross-tenant hiding). | No |
| 409 | `amount_mismatch` | `conflict` | Idempotent retry with different `(currency, amount)`. | No |
| 409 | `lightning_not_enabled` | `configuration_error` | Business hasn't finished Lightning onboarding. | No |
| 410 | `invoice_expired` | `payment_expired` | Invoice expired unpaid. Re-acquire with same `checkout_id`. | Yes |
| 429 | `rate_limited` | `rate_limited` | Rate limit exceeded. | Yes (backoff) |
| 503 | `provider_unavailable` | `temporarily_unavailable` | Upstream temporarily unavailable. | Yes (backoff) |

---

## AP2 Compatibility

The preimage credential MAY be submitted alongside a presented AP2 mandate
that binds the same UCP checkout session. No additional Lightning-specific
mandate fields are required. See the
[AP2 Protocol Spec](https://ucp.dev/specification/ap2-mandates/) for
mandate structure and binding semantics.

---

## Test Vectors

```
preimage     = 0000000000000000000000000000000000000000000000000000000000000001
payment_hash = 4bf5122f344554c53bde2ebb8cd2b7e3d1600ad631c385a5d7cce23c7785459a
```

(`SHA256(\x00...\x01)` over the 32 raw bytes.)

---

## Governance & Licensing

This spec was authored by [Musqet](https://musqet.com) and is open for
community review and contribution. Published under
[Apache License 2.0](http://www.apache.org/licenses/LICENSE-2.0).

---

## References

- **BOLT11:** [Invoice protocol](https://github.com/lightning/bolts/blob/master/11-payment-encoding.md)
- **BOLT12:** [Offers](https://github.com/lightning/bolts/blob/master/12-offer-encoding.md)
- **LNURL-pay (LUD-06):** [Payment request](https://github.com/lnurl/luds/blob/luds/06.md)
- **Lightning Address:** [lightningaddress.com](https://lightningaddress.com/)
- **LUD-12:** [Comments in payRequest](https://github.com/lnurl/luds/blob/luds/12.md)
- **UCP Payment Handler Guide:** [ucp.dev/specification/payment-handler-guide](https://ucp.dev/specification/payment-handler-guide/)
- **UCP AP2 Mandates:** [ucp.dev/specification/ap2-mandates](https://ucp.dev/specification/ap2-mandates/)
