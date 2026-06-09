# ZPAD-20

**An open bonding-curve token standard for Zcash.**

Zcash has no smart-contract VM, so a token cannot live "as code" on chain. ZPAD-20 is an
**application meta-protocol** (no consensus changes): operations are written into Zcash
transactions and an **indexer** recomputes token state deterministically from the chain —
the same trustless model as Bitcoin's BRC-20 / Ordinals, but shaped for a pump.fun-style
**create → trade → graduate** launchpad flow.

Anyone may implement an indexer or wallet against this spec. Same chain → same state, with
no trusted oracle.

## Read the spec

👉 **[SPEC.md](./SPEC.md)** — full specification (envelope, operations, bonding curve,
indexer rules, draw, fees, graduation).

## At a glance

- **Operations:** `deploy` · `buy` · `sell` · `transfer` (+ a Shadow Pass `ticket` op)
- **Carriers:** shielded memo (ZIP-302, private intent) or transparent `OP_RETURN`
- **Curve:** constant-product with virtual reserves (`Z·T = k`), price `= Z / T`,
  deterministic from cumulative ZEC raised
- **Privacy:** *private participation, transparent supply* — implementations expose only
  aggregates (counts, concentration), never per-wallet surveillance
- **Status:** Draft v0.1 · testnet first, then mainnet

## Reference implementation

A reference indexer and front-end ([ZecPad](https://zecpad.com)) are in development. The
indexer that derives ZPAD-20 state from the Zcash chain is being built as open-source,
public-good infrastructure (Zcash Community Grants).

## License

[MIT](./LICENSE) — open; implement freely.
