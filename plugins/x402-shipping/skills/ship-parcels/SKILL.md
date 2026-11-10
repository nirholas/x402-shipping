---
name: ship-parcels
description: Rate-shop a parcel and buy a shipping label through Shippo or EasyPost test mode, paid per call over x402. Use when an agent needs shipping rates or a label returned inline.
---

# x402-shipping rates and labels

Two paid calls: price a shipment across carriers, then buy the chosen rate. The label PDF comes back base64 in the same response.

## Run the server

The repo is a self-hosted Express service. It is not published to npm, so run it from a clone.

```bash
git clone https://github.com/nirholas/x402-shipping && cd x402-shipping
npm install
cp .env.example .env   # optional: the defaults run out of the box
npm run dev            # listens on http://localhost:4023 (override with PORT)
```

Set `PAY_TO_ADDRESS` (Base) and `SOLANA_PAY_TO_ADDRESS` (Solana) in `.env` to receive payments yourself.
Carrier credentials go in `.env`: `SHIPPO_API_TOKEN`, or `EASYPOST_API_KEY` (used only when the Shippo token is unset). Without either, the server runs in fixture mode.
`GET http://localhost:4023/.well-known/x402` lists every route with its price, and `GET /health` is the liveness probe.

## Routes

| Route | Price | Purpose |
|---|---|---|
| `POST /rates` | $0.003 | Body: `from`, `to` (each `name`, `street1`, `city`, `state`, `zip`, `country`), `parcel` (`length`, `width`, `height`, `weight`, `distanceUnit` in or cm, `massUnit` oz, lb, g or kg). Returns rates with `rateId`, plus `cheapest` and `fastest` |
| `POST /label` | $0.02 | Body `{rateId}`. In fixture mode also resend `from`, `to`, `parcel`. Returns tracking number and `labelPdfBase64` |
| `GET /health`, `GET /.well-known/x402` | free | Liveness and discovery |

Errors: 400 `invalid_address`, `invalid_parcel`, `missing_rate_id`; 404 `rate_not_found`; 502 `label_unavailable`, `upstream_error`.

The full request and response schemas are in `skill.md` at the repo root (the running server also serves it at `GET /skill.md`).

## Paying

Paid routes answer an unpaid request with HTTP 402 and an `accepts` array with one entry per rail (USDC on Base and USDC on Solana). A client picks an entry, signs, and retries with the `X-PAYMENT` header. The 200 body carries the artifact and an `X-PAYMENT-RESPONSE` header carries the settlement receipt. Any x402 client works (`x402-fetch` with `viem` on the EVM rail). Each paid call spends real USDC on whichever network the server is configured for (`NETWORK`, `SOLANA_NETWORK`), so show the user the route, price and network and get a yes before paying.

## Worked example

```bash
curl -s http://localhost:4023/health
```

After the user approves the $0.003, pay `POST /rates` with a `from`, `to` and `parcel` body and note the cheapest `rateId`. After a second approval for $0.02, pay `POST /label` with that `rateId` and write `Buffer.from(labelPdfBase64, 'base64')` to a file.
