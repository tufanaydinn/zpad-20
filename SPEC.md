# ZPAD-20 — Bonding-Curve Token Standard for Zcash

> **Status:** Draft v0.1 · **Network:** testnet first, then mainnet
> **Layer:** application meta-protocol (no consensus changes)
> **License:** open — anyone may implement an indexer/wallet against this spec.

ZPAD-20 is a meta-protocol for issuing and trading fungible tokens on Zcash with a
built-in **bonding curve** (instant liquidity, deterministic pricing). It is the
launchpad analogue of inscription standards like BRC-20/ZRC-20, but shaped for a
pump.fun-style create → trade → graduate flow rather than a marketplace.

---

## 1. Rationale

Zcash has **no smart-contract VM**. The only trustless way to put a token on-chain
is a **meta-protocol**: write operations into transactions and let an **indexer**
recompute state deterministically from the chain (same model as Bitcoin Ordinals /
BRC-20). ZPAD-20 defines those operations and the rules every indexer must follow,
so that anyone running the rules derives the **same** state.

The bonding curve itself is **off-chain compute, on-chain settlement**: prices are a
pure deterministic function of cumulative ZEC raised (auditable by anyone), while
value transfer happens in real Zcash transactions.

---

## 2. Privacy model

> **Private participation, transparent supply.**

- **Payments** move as shielded ZEC. **Intent** (which op) may travel in the
  shielded 512-byte memo.
- The **token ledger is transparent** in this version so any indexer can verify
  supply. ZPAD-20 does **not** publish per-wallet leaderboards, trader feeds or
  holder identities — implementations MUST expose only aggregates (distribution,
  counts), never address-level surveillance.
- A **fully shielded ledger** is a future tier (Model B) once Zcash Shielded Assets
  (ZSA / NU7) land; the op semantics below carry over.

---

## 3. Envelope

Every operation is a UTF-8 JSON object with a required protocol tag, carried inside
a Zcash transaction:

```json
{ "p": "zpad-20", "op": "<operation>", ... }
```

**Carriers (in priority order an indexer MUST scan):**

1. **Shielded memo** (ZIP-302, 512 bytes) — for private intent. Readable by the
   recipient/holder of the relevant viewing key (e.g. the curve-reserve's UFVK).
2. **Transparent `OP_RETURN`** — for publicly-verifiable ops (the transparent
   ledger). Requires reading **full blocks** (lightwalletd compact blocks omit
   transparent data — indexers read full blocks from `zebrad`).

An op larger than a single memo MAY be chunked across memos with `{ "i": <index>,
"n": <total> }` and reassembled deterministically by `(txid, index)`.

---

## 4. Operations

### 4.1 `deploy` — define a token

```json
{
  "p": "zpad-20", "op": "deploy",
  "tick": "ZINU", "name": "Zinu",
  "supply": 1000000000,
  "curve": { "z0": 30, "t0": 1073000191, "grad": 85 },
  "fee_bps": 100
}
```

| field    | type   | rule |
|----------|--------|------|
| `tick`   | string | 1–12 chars, unique (first valid deploy wins; later duplicates ignored) |
| `name`   | string | 1–40 chars |
| `supply` | int    | fixed at `1_000_000_000` in v0.1 |
| `curve`  | object | virtual reserves + graduation (see §5); fixed constants in v0.1 |
| `fee_bps`| int    | trade fee in basis points (`100` = 1%) |

The deploy tx also pays the **launch fee** (`0.1 ZEC`) to the platform/pool address.

### 4.2 `buy` — purchase from the curve

```json
{ "p": "zpad-20", "op": "buy", "tick": "ZINU", "zec_in": 1.0 }
```

The tx sends `zec_in` shielded ZEC to the **curve-reserve** address. The indexer
computes `tokens_out` from the curve (§5) and credits the sender.

### 4.3 `sell` — sell back to the curve

```json
{ "p": "zpad-20", "op": "sell", "tick": "ZINU", "amt": 32258065 }
```

`amt` tokens return to the curve; the reserve sends `zec_out` (minus fee) to the
seller.

### 4.4 `transfer` — move tokens between users

```json
{ "p": "zpad-20", "op": "transfer", "tick": "ZINU", "amt": 1000000, "to": "u1..." }
```

### 4.5 `ticket` — Shadow Pass reward ticket

```json
{ "p": "zpad-20", "op": "ticket", "net": "main", "month": "2026-06", "quest": "trade", "to": "u1..." }
```

A claim minted when a user completes a daily quest. One ticket op per quest per
day; the monthly draw (§7) selects winners from claimed tickets.

---

## 5. Bonding curve

Constant-product with virtual reserves (pump.fun-faithful, ZEC-denominated):

```
Z = virtual ZEC reserve, T = virtual token reserve,  k = Z · T  (constant)
spot price = Z / T
```

**Constants (v0.1):** `Z0 = 30 ZEC`, `T0 = 1,073,000,191`, `k = Z0·T0`,
`TOTAL_SUPPLY = 1e9`, `CURVE_SUPPLY = 800,000,000` sold on the curve,
`GRAD = 85 ZEC` raised, `FEE = 1%` (fee-on-input).

**Buy** (`x` ZEC in, fee skimmed first):
```
fee       = x · FEE
effIn     = x − fee
tokensOut = T − k / (Z + effIn)
Z' = Z + effIn ;  T' = T − tokensOut
```

**Sell** (`n` tokens in, fee skimmed from output):
```
grossZec  = Z − k / (T + n)
fee       = grossZec · FEE
zecOut    = grossZec − fee
Z' = Z − grossZec ;  T' = T + n
```

State for each token is fully determined by **cumulative ZEC raised** (`raised`):
`Z = Z0 + raised`, `T = k / Z`. Worked example (no fee): at `Z=30, T=1e9`, a `1 ZEC`
buy yields `32,258,065` tokens — reproducible by any implementation.

---

## 6. Indexer rules

An indexer MUST:

1. Process blocks **in height order**; within a block, ops in tx/output order.
2. **Validate** each op (known `op`, well-formed fields, tick exists for buy/sell/
   transfer, sufficient balance for sell/transfer). Invalid/duplicate ops are
   **ignored**, never errored — state must remain deterministic.
3. Maintain per-token: `raised`, `tokens_sold`, holder balances, and (for fee/pool
   accounting) the launch/trade/featured/ticket fee splits.
4. Derive price, market cap, circulating supply and **holder distribution**
   (counts/concentration only — never expose addresses) from the above.
5. Be **reproducible**: same chain → same state. Handle reorgs by rolling back to
   the common ancestor and replaying. Publishing the ruleset lets anyone run a
   verifying indexer (no trusted oracle).

The reference implementation reads full blocks from `zebrad` for transparent ops
and uses a `lightwalletd` viewing key to detect shielded payments to the reserve.

---

## 7. Monthly draw (Shadow Pass)

Verifiable and tamper-proof — no admin can change the outcome:

```
seedHash      = BLAKE2b( zcashBlockHash | ticketListHash )
ticketScore_i = BLAKE2b( seedHash | ticketNumber_i )
winners       = the lowest scores (rank order)
```

- `zcashBlockHash` = the month's final mainnet block hash (announced in advance).
- `ticketListHash` = BLAKE2b of all claimed ticket numbers for the month.
- Anyone can recompute the winners. Wallets are never revealed (ticket numbers
  only). Prize split: 45% / 25% / 15% to top 3; 15% carries over.

---

## 8. Fees & pool

| fee            | value        | notes |
|----------------|--------------|-------|
| Launch         | 0.1 ZEC      | flat, on `deploy` |
| Trade          | 1% (100 bps) | fee-on-input, every buy/sell |
| Graduation     | one-time cut | when `raised ≥ GRAD` |
| Ticket claim   | ~0.001 ZEC   | makes the ticket real on-chain |
| Featured slot  | 1 / 2.5 / 5 ZEC | 24h / 3d / 7d homepage promotion |

A configured share of each fee funds the community **prize pool** (sustainable, not
minted). Network fees follow ZIP-317 (~0.0001 ZEC/tx) and go to Zcash, not the
platform. All fees are shown before confirmation.

---

## 9. Graduation

When a token reaches `GRAD` (85 ZEC raised, ≈ 793M tokens sold) it **graduates**.
Zcash has no on-chain DEX, so liquidity does not migrate to a third party: the
collected ZEC + remaining tokens become a permanent platform pool and trading
continues on ZecPad. Future: trustless P2P markets via PCZT atomic swaps, and
migration to a Zcash DEX once ZSA exists.

---

## 10. Versioning

`p` is fixed as `zpad-20`. Breaking changes bump a future tag (e.g. `zpad-21`).
Indexers ignore unknown `op`/fields for forward-compatibility. Model B (native
shielded ZSA token tier) will reuse these op semantics over a shielded ledger.

---

## 11. References

- Zcash Protocol Spec — https://zips.z.cash/protocol/protocol.pdf
- ZIP-302 (memo) · ZIP-317 (fees) · ZIP-316 (unified addresses)
- ZIP-226 / ZIP-227 (ZSA / OrchardZSA)
