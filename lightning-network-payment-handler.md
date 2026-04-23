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
* **Version:** `2026-04-22` (Draft)
* **Target UCP Version:** `2026-04-08`
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
| `com.musqet.bolt12`      | BOLT12 offers (no HTTP surface, no shared secret)                      | Non-custodial Businesses whose Lightning node already supports BOLT12. Agents request invoices directly from the node over the Lightning peer-to-peer network. |
| `com.musqet.lnurl-pay`   | LNURL-pay / Lightning Address                                          | Businesses with existing LNURL-pay infrastructure (mobile wallet apps, Alby, BTCPayServer, etc.). Works with any Lightning node. Sats-only. |
| `com.musqet.invoice-api` | Provider-mediated HTTP API for invoice issuance and (optional) verification | Businesses that want fiat-denominated invoices with FX locking or provider-managed invoice lifecycle. |

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

**Schema:** `https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/heads/main/lightning/instrument.json`

The instrument schema extends the UCP base `payment_instrument.json` via `allOf`
and declares a `$defs.available_lightning_instrument` variant for use in
`available_instruments` arrays.

```json
{
  "id": "ln_1a2b3c4d",
  "handler_id": "bolt12",
  "type": "com.musqet.preimage",
  "credential": { ... }
}
```

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `id` | string | Yes | Platform-assigned instrument identifier. |
| `handler_id` | string | Yes | Business-assigned handler instance id. MUST match the `id` of an entry in the Business's `payment.handlers[]` whose `name` is one of `com.musqet.bolt12`, `com.musqet.lnurl-pay`, or `com.musqet.invoice-api`. |
| `type` | const `com.musqet.preimage` | Yes | Credential type. Same across all profiles. |
| `credential` | object | Yes | See [Payment Credential](#payment-credential). |
| `billing_address` | object | No | Not used for payment verification (Lightning has no AVS). MAY be provided when the Business requires a billing or shipping address for order fulfillment. |
| `display` | object | No | Optional display metadata (`label`, `icon_url`) for Platform UIs. |

### Payment Credential

**Schema:** `https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/heads/main/lightning/credential.json`

```json
{
  "type": "com.musqet.preimage",
  "preimage": "c3a1...d9f2",
  "checkout_id": "chk_01HXYZ..."
}
```

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `type` | const `com.musqet.preimage` | Yes | Credential type. |
| `preimage` | 64-char lowercase hex | Yes | The 32-byte preimage revealed during HTLC settlement. MUST be normalised to lowercase hex (`[0-9a-f]{64}`). |
| `checkout_id` | string | Yes | The UCP checkout this payment settles. Carried in-credential so binding data is covered by checkout integrity, not transport headers. |

The credential is intentionally minimal. The `payment_hash` is derived by
the Business as `SHA256(preimage)`; the BOLT11 the Platform paid is already
held by the Business (keyed by `payment_hash`), so re-submitting it would
add nothing a forger couldn't also provide.

> **Credential expiry exemption.** UCP recommends that token-style
> credentials carry an expiry field. A Lightning preimage is not a
> reusable token — it is a one-time cryptographic proof of a specific
> HTLC settlement. Its validity is bounded by the BOLT11 invoice's own
> `expires_at`: once the invoice expires unpaid, no preimage can be
> produced; once paid, the preimage is permanently valid as proof of that
> settlement. Accordingly, no separate `expiry` or `ttl` field is
> included in the credential. Implementations that need time-bounded
> acceptance SHOULD enforce the invoice's `expires_at` during
> verification step 3 (Settlement).

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
     service. The endpoint MUST support LUD-12 comments with
     `commentAllowed >= 64`; see
     [Comment-based binding](#comment-based-binding).
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

## Profile: BOLT12 (`com.musqet.bolt12`)

### When to use

A Business whose Lightning node supports BOLT12 offers (e.g., Core
Lightning with `experimental-offers`, or LND with a BOLT12 plugin). No HTTP
endpoint is required on the Business side; everything happens over the
Lightning peer-to-peer network.

### Handler Configuration

**Schema:** `https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/heads/main/lightning/bolt12.config.json`

```json
{
  "payment": {
    "handlers": [
      {
        "id": "bolt12",
        "name": "com.musqet.bolt12",
        "version": "2026-04-22",
        "spec": "https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/heads/main/lightning-network-payment-handler.md",
        "config_schema": "https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/heads/main/lightning/bolt12.config.json",
        "instrument_schemas": [
          "https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/heads/main/lightning/instrument.json"
        ],
        "available_instruments": [
          { "type": "com.musqet.preimage" }
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
    "type": "com.musqet.preimage",
    "credential": {
      "type": "com.musqet.preimage",
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

## Profile: LNURL-pay (`com.musqet.lnurl-pay`)

### When to use

A Business that already exposes an LNURL-pay endpoint or a Lightning
Address (the overwhelming majority of existing Lightning merchant setups).
HTTP-based, widely supported by wallets, but sats-only.

### Handler Configuration

**Schema:** `https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/heads/main/lightning/lnurl-pay.config.json`

```json
{
  "payment": {
    "handlers": [
      {
        "id": "lnurl-pay",
        "name": "com.musqet.lnurl-pay",
        "version": "2026-04-22",
        "spec": "https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/heads/main/lightning-network-payment-handler.md",
        "config_schema": "https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/heads/main/lightning/lnurl-pay.config.json",
        "instrument_schemas": [
          "https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/heads/main/lightning/instrument.json"
        ],
        "available_instruments": [
          { "type": "com.musqet.preimage" }
        ],
        "config": {
          "lightning_address": "alice@example.com",
          "display_name": "Pay with Lightning"
        }
      }
    ]
  }
}
```

| Field | Type | Required | Description |
|:------|:-----|:---------|:------------|
| `lightning_address` | string | One of `lightning_address` or `lnurl` | Lightning Address (`user@host`). Resolves to `https://{host}/.well-known/lnurlp/{user}`. The resolved endpoint MUST support LUD-12 comments with `commentAllowed >= 64`. |
| `lnurl` | string (bech32 `lnurl1...` or `https://...`) | One of `lightning_address` or `lnurl` | Direct LNURL-pay endpoint. MUST support LUD-12 comments with `commentAllowed >= 64`. |
| `display_name` | string | No | Human-readable label. |

> **Platform config.** No Platform-side onboarding or configuration is
> required for this profile. The Platform only needs a Lightning wallet
> capable of paying BOLT11 invoices. Accordingly, there is no separate
> `platform_config` schema.
>
> **Response config.** No additional runtime state beyond the static
> handler config is returned in checkout responses. The Platform resolves
> the LNURL at payment time. Accordingly, there is no separate
> `response_config` schema.

### Platform Integration

#### Prerequisites

1. A Lightning wallet capable of paying BOLT11 invoices.
2. The Platform MUST validate TLS certificates on all HTTPS requests in
   this flow (LNURL resolution, callback). Certificate errors MUST abort
   the flow — the Platform MUST NOT fall back to HTTP or ignore invalid
   certificates.

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

The Platform MUST verify:

- `tag` equals `"payRequest"`.
- `commentAllowed` is present and `>= 64`. If the endpoint does not
  support comments or `commentAllowed < 64`, the Platform MUST abort
  with an error — the LNURL endpoint does not meet the requirements of
  this profile.
- `metadata` is retained for the description-hash check in Step 2.

#### Step 2: Request BOLT11

```http
GET {callback}?amount={msats}&comment={checkout_id}
```

- `amount` is the checkout total in millisats. Since this profile is
  sats-only, the Business MUST have priced the order in sats and the
  UCP checkout total MUST already be denominated in sats (or the
  equivalent millisats value). The Platform converts: `msats = sats × 1000`.
  `amount` MUST be within `[minSendable, maxSendable]`. If the checkout
  total falls outside this range, the Platform MUST abort and surface a
  `amount_out_of_range` error.
- `comment` MUST be set to the UCP `checkout_id`. The Platform MUST
  verify that `commentAllowed >= len(checkout_id)` (checked in Step 1).

Response:

```json
{
  "pr": "lnbc10u1p3xnhl2...",
  "routes": []
}
```

**Metadata hash verification (LUD-06).** The Platform MUST decode the
returned BOLT11 invoice and verify that the invoice's `description_hash`
equals `SHA256(metadata)`, where `metadata` is the raw string returned in
Step 1. This prevents a MITM from substituting an invoice that pays a
different destination. If the hash does not match, the Platform MUST NOT
pay the invoice.

#### Step 3: Pay and Capture Preimage

Platform pays `pr` and captures the 32-byte preimage revealed during HTLC
settlement. The preimage MUST be normalised to lowercase hex before
submission.

#### Step 4: Submit Credential

Same shape as [BOLT12 Step 4](#step-4-submit-credential), with
`handler_id: "lnurl-pay"`.

#### Retry on Invoice Expiry

If the BOLT11 invoice expires before the Platform completes payment, the
Platform MAY repeat Steps 1–3 with the same `checkout_id` to obtain a
fresh invoice. The LNURL callback is stateless with respect to prior
invoices, so a new `GET` with the same parameters produces a new BOLT11.
The Business's LNURL endpoint will issue a new invoice (with a new
`payment_hash`) and bind the same `checkout_id` via the `comment`
parameter. The Platform MUST NOT reuse a preimage from a previously
expired invoice.

### Business Integration

1. Host an LNURL-pay endpoint (or a Lightning Address that proxies to one).
   Existing tools that provide this out-of-the-box: BTCPayServer,
   LNbits, Alby Hub, Phoenixd, Zeus.
2. The endpoint MUST support LUD-12 comments with `commentAllowed >= 64`.
   The Business's LNURL service MUST persist the `comment` value from
   each callback request alongside the generated invoice's `payment_hash`.
3. On payment, look up `payment_hash` on the node; read the recorded
   `comment` to recover `checkout_id`; match to order.
4. Apply the four verification checks from [Verification](#verification).

#### Comment-based binding

The `checkout_id` is carried to the Business via the LUD-12 `comment`
parameter on the LNURL callback request. This is the sole binding
mechanism for the LNURL-pay profile.

**Requirements:**

- The LNURL endpoint MUST advertise `commentAllowed >= 64` in the
  LUD-06 response. A value of 64 accommodates typical UCP checkout
  identifiers (ULIDs = 26 chars, UUIDs = 36 chars, prefixed IDs up to
  ~60 chars). Implementations SHOULD use `commentAllowed: 128` or higher
  to allow for future growth.
- The LNURL endpoint MUST persist `(payment_hash, comment)` at
  invoice-creation time. If the endpoint does not store the comment,
  binding verification will fail and the payment cannot be matched to
  an order.
- If `commentAllowed` is absent or `< 64`, the endpoint does not qualify
  for the `com.musqet.lnurl-pay` profile. The Business MUST either
  upgrade the endpoint or use the `com.musqet.invoice-api` profile
  instead.

### Limitations

- **Sats only.** LNURL-pay amounts are millisats. The Business MUST
  price the order in sats and the UCP checkout total MUST be denominated
  in sats. Fiat-to-sats conversion is the Business's responsibility and
  MUST be completed before the checkout is created.
- **LUD-12 comment support required.** Endpoints that do not support
  LUD-12 comments with `commentAllowed >= 64` cannot use this profile;
  use `com.musqet.invoice-api` instead.

---

## Profile: Invoice API (`com.musqet.invoice-api`)

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

**Schema:** `https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/heads/main/lightning/invoice-api.config.json`

```json
{
  "payment": {
    "handlers": [
      {
        "id": "invoice-api",
        "name": "com.musqet.invoice-api",
        "version": "2026-04-22",
        "spec": "https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/heads/main/lightning-network-payment-handler.md",
        "config_schema": "https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/heads/main/lightning/invoice-api.config.json",
        "instrument_schemas": [
          "https://raw.githubusercontent.com/Musqet/ucp-lightning-spec/refs/heads/main/lightning/instrument.json"
        ],
        "available_instruments": [
          { "type": "com.musqet.preimage" }
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

---

## Security Considerations

| Requirement | Description |
|:------------|:------------|
| **TLS everywhere** | All HTTP surfaces (LNURL endpoint, Invoice API) MUST use TLS. Platforms MUST validate server certificates and MUST NOT fall back to plaintext HTTP or ignore certificate errors. |
| **Credential binding** | `checkout_id` MUST be carried in the `credential` payload, not in headers or query strings, so it is covered by checkout integrity. |
| **Issuance-time binding** | The Business (or Invoice Provider) MUST persist `(payment_hash → checkout_id)` at invoice-creation time. Verification rejects any credential whose recomputed `payment_hash` does not resolve to the submitted `checkout_id`. Per profile the bound value lives in `payer_note` (BOLT12), the LUD-12 `comment` (LNURL-pay), or the Provider's invoice record (Invoice API). |
| **Preimage is not Platform-exclusive** | A Lightning preimage is revealed to every routing hop at HTLC settlement, so any intermediary may learn it before the Platform submits the credential. Businesses MUST treat the preimage as a shared secret: it proves *an* HTLC settled, not *who* paid. Authorization of the payment comes from the issuance-time binding above, not from preimage possession. |
| **Settlement observed locally** | The Business MUST confirm settlement by consulting their own Lightning node (Option A) or their Invoice Provider (Option B). The Platform's claim of settlement is never sufficient on its own. |
| **Invoice expiry** | Platforms SHOULD pay invoices well before `expires_at`. Expired invoices produce `invoice_expired` on verification. Platforms MAY retry the invoice-acquisition flow with the same `checkout_id` if an invoice expires before payment; see the retry guidance in each profile. |
| **Amount tampering (Platform-side)** | Because the Platform (agent) chooses the amount in the `invoice_request` / LNURL callback / Invoice API POST, the Business MUST re-check the paid sats against the expected amount for the order. Expected amount is the sats amount the Business (or its Provider, in `invoice-api`) locked at invoice creation. |
| **Rate limiting** | Providers and LNURL endpoints MUST rate-limit invoice creation per `(merchant_id, client-ip)` to mitigate abuse. |
| **Cross-tenant info leak (invoice-api)** | The `/verify` endpoint MUST NOT reveal whether a preimage belongs to another Business. Return `404 invoice_not_found` uniformly for non-owned and non-existent invoices. |
| **LNURL metadata hash (LUD-06)** | Before paying a BOLT11 obtained via LNURL-pay, the Platform MUST verify that the invoice's `description_hash` equals `SHA256(metadata)` from the LUD-06 response. This prevents invoice-substitution attacks where a MITM replaces the callback response with an invoice paying a different node. |
| **Comment integrity (LNURL-pay)** | The LUD-12 `comment` parameter carrying `checkout_id` is transmitted in plaintext over TLS. The binding's integrity depends on the Business's LNURL service faithfully persisting the comment alongside the `payment_hash`. Businesses MUST ensure their LNURL implementation records the `comment` atomically with invoice creation. If using a third-party LNURL service, the Business MUST verify that the service persists comments and makes them available for verification. |
| **Hex normalisation** | Preimage values MUST be normalised to lowercase hex (`[0-9a-f]{64}`) before storage, comparison, or hashing. Mixed-case comparison could cause verification failures or allow duplicate credentials to be treated as distinct. |

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

### UCP Standard Error Mapping

The following table maps Lightning handler error codes to UCP standard
error categories. Platforms and Businesses MUST use the UCP error
definitions when surfacing failures through UCP APIs.

| Handler Code | UCP Error Category | UCP Guidance |
|:-------------|:-------------------|:-------------|
| `invalid_request` | `validation_error` | Non-retryable. The Platform should fix the request. |
| `unsupported_currency` | `validation_error` | Non-retryable. The Platform should select a supported currency. |
| `amount_out_of_range` | `validation_error` | Non-retryable. The Platform should adjust the amount. |
| `binding_mismatch` | `payment_declined` | Non-retryable. Indicates a potential replay or misuse. |
| `merchant_not_found` | `configuration_error` | Non-retryable. Business onboarding may be incomplete. |
| `invoice_not_found` | `payment_declined` | Non-retryable. Preimage does not correspond to a known invoice. |
| `amount_mismatch` | `conflict` | Non-retryable. An invoice for this checkout already exists with different terms. |
| `lightning_not_enabled` | `configuration_error` | Non-retryable. Business must complete Lightning onboarding. |
| `invoice_expired` | `payment_expired` | Retryable. The Platform may re-acquire an invoice with the same `checkout_id`. |
| `rate_limited` | `rate_limited` | Retryable after backoff. |
| `provider_unavailable` | `temporarily_unavailable` | Retryable after backoff. |

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

This spec was authored by [Musqet](https://musqet.com) and is open for
community review and contribution. The spec is published under
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

