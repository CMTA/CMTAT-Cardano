# CMTAT on Cardano

Working repository for the analysis of **CMTAT-equivalent security tokens on Cardano**.

[CMTAT](https://github.com/CMTA/CMTAT) (CMTA Token) is the security-token framework published by the Capital Markets and Technology Association for tokenizing regulated financial instruments. The framework is blockchain-agnostic, but its reference implementation is Solidity and assumes an EVM: a token is a contract with methods, roles live in an OpenZeppelin role registry, and transfer rules are delegated to an external `RuleEngine`. That is the version these assessments compare against. Cardano offers none of it. Value settles in unspent outputs, scripts are immutable once deployed, and there is no `msg.sender`.

Three Cardano codebases have nonetheless built the CMTAT feature set on top of [CIP-113](https://github.com/HarmonicLabs/CIPs/blob/master/CIP-0113/README.md) (programmable tokens). This repository holds, for each of them, a **filled CMTA equivalency assessment** mapping the implementation function by function against the CMTAT checklist, plus two **articles** that explain how the mapping works and where the two models pull apart.

Nothing here is a security audit or a legal opinion. The assessments are functional mappings, and each one carries the audit status of the code it describes.

Parts of this project were written with the help of the AI coding assistant Claude Code (Anthropic).

---

## The three codebases assessed

They form a lineage, oldest first. Each was assessed independently against the same template.

| # | Codebase | What it is | Assessment |
|---|---|---|---|
| 1 | [`FluidTokens/fn-bafin-cardano-sc`](https://github.com/FluidTokens/fn-bafin-cardano-sc) | The original BaFin-oriented Aiken implementation (package `ft/bafin`), following the Harmonic Labs CIP-113 design. The CF substandard's `aiken.toml` names the origin as `easy1staking-com/fn-bafin-cardano-sc`, a path this repository has not verified. | [`README-fn-bafin-cardano-sc.md`](./README-fn-bafin-cardano-sc.md) |
| 2 | [`cardano-foundation/cip113-programmable-tokens-platform`](https://github.com/cardano-foundation/cip113-programmable-tokens-platform) | The Cardano Foundation's `security-token` substandard (`src/substandards/security-token/`), a port of the above into the CIP-113 platform. | [`README-cip113-security-token.md`](./README-cip113-security-token.md) |
| 3 | [`cardano-foundation/cpt-rwa-ch-de-cmta-reference`](https://github.com/cardano-foundation/cpt-rwa-ch-de-cmta-reference) | The standalone successor, shipping a German (eWpG) and a Swiss (CMTA) profile over one shared set of validators. | [`README-cpt-rwa-ch-de-cmta-reference.md`](./README-cpt-rwa-ch-de-cmta-reference.md) |

Where the CMTAT answers differ between the three is tabulated in the third assessment, under *Relationship to the other assessments in this repository*. The short version: the newest codebase is the first to have a permanent deactivation switch, the first with a real upgrade path, and the first in which a sanctioned holder's tokens cannot be burned through the standard path.

---

## Files in this repository

### Equivalency assessments

Each is a filled copy of the CMTA template (`v0.2.0`), covering all 54 numbered items plus the guideline sections, with the implementation pinned to a specific commit.

| File | Covers | Pinned at |
|---|---|---|
| [`README-cpt-rwa-ch-de-cmta-reference.md`](./README-cpt-rwa-ch-de-cmta-reference.md) | `cpt-rwa-ch-de-cmta-reference` (Swiss + German profiles) | commit `ff5624e`, 2026-08-24 |
| [`README-cip113-security-token.md`](./README-cip113-security-token.md) | CIP-113 platform, `security-token` substandard | commit `bab6fc8`, 2026-06-05 |
| [`README-fn-bafin-cardano-sc.md`](./README-fn-bafin-cardano-sc.md) | `fn-bafin-cardano-sc` | commit `67ab7d9`, 2026-06-23 (the submodule tracks `0b9641d`, five commits later) |

The blank template itself lives in the submodule, at [`CMTAT-equivalency-assessment/README.md`](./CMTAT-equivalency-assessment/README.md).

### Articles

| File | Subject |
|---|---|
| [`2026-07-17-cmtat-cardano-cip113.md`](./2026-07-17-cmtat-cardano-cip113.md) | How the CIP-113 `security-token` substandard reconstructs CMTAT on eUTXO: the validator set, the three authorities, the compliance gate, and the equivalency result. |
| [`2026-08-26-cardano-cmta-swiss-german-profiles.md`](./2026-08-26-cardano-cmta-swiss-german-profiles.md) | What the successor codebase changed: the permanent proxy and swappable minting authority, the one-way upgrade lock and terminal deactivation state, and two behaviours that emerge from composing the deployment with the CIP-113 base layer. |
| [`2026-08-31-cardano-absence-proof-sanctions-list.md`](./2026-08-31-cardano-absence-proof-sanctions-list.md) | How a party proves it is *not* on the sanctions list: the sorted linked list, the covering-node absence proof, why a caller-supplied reference input is not a trust hole, and what a transaction builder must do to satisfy it. |

Both are written for this repository and are not published elsewhere. Each embeds its diagrams as PNGs from `assets/`.

### Diagrams

PlantUML sources and rendered PNGs live under `assets/article/blockchain/cardano/`. [`tree.txt`](./tree.txt) is the registry mapping each article to the `.puml` files that produced its figures, so a later edit can find the source rather than reverse-engineer the image.

Render one with:

```sh
plantuml -tpng assets/article/blockchain/cardano/<name>.puml
```

### Submodules

| Path | Repository | Why it is here |
|---|---|---|
| `CMTAT-equivalency-assessment/` | [CMTA/CMTAT-equivalency-assessment](https://github.com/CMTA/CMTAT-equivalency-assessment) | The blank assessment template, and the CMTAT / RuleEngine / Rules / SnapshotEngine Solidity sources it pins as the reference. |
| `cip113-programmable-tokens-platform/` | [cardano-foundation/cip113-programmable-tokens-platform](https://github.com/cardano-foundation/cip113-programmable-tokens-platform) | Codebase 2, the `security-token` substandard. |
| `cpt-rwa-ch-de-cmta-reference/` | [cardano-foundation/cpt-rwa-ch-de-cmta-reference](https://github.com/cardano-foundation/cpt-rwa-ch-de-cmta-reference) | Codebase 3, the Swiss and German reference profiles. |
| `fn-bafin-cardano-sc/` | [FluidTokens/fn-bafin-cardano-sc](https://github.com/FluidTokens/fn-bafin-cardano-sc) | Codebase 1, the upstream origin. |

Clone with the sources attached:

```sh
git clone --recurse-submodules <this-repo>
# or, in an existing clone:
git submodule update --init --recursive
```

---

## Reading order

If you are new to the subject, start with the first article for the eUTXO and CIP-113 background, then read the second for what the current codebase does differently. If you only need the compliance answer for one implementation, go straight to its assessment; each is self-contained and opens with an architecture primer explaining the mapping conventions used in its tables.

## Audit status of the assessed code

Stated plainly, because every assessment repeats it and it bears on how the results should be read:

- **`cpt-rwa-ch-de-cmta-reference`** — a formal third-party audit is planned and **not yet completed**. Two FT Labs penetration tests (23 and 26 June 2026) predate the assessed commit. An internal adversarial review found and fixed eleven defects, two of them critical, each pinned by a regression test.
- **CIP-113 platform** — flagged by its own authors as research and development, not production ready, pending a professional security audit.
- **`fn-bafin-cardano-sc`** — no audit recorded.

The CMTAT Solidity reference itself is pinned at `v3.2.0`, which has not been fully audited; the latest fully audited CMTAT release is `v3.0.0`.

## Reference

- [CMTAT](https://github.com/CMTA/CMTAT) and the [CMTAT Equivalency Assessment](https://github.com/CMTA/CMTAT-equivalency-assessment) (CMTA)
- [CIP-113 — Programmable Tokens](https://github.com/HarmonicLabs/CIPs/blob/master/CIP-0113/README.md) (Harmonic Labs)
- [CIP-68 — Datum Metadata Standard](https://cips.cardano.org/cip/CIP-0068)
- [ERC-3643 — T-REX permissioned tokens](https://eips.ethereum.org/EIPS/eip-3643)
- [Aiken](https://aiken-lang.org/) and the [Anastasia Labs design patterns](https://github.com/Anastasia-Labs/aiken-design-patterns)
