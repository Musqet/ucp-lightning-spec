# UCP Lightning Network Payment Handlers

Payment handler specification for the [Universal Commerce Protocol (UCP)](https://ucp.dev) enabling Lightning Network payments between autonomous agents and businesses.

## Profiles

| Family Key | Method |
|:-----------|:-------|
| `com.musqet.bolt12` | BOLT12 offers — peer-to-peer, no HTTP surface |
| `com.musqet.lnurl-pay` | LNURL-pay / Lightning Address — sats-only |
| `com.musqet.invoice-api` | Provider-mediated HTTP API — fiat support, FX locking |

All three produce the same credential: a SHA-256 preimage proving HTLC settlement.

## Files

- [`lightning-network-payment-handler.md`](lightning-network-payment-handler.md) — Full specification
- [`lightning/`](lightning/) — JSON Schemas (instrument, credential, config per profile)

## Status

**Beta** — version `2026-04-28`, targeting UCP `2026-04-08`.

## License

[Apache License 2.0](LICENSE)
