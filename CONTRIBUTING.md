# Contributing to ZPAD-20

ZPAD-20 is an open standard for the Zcash ecosystem. Feedback, review and independent
implementations are welcome.

## How to contribute

- **Spec feedback / discussion:** open an issue, or join the conversation on the
  [Zcash Community Forum](https://forum.zcashcommunity.com/).
- **Corrections / clarifications:** open a pull request against `SPEC.md`. Keep changes
  precise and rationale-backed; the goal is that any implementer derives the **same**
  state from the same rules.
- **Implementations:** if you build an indexer or wallet against ZPAD-20, open an issue so
  we can link it here.

## Principles

- **Determinism first.** Any change must preserve "same chain → same state." Invalid or
  ambiguous ops must be ignored, never errored.
- **Privacy by default.** Implementations expose only aggregates (counts, concentration),
  never per-wallet surveillance.
- **Forward-compatible.** Indexers ignore unknown `op`/fields; breaking changes bump a
  future protocol tag (e.g. `zpad-21`).

## Versioning

`p` is fixed as `zpad-20`. This repository tracks the canonical specification; a path to a
formal ZIP for broader community review is intended.

By contributing you agree your contributions are licensed under the repository's
[MIT License](./LICENSE).
