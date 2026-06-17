# ZPAD-20 — Bonding-Curve Token Standard for Zcash

> **Status:** Draft v0.2 · **Network:** testnet first, then mainnet
> **Layer:** application meta-protocol (no consensus changes)
> **License:** open — anyone may implement an indexer/wallet against this spec.
>
> **v0.2 changes:** added **§4.6 Identity & authorization** (claim-key + signed
> `sell`/`transfer`, the missing answer to "who owns a balance when the payer is
> shielded"); strengthened **§6** with explicit **atomicity**, **confirmation/reorg**, and
> **security-invariant** rules; made the **settle-against-on-chain-value** rule normative
> in §4.2; and clarified that `ticket`/draw (§4.5/§7) are **optional, non-normative**
> platform extensions, not core token conformance. All additions are protocol-level and
> implementation-neutral.

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

The **core token protocol** is four ops: `deploy`, `buy`, `sell`, `transfer` (§4.1–4.4),
plus the identity/authorization rules in §4.6. Everything a conforming indexer needs to
derive token state lives here. `ticket` (§4.5) is an **optional, non-normative** platform
extension — see its note.

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
{ "p": "zpad-20", "op": "buy", "tick": "ZINU", "cpk": "<claim-key, 32-byte hex>" }
```

The tx sends shielded ZEC to the **curve-reserve** address. The indexer computes
`tokens_out` from the curve (§5) and credits the identity named by `cpk` (§4.6).
Unlike `sell`/`transfer`, a `buy` carries **no signature**: the on-chain payment *is* the
authorization (you cannot pay on someone else's behalf), so the memo only needs to declare
which `cpk` to credit.

> **Security invariant — settle against on-chain value.** A conforming indexer MUST
> compute `tokens_out` from the ZEC the reserve **actually received** in that transaction
> — never from any amount asserted in the memo. The op declares *who* is buying (`cpk`),
> not *how much*; the amount is whatever the chain shows. This is the core defence against
> fake-payment / curve-pump attacks (a memo cannot claim value it did not pay).

### 4.3 `sell` — sell back to the curve

```json
{
  "p": "zpad-20", "op": "sell", "tick": "ZINU", "amt": 32258065,
  "cpk": "<seller claim-key, hex>", "payout": "<address ZEC is sent to>",
  "nonce": 7, "sig": "<signature over the op, see §4.6>"
}
```

`amt` tokens return to the curve; the reserve sends `zec_out` (minus fee) to `payout`.
A sell is **authorized**, not merely asserted: the indexer MUST verify `sig` against
`cpk`, enforce the `nonce` rule (§4.6), and confirm `cpk` holds ≥ `amt` before applying
it. `payout` is inside the signed message, so an observer cannot redirect the proceeds.

### 4.4 `transfer` — move tokens between users

```json
{
  "p": "zpad-20", "op": "transfer", "tick": "ZINU", "amt": 1000000,
  "cpk": "<sender claim-key, hex>", "to": "<recipient claim-key, hex>",
  "nonce": 8, "sig": "<signature over the op, see §4.6>"
}
```

The recipient is named by their **claim-key** (`to`), not a Zcash address — they simply
share their `cpk` to receive. The indexer verifies `sig`/`nonce` and that `cpk` holds
≥ `amt` (§4.6) before moving the balance.

### 4.5 `ticket` — reward ticket (optional, non-normative)

> **Optional platform extension — NOT part of the core token protocol.** `ticket` and the
> draw it feeds (§7) describe a reward primitive used by the reference platform; they are
> **not** required for ZPAD-20 token conformance. A conforming indexer MAY ignore `ticket`
> ops entirely — core token state (`deploy`/`buy`/`sell`/`transfer`) does not depend on
> them. Documented here only so reward-aware implementations agree on the shape.

```json
{ "p": "zpad-20", "op": "ticket", "net": "main", "month": "2026-06", "quest": "trade", "to": "u1..." }
```

A claim minted when a user completes a daily quest. One ticket op per quest per
day; the optional monthly draw (§7) selects winners from claimed tickets.

### 4.6 Identity & authorization (claim-key)

A meta-protocol indexer faces a problem the base chain does not answer: when a buyer
pays from a **shielded** address, the sender is hidden — so *whom* does the indexer
credit, and later, how does it know a `sell`/`transfer` came from the rightful owner?
Pre-ZSA, Zcash addresses also have **no standard message signing**, so the address
itself cannot authorize an op. ZPAD-20 resolves this with a **claim-key**.

- **Identity = a claim-key (`cpk`)** — a public key of a signature scheme (e.g. Ed25519),
  represented as 32-byte hex. It is *not* a Zcash address and holds no ZEC; it exists only
  to key balances and authorize ops. A user MAY derive it deterministically from a secret
  they control, and MAY use a fresh `cpk` per token for unlinkability. Token balances are
  keyed by `cpk`, never by an address.

- **`buy` binds ownership at payment time.** The `cpk` travels in the same note as the
  payment, set by the payer and immutable once mined. The indexer credits that `cpk`. No
  signature is needed — *paying is the act*; an observer can read `cpk` but cannot reassign
  the credit.

- **`sell`/`transfer` carry an authorizing signature.** The op includes `cpk`, a `nonce`,
  and a `sig`. A conforming indexer MUST:
  1. **Verify** `sig` against `cpk` over a canonical message that **binds every
     value-bearing field** of the op (at minimum: op type, tick, amount, destination —
     `payout` for sell / recipient `to` for transfer — and the nonce). Binding all fields
     means no field can be altered after signing (no amount/destination tampering).
  2. **Enforce a strictly-increasing `nonce` per `cpk`** (per token). An op whose nonce is
     ≤ the last applied nonce for that `cpk` is rejected — replay protection. (This
     complements the per-`(txid, index)` single-application rule of §6.)
  3. **Check balance** — `cpk` must hold ≥ `amt`.
  Any failure → the op is **ignored** (consistent with §6's "invalid → ignore, never
  error"), so state stays deterministic.

> The spec fixes *which fields the signature must bind* and *how the nonce behaves*, not a
> specific byte layout — conforming implementations choose the exact canonical encoding and
> signature scheme, as long as any indexer can reproduce the verification.

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
3. Maintain per-token: `raised`, `tokens_sold`, and holder balances (the core state).
   An implementation MAY additionally track platform fee/pool accounting (launch/trade and
   any optional extensions), but that is not required for core token conformance.
4. Derive price, market cap, circulating supply and **holder distribution**
   (counts/concentration only — never expose addresses) from the above.
5. Be **reproducible**: same chain → same state. Publishing the ruleset lets anyone run
   a verifying indexer (no trusted oracle).
6. Apply each op **atomically**: all of an op's ledger effects (balance debit/credit,
   nonce consumption, any payout obligation) commit together or not at all. A partially
   applied op is forbidden — it would break determinism and could move a balance without
   recording the nonce (a replay window) or owe a payout without debiting. Each op is also
   applied **at most once**, keyed by `(txid, output index)`.
7. **Confirmations & reorgs.** A payment is not "settled" until it has the confirmations the
   implementation requires for its value (deeper for larger amounts is recommended). An op
   buried past a **finality depth** is treated as final. On a reorg, roll back to the common
   ancestor and **replay** deterministically; an op whose containing block was orphaned
   inside the finality window MUST be undone (or replay must reproduce the corrected state).
   A conforming indexer MUST NOT continue settling on top of state it can no longer prove.

**Security invariants** (a conforming indexer MUST uphold):
- **Settle against on-chain value, never the memo's claim** (§4.2) — token amounts derive
  from ZEC actually received, not from any figure asserted in the op.
- **Authorize value-moving ops** — `sell`/`transfer` apply only with a valid signature,
  fresh nonce, and sufficient balance (§4.6).

The reference implementation reads full blocks from `zebrad` for transparent ops
and uses a `lightwalletd` viewing key to detect shielded payments to the reserve.

---

## 7. Monthly draw (optional, non-normative)

> **Optional platform extension** tied to the `ticket` op (§4.5) — not part of core token
> conformance. Included to show that *if* an implementation runs a reward draw, it can do so
> verifiably. Implementations without rewards omit this entirely.

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

The **core** fees are Launch, Trade and Graduation; **Ticket claim** and **Featured slot**
belong to optional platform extensions (§4.5/§7) and are listed only for completeness — a
minimal conforming implementation need not charge them. A configured share of each fee MAY
fund a community **prize pool** (sustainable, not minted). Network fees follow ZIP-317
(~0.0001 ZEC/tx) and go to Zcash, not the platform. All fees are shown before confirmation.

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
