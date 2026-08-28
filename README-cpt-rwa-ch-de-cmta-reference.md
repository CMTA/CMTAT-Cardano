# CMTAT Equivalency Assessment — `cpt-rwa-ch-de-cmta-reference` (Cardano, Swiss/German profiles)

> **This is a filled copy of [`CMTAT-equivalency-assessment/README.md`](./CMTAT-equivalency-assessment/README.md) (assessment template `v0.2.0`).** The *implementation being approved* is [`cardano-foundation/cpt-rwa-ch-de-cmta-reference`](https://github.com/cardano-foundation/cpt-rwa-ch-de-cmta-reference) — "Programmable asset tokens on Cardano — German and Swiss profiles", the Aiken (Plutus V3) contracts under `validators/` and `lib/`, at commit **`ff5624e`** (2026-08-24). It is the **successor** to the `security-token` substandard of the CIP-113 Programmable Tokens Platform (assessed in [`README-cip113-security-token.md`](./README-cip113-security-token.md)), which was itself a port of [`fn-bafin-cardano-sc`](./README-fn-bafin-cardano-sc.md). The repository ships **two profiles over one shared codebase**: a *German profile* (eWpG / BaFin) and a *Swiss profile* (CMTA Framework); only the metadata schema and which behaviours are mandatory differ.

## Table of Contents

- [Document Version](#document-version)
- [How to Use This Document](#how-to-use-this-document)
- [General Note](#general-note)
- [Warning](#warning)
- [Relationship to the other assessments in this repository](#relationship-to-the-other-assessments-in-this-repository)
- [Architecture Primer (read first)](#architecture-primer-read-first)
- [CMTAT Function Equivalency Table](#cmtat-function-equivalency-table)
  - [Metadata](#metadata)
  - [Token Attributes](#token-attributes)
    - [Token module](#token-module)
  - [Pause module (mandatory)](#pause-module-mandatory)
    - [Enforcement](#enforcement)
    - [Transfer restriction (optional)](#transfer-restriction-optional)
    - [Access Control](#access-control)
    - [Snapshot (optional)](#snapshot-optional)
    - [Dividend (optional)](#dividend-optional)
    - [Credit Events (optional)](#credit-events-optional)
  - [Debt (optional)](#debt-optional)
- [Guideline for New Blockchain Implementations](#guideline-for-new-blockchain-implementations)
  - [Freeze](#freeze)
  - [CMTAT Extended](#cmtat-extended)
  - [Forced Burn and Forced Transfer](#forced-burn-and-forced-transfer)
  - [Implementation Details](#implementation-details)
  - [Self-Burn](#self-burn)
- [Supplementary features](#supplementary-features)
  - [Lifecycle and governance](#lifecycle-and-governance)
  - [Compliance gating](#compliance-gating)
  - [State integrity](#state-integrity)
  - [Token metadata](#token-metadata)
  - [Operational evidence](#operational-evidence)
- [Reference](#reference)

## Document Version
`v0.2.0` (assessment template)

Note:

- versions with the `rc` suffix are draft versions.
- version before `1.0` are also draft versions

## How to Use This Document
- Use the **CMTAT Function Equivalency Table** as the fillable assessment checklist.
- Use **Guideline for New Blockchain Implementations** as reference guidance when designing or mapping non-Solidity implementations.

## General Note
- The listed functionalities are the **minimal set** required for each module.
- The key words "MUST", "MUST NOT", "REQUIRED", "SHOULD", and "MAY" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/info/rfc2119) and [RFC 8174](https://www.rfc-editor.org/info/rfc8174).

## Warning
An implementation MAY satisfy the CMTAT standard while still failing to meet the criteria required for tokenized shares under Swiss law at the underlying-ledger level. In particular, compliance with CMTAT does not, by itself, demonstrate that decentralization-related legal criteria are satisfied.

> **Additional warnings specific to this implementation.**
>
> 1. **Unaudited.** The repository states plainly that a formal third-party security audit is *planned and not yet completed*, and that the code should be treated as unaudited until it is. Two penetration-testing engagements were carried out by FT Labs on 23 and 26 June 2026 against commits `dd2b754` and `1beeed6`; the minting-proxy upgradability, the `GlobalStateLocation` change and the 2026-08-19/20 internal-review fixes all postdate both, so **neither report covers the assessed code**. 2. **Internal adversarial review, not an audit.** An internal review of the compliance layer found eleven defects (two critical). **Ten were fixed**, each pinned by a test in `validators/regression.ak`; the eleventh — *pause does not stop issuance*, severity medium — was **accepted as intended behaviour rather than fixed**, and is recorded here as the `Mint while pause` row of [Implementation Details](#implementation-details). All eleven are written up in [`documents/security/security-fixes.md`](./cpt-rwa-ch-de-cmta-reference/documents/security/security-fixes.md). 3. This assessment is a **functional-mapping exercise only**. It is not a security review and not a legal opinion. 4. **Two deployment properties no on-chain code can check** (recorded by the implementation itself) materially affect several `y` answers below: the GlobalState NFT must actually land at `global_state_spend_validator`'s address at genesis, and every role credential must name something that genuinely decides (the `must_be_signed_by_credential` helper accepts a script withdrawal, so a permissive script hash in any role field fails **open**, silently). Both must be verified off-chain, after genesis and after every rotation.

## Relationship to the other assessments in this repository

Three Cardano CIP-113 codebases in this lineage have now been assessed. The table records only where the *CMTAT answers* differ; everything not listed is unchanged.

| CMTAT item | `fn-bafin-cardano-sc` | CF `security-token` substandard | **This implementation** |
|---|---|---|---|
| ID 14 — Deactivate contract | n (partial) | n (partial) | **y** — irreversible `DeactivateContract`, admin-signed, requires a paused protocol first |
| ID 1/2/5 — Name, ticker, token id | partial | partial | **y (metadata)** — CIP-68 reference NFT, admin-pinned at registration |
| Sender-KYC toggle | fixed on | fixed on | **admin-toggleable** (`SetRequiresSenderKyc`) |
| Forced transfer while paused | blocked | blocked | **allowed** (deliberate; blocked only once deactivated) |
| Standard burn on a frozen address | n (design-dependent) — expected to fail the sender denylist check, but on a base-layer assumption that assessment could not verify | permitted (divergence) | **not permitted** — matches CMTAT, and now verified against the base layer; must route through the seizure path |
| Burn while paused | design-dependent (no transfer-hook bypass) | permitted | **blocked in practice** by the CIP-113 base layer (see [Implementation Details](#implementation-details)) |
| Upgradeability | n (partial) | n (partial) | **partial** — swappable minting authority + registry-node upgrade + one-way `LockUpgrades` |
| Freeze / unfreeze access control | power user `is_admin` | master admin | **power user `is_admin`** (delegable compliance role) |

## Architecture Primer (read first)

This implementation is **not** an ERC-20-style contract with named methods. It is a set of **Aiken (Plutus V3) validators** operating on Cardano's **eUTXO** ledger, following the **CIP-113 programmable-token** pattern. Understanding the following mapping conventions is required to read the tables:

- **"Validator" here means a Plutus script**, not a node or a stake pool: no network participant approves anything, and the issuer deploys these scripts themselves. The deployment compiles **ten validators across seven source modules** (`global_state`, `power_users` and `denylist` each declare both a `mint` and a `spend` validator; the other four modules declare one each), which is why the repository's parameter-application table has ten rows. CIP-113 itself mandates far less — a registry node naming a minting-logic script and the transfer-logic scripts, which the platform's `dummy` substandard satisfies in a single 446-byte file. The rest is what CMTAT semantics cost on eUTXO: mutable configuration needs the GlobalState UTxO, the absence of a mapping type turns each of the two sets into a linked list (a mint plus a spend validator apiece), and upgradeability needs the proxy/authority split. **One prerequisite is not in this repository:** the CIP-113 base layer (registry + programmable-logic base) must already be deployed on the target network, and `registry_policy_id` is obtained from it rather than compiled here.
- **The token** is a **native Cardano asset**. Its identity is the CIP-113 *issuance policy id* — pinned into every validator at compile time as `expected_issuance_policy_id` — plus a compile-time `security_asset_name: ByteArray`. There is no `name()/symbol()/decimals()/totalSupply()/balanceOf()`; those are ledger-, CIP-68- or off-chain-level concepts on Cardano.
- **Every transfer** triggers a **withdraw-0 "transfer logic" stake validator** (`transfer_logic_validator`) that must succeed for the spend to be valid — this is the CIP-113 equivalent of a CMTAT RuleEngine / transfer hook. Seizures go through a second one (`third_party_transfer_logic_validator`), and mint/burn through a third pair (`minting_logic_validator` → `minting_authority_validator`).
- **Global mutable state** lives in a single **GlobalState NFT-authenticated UTxO** (`GlobalStateDatum`, 14 fields): `transfers_paused`, `deactivated`, `mintable_amount` (remaining supply cap), `admin_credential_hash`, the two linked-list policy ids, `security_info`, the trusted-entity list, `member_root_hash`, `requires_sender_kyc`, `requires_receiver_kyc`, an immutable `network_id`, `minting_script_credential_hash`, and `upgrades_locked`.
- **A permanent proxy + a swappable authority.** `minting_logic_script.ak` is the script whose hash every CIP-113 registry node freezes forever (and from which the issuance policy id derives). It deliberately decides *almost nothing*: it authenticates GlobalState, refuses if `deactivated`, refuses to delegate to itself, and requires the withdraw-0 of whatever script `minting_script_credential_hash` currently names. **Every substantive mint/burn rule** — supply cap, power-user roles, KYC, denylist, destinations, registration structure — lives in `minting_authority.ak`, which the admin can replace in place via `RotateMintingScript`.
- **Three separable authorities** replace OpenZeppelin's single role registry:
  1. **Admin** — one `admin_credential_hash` in `GlobalStateDatum`, rotatable via `RotateAdmin` (dual signature: outgoing + incoming). Gates power-user list mutation, trusted entities, membership root, security info, both KYC toggles, `RotateMintingScript`, `LockUpgrades`, `UpgradeRegistryNode`, CIP-113 registration and `DeactivateContract`.
  2. **Power users** — nodes in the *power-users linked list*, each carrying independent flags: `is_admin` (add/remove denylist entries), `can_mint`, `can_burn`, `can_pause`, `can_force_transfer`. Each privileged action requires that power user's signature **and** that their node still sits at the power-users list address (so revocation actually sticks).
  3. **Trusted entities (KYC providers)** — up to 64 Ed25519 vkeys in `GlobalStateDatum.trusted_entity_vkeys` ("TEL"). They sign **off-chain** KYC attestations; they are never on-chain transactors.
- `utils.must_be_signed_by_credential` accepts **either** a transaction signature (key hash in `extra_signatories`) **or** a script withdrawal keyed by `Script(hash)` — so any authority MAY be a script (multisig / DAO). See Warning 4 for the corresponding footgun.
- **Two linked lists** (`anastasia-labs/aiken-design-patterns` v1.8.0): the **denylist** (presence of a node keyed by a 28-byte credential hash *is* the sanction; absence is proved at transfer time with a covering-node reference input) and the **power-users list**.
- **Deactivation is terminal.** `DeactivateContract` sets `deactivated = True`; a guard at the top of `global_state_spend_validator` then rejects *every* spend of the GlobalState UTxO, so no branch can ever unset it. Every other validator (transfer, seizure, minting proxy, minting authority, denylist, power users) independently refuses to run once the flag is set.

## CMTAT Function Equivalency Table

### Metadata
- Implementation language: **Aiken** (on-chain, Plutus V3 / eUTXO). Off-chain tooling is out of scope for this repository (it ships contracts, `plutus.json` and a KYC test-vector script only).
- Implementation version: Aiken package `ft/rwa` **v0.0.0**; Aiken compiler **v1.1.23**; `plutus = "v3"`; repo commit **`ff5624e`** (2026-08-24). Deps: `aiken-lang/stdlib v3.1.0`, `anastasia-labs/aiken-design-patterns v1.8.0`, `aiken-lang/merkle-patricia-forestry v2.1.0`, `aiken-lang/fuzz v2.2.0`, `keyan-m/aiken-scott-utils v1.4.0`. CIP-113 base-layer guarantees verified against base commit **`018415d`**.
- Reference implementation compared against: **CMTAT v3.2.0** (Solidity) — the repository names it explicitly as its semantic reference, and also records that v3.2.0 has not itself been fully audited: **v3.0.0 is the latest fully audited CMTAT release**.

### Token Attributes
#### Mandatory
| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 1 | Name attribute | ERC20 `name` | Public (`view`) |  | y (metadata) | Admin (CIP-68 metadata authority, pinned at registration) | No `name()` accessor. Two on-chain carriers: the **native asset name** (`security_asset_name`, a compile-time validator parameter) and the **CIP-68 reference NFT** minted once at registration under the same issuance policy at `reference_asset_name` (`(100)` label). Its inline datum is `Constr 0 [metadata map, version, extra]` and holds the human-readable name. `reference_nft_output_is_pinned` forces that NFT's owner to be the GlobalState admin credential at registration, so the admin is the metadata authority by construction. **Datum content is not inspected on-chain.** `SecurityInfo.issuer_name` carries the issuer's legal name separately. |
| 2 | Ticker symbol attribute | ERC20 `symbol` | Public (`view`) |  | y (metadata) | Admin (CIP-68 metadata authority) | Same CIP-68 reference-NFT datum (`ticker`). A metadata update is the admin spending that UTxO and re-outputting it with a new inline datum — an ordinary CIP-113 owner-consent transfer, which therefore still passes the transfer logic's gates (blocked while paused and after deactivation). Not validator-enforced. |
| 3 | Reference to legally required documentation | `terms` | Public (`view`) |  | partial | `ModifySecurityInfo` → admin (`admin_credential_hash`) | No CMTAT `terms` (doc URI + hash) structure. `GlobalStateDatum.security_info` is opaque `Data`; `lib/types/security/bafin.ak` defines the intended `SecurityInfo` shape (`isin`, `term_of_issue`, `crypto_security_register`, `issuer_registration`, `record_keeping`, custodians, …) — the **German profile's**, and the only schema type the repository ships. The Swiss profile defines none on-chain; the field is opaque either way. **Metadata only — parsed by no validator.** Capped at 4 096 CBOR bytes (`constants.max_security_info_bytes`), enforced at genesis and on every update. There is no on-chain document-hash field; a URI/hash would have to be encoded inside the blob or in the CIP-68 datum. |
| 4 | No fractions | ERC20 `decimals` | Public (`view`) | - Decimals must be set to zero unless governing law permits fractions.<br />- CMTAT Solidity allows configurable decimals at deployment | y | Ledger-enforced (integer-only) | Cardano native assets are **integer-only at the ledger level**; there is no on-chain `decimals`. "No fractions" holds by construction. A CIP-68 `decimals` entry would be a display convention only and cannot create on-chain fractional balances. |

For CMTAT reference implementations, decimals SHOULD be configurable rather than defaulting to zero, to support use cases beyond tokenized shares in Switzerland.

##### Note

> On Cardano the ledger does not model decimals; a security token is intrinsically indivisible on-chain, which aligns with the Swiss "no fractions" requirement but means fractionalization (if ever needed) would require a display convention in the CIP-68 datum plus a larger issued supply. `SecurityInfo.nominal_amount` and `volume_of_issuance` document the intended economic denomination.

#### Optional
| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 5 | Token ID attribute | `tokenId` | Public (`view`) | Optional parameter. | y (metadata) | Public (ledger-derived); ISIN via admin `ModifySecurityInfo` | The unique on-chain identity is the **issuance policy id** — a 28-byte hash derived from the permanent minting proxy's hash, and unique per deployment because the GlobalState policy is one-shot-bound to a genesis UTxO (`tx0`/`index0`). Identical source therefore compiles to a different token per deployment. The regulatory identifier is `SecurityInfo.isin`. No standalone `tokenId` accessor. |

For CMTAT reference implementations, `tokenId` SHOULD be included.

##### Note

> The issuance policy id is globally unique and immutable — and, unlike in the predecessor codebases, it is now **pinned at compile time** into the transfer, seizure and minting-authority validators (`expected_issuance_policy_id`) rather than re-derived from a registry node at run time. That change closed a critical defect in which a caller could choose which token the transfer scripts policed (`security-fixes.md` §2).

#### Token module

##### Mandatory

| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 6 | Know total supply | ERC20 `totalSupply` | Public (`view`) |  | partial | Public (on-chain state + ledger query) | No `totalSupply()`. On-chain, `GlobalStateDatum.mintable_amount` is a **remaining-mintable cap** — decremented on mint, credited back on burn, and required to stay `>= 0` by the `MintSecurity` branch. `SecurityInfo.volume_of_issuance` records the intended total. Circulating supply is the sum of on-chain UTxOs, obtainable from any chain indexer. Cap integrity rests on two additional guards: every non-`MintSecurity` branch of the GlobalState spend validator forbids a concurrent security-token mint, and the minting authority refuses any mint or burn that does not **spend** GlobalState. |
| 7 | Know balance | ERC20 `balanceOf` | Public (`view`) |  | partial | Public (ledger query) | No `balanceOf()`. In eUTXO a holder's balance is the sum of the security-token quantity across programmable-base UTxOs whose inline **stake credential** is that holder; queried off-chain via the ledger/indexer. The stake credential *is* the owner identity under CIP-113, and enterprise addresses (no stake credential) are rejected outright by both transfer paths. |
| 8 | Transfer tokens | ERC20 `transfer` | Token holder (`msg.sender`) |  | y | Token holder (owns/spends the UTxO), **gated** by denylist absence on both sides + KYC per flag + not-paused + not-deactivated | `transfer_logic_validator.withdraw` (`transfer_logic_script.ak`) runs on every programmable-token spend. Unique senders and unique destinations are aggregated by `Credential` (a key and a script sharing a hash count as two parties), then vetted in lockstep by `compliance.verify_parties`: **denylist absence always**, **KYC when `requires_sender_kyc` / `requires_receiver_kyc`**. Plus `transfers_paused == False` and `deactivated == False`. A short action list fails closed. |
| 9 | Create tokens | `mint` / `batchMint` | Role-restricted (issuer/minter authorized) |  | y | **Power user with `can_mint`** (must sign; node must still sit at the list address) | `minting_logic_validator.withdraw` (the proxy) delegates to `minting_authority_validator.withdraw`, redeemer `MintBurn` (or `RegisterMint` for the genesis mint riding on CIP-113 registration). Requires: no registry node spent, GlobalState **spent** (so `MintSecurity` decrements the cap), `!deactivated`, only `security_asset_name` minted under the policy, `can_mint` + signature, and **per unique destination** a denylist-absence proof plus receiver KYC when `requires_receiver_kyc`. Batch = a single tx minting to many outputs (eUTXO-native). **One deviation from CMTAT Allowlist, recorded in-code as deliberate:** when a destination is the authorising `can_mint` power user's own list-authenticated credential, the receiver-KYC requirement is met without an attestation. Denylist absence still applies to that operator, so a sanctioned one cannot mint to itself, and every other destination needs a proof. Exact CMTAT Allowlist behaviour is recovered by issuing the treasury credential a KYC attestation at setup. |
| 10 | Cancel tokens | `burn` / `batchBurn` / `burnFrom` | Role-restricted (issuer/burner authorized) | Implementations SHOULD use a dedicated issuer/authorized burn path for forced cancellation scenarios. | y | **Power user with `can_burn`** — plus `can_force_transfer` whenever the holder is sanctioned, uncooperative, or the protocol is paused | Same authority, negative `minted_amount`, gated on `can_burn`. There is **no holder self-burn**. Destroying existing supply *spends a programmable-base UTxO*, so the CIP-113 base layer makes this deployment's **transfer logic run over it** — pause gate, sender denylist proof and sender KYC included. A burn from a frozen holder, or during a pause, must therefore be routed through the **seizure** path (`third_party_transfer_logic_validator`, no source checks, no pause gate), which needs `can_force_transfer` as well. This is the `burnFrom` / forced-cancellation route. |

##### Note

> **How a burn is actually routed, and why cancelling a sanctioned holder's position needs two roles.**
>
> Destroying supply means **spending** a programmable-base UTxO, and the CIP-113 base layer gates every such spend on the programmable-logic global's withdraw-0. For a burn that resolves to `TransferAct`, which requires a token-exists proof for the policy, which makes **this deployment's `transfer_logic_validator` run over the UTxO being burned**. The burn therefore inherits the whole sender-side gate: the pause flag, the sender's denylist-absence proof, and sender KYC when `requires_sender_kyc` is set. Neither base-layer escape applies: `TokenDoesNotExist` needs a registry node covering the policy and a registered policy has none, and `UnfrackingAct` is unavailable because registration pins `unfracking_logic_script` to the empty verification key.
>
> That leaves two routes, and which one is available depends on the holder and the pause state:
>
> - **Standard route** — an unsanctioned, cooperative holder while transfers are unpaused. `transfer_logic_validator` passes, then `minting_authority_validator`'s `MintBurn` branch requires `can_burn` plus that operator's signature, and GlobalState is spent so its `MintSecurity` branch credits the burned amount back to `mintable_amount`. **Role needed: `can_burn`.**
> - **Seizure route** — a sanctioned or uncooperative holder, or any burn during a pause. The transaction takes `third_party_transfer_logic_validator` instead, which performs **no source-side checks** and has **no pause gate**, requiring only `can_force_transfer`, that operator's signature, and at least one spent input carrying the security token. The minting authority still runs and still demands `can_burn`. **Roles needed: `can_force_transfer` *and* `can_burn`.**
>
> The seizure route burns rather than moves because of what the destination scan sees: with a negative mint and no output carrying the security token, the unique-destination list is empty, so there are no destinations to vet and the seizure withdrawal is satisfied by the seized input alone.
>
> Both routes must **spend** GlobalState rather than reference it, since `MintSecurity` is what enforces and restores the supply cap. Conway rejects a transaction that both spends and references the same UTxO, which is why the transfer, seizure and minting redeemers all carry a `GlobalStateLocation` instead of a fixed reference index.
>
> **Practical consequence for role assignment:** grant `can_burn` and `can_force_transfer` together to whoever is expected to execute court-ordered or regulator-ordered cancellations. `can_burn` alone is not sufficient authority to retire a sanctioned holder's position. See also ID 15 (the denylist), ID 17 (forced transfer, including the all-or-nothing-per-UTxO limit) and the [Implementation Details](#implementation-details) table.

#### Optional

| ID   | Requirement | CMTAT Solidity corresponding feature            | Access Control (CMTAT Solidity) | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 11   | Approve     | ERC20 `approve(address spender, uint256 value)` | Token holder                    | Grants a delegate permission to transfer a specific amount of tokens from the token account. This is optional, but implementations SHOULD include it since secondary market capability may depend on delegated approval to automate trading and settlement for regulated entities. Issuers SHOULD consult relevant trading and settlement venues if listing is contemplated. | n | — | **Absent.** eUTXO has no allowance model, and the repository lists delegated approval among the optional CMTA modules it does not implement. There is no `approve`/`transferFrom` analogue; delegation is expressed instead through a script credential at the holder's own address (native/multisig/Plutus), or via the admin **forced transfer** (ID 17). |

##### Note

> Secondary-market settlement is expressed as a single multi-party transaction (atomic swap / DvP) rather than as stored on-chain allowance. Note the practical ceiling documented by the implementation: at 25 % of the shared CIP-113 execution budget, an ordinary transfer supports ≈ 13 unique parties per side denylist-only, ≈ 9 per side with attestation KYC on both sides; the 16 KiB transaction limit binds near 40 attested parties.

### Pause module (mandatory)

| ID   | Requirement         | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity)           | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 12   | Pause tokens        | `pause`                              | Role-restricted (pauser/admin authorized) | Pause must prevent all transfers until `unpause` is called.  | y | **Power user with `can_pause`** (must sign) | `global_state_spend_validator` action `PauseTransfers { transfers_paused: True, .. }`. Enforced by `transfer_logic_validator` (`transfers_paused == False`). **Scope, stated as a product decision:** the pause covers *transfers*; minting stays available; forced transfer (seizure) stays available; burning is blocked in practice because it spends a programmable-base UTxO and so re-enters the paused transfer logic. See [Implementation Details](#implementation-details). The pausing transaction may not move the security token itself (`no_security_token_spent`), closing the pause-and-transfer atomicity bypass. |
| 13   | Unpause tokens      | `unpause`                            | Role-restricted (pauser/admin authorized) |                                                              | y | **Power user with `can_pause`** (must sign) | Same `PauseTransfers` action with `transfers_paused: False`. Not available once `deactivated` — the terminal guard rejects every GlobalState spend, so a deactivated protocol is permanently paused. |
| 14   | Deactivate contract | `deactivateContract`                 | Role-restricted (admin authorized)        | Must permanently disable the token (except in upgradeability patterns where deactivation behavior is explicitly defined). | y | **Admin** (`admin_credential_hash`, must sign) | `DeactivateContract` sets `deactivated = True`, admin-signed and permitted only when the protocol is already paused. Irreversible by construction, and supply need not be zero first. See the note below. |

##### Note

> **Deactivation.** `DeactivateContract` sets `deactivated = True`. Preconditions mirror CMTAT: already paused, admin signature, no security token moving in the same transaction, and exactly that one datum field flipped.
>
> **Irreversible by construction.** A terminal guard at the top of the spend validator (`expect !input_datum.deactivated`) rejects every later spend of the GlobalState UTxO *before* branch dispatch — so no branch can clear the flag, `MintSecurity` can never run again, and the pause can never be lifted. Transfer, seizure, the minting proxy and authority, the denylist and the power-users list each refuse independently once it is set.
>
> Supply is **not** required to be zero, so the CMTA spec's *cancelled* (burn down first) and *immobilised* (freeze in place) variants are both supported.

#### Enforcement

#### Mandatory

| ID   | Requirement | CMTAT Solidity corresponding feature                         | Access Control (CMTAT Solidity)               | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 15   | Freeze      | `freeze` or `setAddressFrozen(true)` *(inferred from extracted PDF text)* | Role-restricted (compliance/admin authorized) | Must block transfers to and from a given address. Single-function implementations are acceptable if they set a frozen status. | y | **Power user with `is_admin`** (must sign); the flag itself is granted/revoked by the GS admin | Implemented as a **denylist** (blocklist), not a per-address flag. `denylist.ak` `mint` redeemer `AddToDenylist { user_pkh, .. }` inserts an ordered linked-list node keyed by a 28-byte credential hash (length-checked, so a malformed key cannot silently lock the sanction out). **Presence blocks both sending and receiving** — both transfer paths require a covering-node absence proof for every party — which is stricter than CMTAT freeze. Deliberately keyed on the **bare hash**, so sanctioning a hash sanctions both the key and script credential forms. Uses two functions (add/remove) rather than one `setAddressFrozen`. Delegating this to `is_admin` rather than the master admin lets a compliance function be granted without handing over protocol control. Unavailable once `deactivated`. |
| 16   | Unfreeze    | `unfreeze` or `setAddressFrozen(false)` *(inferred from extracted PDF text)* | Role-restricted (compliance/admin authorized) | Single-function implementations are acceptable if they clear a frozen status. | y | **Power user with `is_admin`** (must sign) | `RemoveFromDenylist { user_pkh, .. }` removes the node. Same authority and same deactivation gate. |

#### Optional

| ID   | Requirement        | CMTAT Solidity corresponding feature                         | Access Control (CMTAT Solidity)                  | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 17   | Enforce a transfer | `forcedTransfer(address from, address to, uint256 value)`    | Role-restricted (operator/compliance authorized) | Enforcement transfer is performed via `forcedTransfer`.      | y | **Power user with `can_force_transfer`** (must sign) | `third_party_transfer_logic_validator.withdraw`. **No source checks** — the source may be denylisted or KYC-expired; that is the point — but every **destination** must be denylist-absent and, when `requires_receiver_kyc`, KYC-valid. Additionally: at least one spent input must actually carry the security token (so a "seizure" cannot be a no-op), and `deactivated == False`. **Deliberately NOT blocked by a pause** (a divergence from the two predecessor codebases): enforcement availability is judged more important than a literal reading of "prevent all transfers", following the CMTAT reference implementation where `forcedTransfer` calls `_transfer` directly. **Operational limit:** seizure from a sanctioned holder is all-or-nothing per UTxO — the base layer returns a partial seizure's residual to the holder's own address, which then cannot clear the denylist. Drain whole UTxOs; partial seizure is achieved by choosing which UTxOs to spend. |
| 18   | Partial freeze     | `freezePartialTokens(address account, uint256 value)` / `unfreezePartialTokens(address account, uint256 value)` | Role-restricted (operator/compliance authorized) | Intended only to block a sold amount to avoid double-spend during settlement. | n | — | **Absent.** Denylist and pause are all-or-nothing (full address block / full global pause). There is no amount-scoped freeze. A holder can partition their own position across UTxOs, but that is a wallet convention, not an enforced hold. See the note below for why the EVM form has no direct eUTXO equivalent, and for the forced-transfer substitute available here. |

##### Note

> **Why an amount-scoped freeze has no direct eUTXO form.** `freezePartialTokens` works on EVM because a balance is a single integer: the contract keeps a second integer per account and checks `balanceOf(from) - frozenTokens[from] >= amount` on every transfer. Cardano has no balance. A holder's position is a *set* of UTxOs, each independently spendable, and a validator sees only the inputs and outputs of the transaction it is validating, never the holder's other UTxOs. "At most N of your M units may move" is therefore not expressible, because the script does not know M. The obstacle is the missing view of the account, not the arithmetic.
>
> **Three constructions recover the effect on Cardano generally**, none of them implemented here:
>
> - **UTxO partitioning.** Hold the restricted tranche at a different credential (an escrow script, an issuer-controlled sub-account) and the free tranche at the holder's own. The restriction is custody rather than a flag, so it needs the holder's consent or an issuer-side move, and no party can read a "frozen" status off the chain.
> - **Per-holder account state.** An on-chain account node per holder recording total and frozen amounts, which every transfer must spend or reference. This rebuilds account semantics on eUTXO, at the cost of serialising that holder's transfers to roughly one per block, a min-ADA deposit per holder, heavier transfers, and soundness only if every unit is forced through the accounting.
> - **Time-locked tranche.** The locked UTxO's validator demands a validity-range lower bound past the unlock slot. This covers vesting and lock-ups rather than discretionary compliance holds, and is not revocable early.
>
> **The settlement case is partly answered by the ledger.** The template scopes this functionality to "block a sold amount to avoid double-spend during settlement". Under eUTXO the seller's UTxO is an input to the settlement transaction and cannot be spent twice, so double-*settlement* is prevented by the ledger with no freeze flag involved. What that does not prevent is reneging: the seller may still spend the UTxO in a competing transaction, invalidating the settlement rather than double-spending it. Preventing that needs escrow, which is the first construction above.
>
> **The substitute available in this implementation** is a forced transfer of exactly the tranche to an issuer-controlled credential. `third_party_transfer_logic_validator` vets every token-bearing output as a destination (denylist absence always, receiver KYC when the flag is set), and the residual returning to the holder's own address is one such output, which an unsanctioned and KYC-valid holder passes. An operator holding `can_force_transfer` can therefore move N units into escrow and leave the remainder in place, reversing it later with a second forced transfer. **Ordering matters:** this fails against an already-sanctioned holder, whose residual output cannot produce a denylist-absence proof (see ID 17). Place the hold first and sanction second; the reverse order leaves only whole-UTxO seizure.

#### Transfer restriction (optional)

| ID   | Requirement                   | CMTAT Solidity corresponding feature                         | Access Control (CMTAT Solidity)                         | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 19   | Conditional transfer request  | `RuleConditionalTransferLight.detectTransferRestriction(...)` and `approvedCount(...)` | Public (`view`)                                         | Request is represented by a transfer restricted until approval count is non-zero. | n (implicit) | — | **No explicit on-chain request/hold workflow.** Every transfer is *implicitly* conditional: it requires denylist absence for every party and, when the corresponding flag is set, a fresh TTL-bound **trusted-entity KYC attestation** (or MPF membership proof). Pre-approval happens off-chain — a trusted entity signs the attestation — rather than through an on-chain pending-approval queue. |
| 20   | Conditional transfer approval | `RuleConditionalTransferLight.approveTransfer(...)` | Role-restricted (compliance/approver authorized)        | Approval is consumed on transfer via `transferred(...)`; cancellation via `cancelTransferApproval(...)`. | n (implicit) | Trusted entity (off-chain Ed25519 attestation) | The "approval" is the off-chain `AttestationProof`: a 67-byte payload raw-Ed25519-signed by a vkey in the TEL, binding **who** (`user_pkh`), **which credential form** (key vs script — an attestation issued for one is rejected for the other), **tier**, **how long** (`valid_until_ms`, checked against the transaction's validity-range upper bound, which must be `Finite`), **which token** (`security_policy_id`) and **which network** (`network_id`). Validated per transaction; there is no on-chain single-use consumption or cancellation. Revocation is by TEL removal, key rotation, MPF root update, or the denylist — not by cancelling an individual approval, so an attestation stays valid until its TTL. |
| 21   | Assign to whitelist           | CMTAT Allowlist / Rules whitelist setters and `isAddressListed` | Role-restricted for setters; public (`view`) for checks | CMTAT Allowlist and Rules whitelist are alternative whitelist implementations. | y | Admin manages the TEL, the membership root and both toggles; **trusted entities** issue attestations | Two interchangeable mechanisms in `lib/kyc/verify.ak`: (a) **Attestation** — Ed25519 proof by a TEL issuer (`AddTrustedEntity` / `RemoveTrustedEntity` / `UpdateTrustedEntity`, admin-gated, vkeys length-checked and kept sorted without duplicates, max 64); (b) **Membership** — a Merkle-Patricia-Forestry inclusion proof against `member_root_hash` (admin-updated via `UpdateMemberRootHash`; must be empty or exactly 32 bytes), whose leaf binds credential form, hash, TTL, policy id and network identically. **Both sides are independently toggleable**: `SetRequiresSenderKyc` and `SetRequiresReceiverKyc`, each admin-gated. |

##### Note

> The KYC/allowlist model is **capability-based and off-chain-attested** rather than an on-chain address set: a valid, unexpired issuer signature (or MPF membership proof) is presented at transfer time. This keeps per-address on-chain state small and supports tiered KYC (`tier_user` / `tier_institutional` / `tier_vlei`; the validator currently checks only that the tier is not the `0x00` invalid sentinel). `lib/kyc/verify.ak` exports `membership_leaf_key` and `membership_leaf_value` as the **normative** encoders — the off-chain tree builder must produce byte-identical leaves. The denylist (IDs 15/16) is the complementary on-chain blocklist and is checked unconditionally, independent of both KYC flags. One receiver-side exemption is deliberate and on record: a `can_mint` power user minting to their own list-authenticated credential is not asked for an attestation (see ID 9).

#### Access Control

| ID   | Requirement      | CMTAT Solidity corresponding feature                         | Access Control (CMTAT Solidity)                 | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 22   | Grant role       | `grantRole(bytes32 role, address account)` | Role admin (`DEFAULT_ADMIN_ROLE` or role admin) | Used for roles such as `ALLOWLIST_ROLE`, `DEBT_ROLE`, `OPERATOR_ROLE`, `COMPLIANCE_MANAGER_ROLE`. | y | **Admin** (`admin_credential_hash`, must sign) | Roles are power-user nodes. `power_users.ak` `mint` redeemer `AddPowerUser { new_power_user_key, .. }` inserts a `PowerUser { is_admin, can_mint, can_burn, can_pause, can_force_transfer }`. The KYC-issuer role is `AddTrustedEntity` on GlobalState. Both admin-gated, and both refused once `deactivated` (`active_admin_from_ref_input` traps). |
| 23   | Revoke role      | `revokeRole(bytes32 role, address account)`                  | Role admin (`DEFAULT_ADMIN_ROLE` or role admin) | AccessControl role removal.                                  | y | **Admin** (`admin_credential_hash`, must sign) | `RemovePowerUser { .. }` and `RemoveTrustedEntity { vkey }`. Flags can also be re-scoped in place via spend redeemer `ModifyPowerUser { wanted_user_state, .. }`. **Revocation is made to stick** by a change new to this codebase: every consumer authenticates a power-user node against `power_user_list_script_hash`, i.e. requires it to still live at the list's spend address — otherwise a node NFT carried out into a private wallet would keep authenticating whatever flags its holder wrote. The admin itself is rotated via `RotateAdmin`, which requires **both** the outgoing and incoming admin to sign. |
| 24   | Role attribution | `hasRole(...)` / `getRoleAdmin(...)` | Public (`view`)                                 | In CMTAT `AccessControlModule`, `DEFAULT_ADMIN_ROLE` is treated as having all roles. | y | Public (on-chain datum inspection) | Role membership is read on-chain by referencing a power-user node UTxO, or from `GlobalStateDatum.admin_credential_hash`; anyone can inspect these UTxOs. No `hasRole` view function, and **no single super-role**: the admin gates *list and state governance*, the capability flags gate *operations*, and the admin holds none of `can_mint` / `can_burn` / `can_pause` / `can_force_transfer` implicitly. |

##### Note

> Authorization is intentionally decomposed into three authorities (see [Architecture Primer](#architecture-primer-read-first)) rather than a single OpenZeppelin `AccessControl` registry. Unlike in the CF `security-token` substandard, `PowerUser.is_admin` is now **live**: it is the authority that adds and removes denylist entries, deliberately split from the master admin key so a compliance function can be delegated without handing over the protocol. The corresponding hazard is recorded by the implementation itself and repeated here as Warning 4: because `must_be_signed_by_credential` accepts a script withdrawal, any of the five role credentials set to a permissive script hash is satisfiable by a stranger, and the failure is silent and fails **open**. The prescribed control is a post-grant smoke test, per role, asserting that the operation is *rejected* without that operator's signature.

#### Snapshot (optional)
| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 25 | Schedule a snapshot | `scheduleSnapshot(uint256 time)` | Role-restricted | SnapshotEngine `ISnapshotScheduler`. | n | — | **No snapshot module.** Listed by the repository among the optional CMTA modules deliberately not implemented. |
| 26 | Reschedule a snapshot | `rescheduleSnapshot(...)` | Role-restricted |  | n | — | Not implemented. |
| 27 | Unschedule a snapshot | `unscheduleLastSnapshot(...)` | Role-restricted |  | n | — | Not implemented. |
| 28 | Snapshot time | `getAllSnapshots()` / `getNextSnapshots()` | Public (`view`) |  | n | — | Not implemented. |
| 29 | Snapshot total supply | `snapshotTotalSupply(uint256 time)` | Public (`view`) |  | n | — | Not implemented. |
| 30 | Snapshot balance | `snapshotBalanceOf(uint256 time, address tokenHolder)` | Public (`view`) |  | n | — | Not implemented. |
##### Note
> No on-chain snapshot mechanism exists. Historical balances at a given slot can be reconstructed off-chain from the ledger — and the German profile leans on exactly that property, describing every state change as "a ledger transaction, ordered and immutable once settled" — but reconstruction is an indexer capability, not a contract feature, and nothing on-chain freezes a balance set at a point in time.

#### Dividend (optional)

| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 31 | Distribution create parameters |  |  |  | n | — | **No distribution logic.** Only the descriptive `SecurityInfo.dividend_payment_frequency` (and `payment_bond`) metadata fields exist; no create/deposit/claim validators. |
| 32 | Distribution set eligibility |  |  |  | n | — | Not implemented. |
| 33 | Distribution set deposit |  |  |  | n | — | Not implemented. |
| 34 | Distribution claim deposit |  |  |  | n | — | Not implemented. |
| 35 | Distribution schedule |  |  |  | n | — | Not implemented. |
| 36 | Distribution unschedule |  |  |  | n | — | Not implemented. |
##### Note
> No direct CMTAT Solidity equivalent is defined for these; they are implementation-specific (cf. the CMTA IncomeVault prototype). This codebase implements none of them on-chain — distributions are named explicitly among the optional modules left out, and would be a separate future module.

#### Credit Events (optional)
| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 37 | Flag as default | `setCreditEvents(...)` → `flagDefault` | Role-restricted | Managed in `ICMTATCreditEvents.CreditEvents`. | n | — | **No credit-events structure.** No `flagDefault`/`flagRedeemed`/`rating` fields in `GlobalStateDatum`. Could only be represented informally inside the opaque `security_info` blob. |
| 38 | Remove default flag | `setCreditEvents(...)` `flagDefault = false` | Role-restricted |  | n | — | Not implemented. |
| 39 | Flag as redeemed | `setCreditEvents(...)` → `flagRedeemed` | Role-restricted |  | n | — | Not implemented. |
| 40 | Set rating | `setCreditEvents(...)` → `rating` | Role-restricted |  | n | — | Not implemented. |
##### Note
> Credit-event state is not modeled. The admin can rewrite `security_info` (`ModifySecurityInfo`, capped at 4 096 CBOR bytes), which could carry an ad-hoc status, but there is no typed or enforced credit-event field. The nearest *enforced* terminal state is `DeactivateContract` (ID 14), which is closer to redemption/decommissioning than to a default flag.

### Debt (optional)
| ID | Attribute | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 41 | Guarantor identifier | `debt().debtIdentifier.guarantor` | Read public; write role-restricted | Debt module. | n | — | No debt module — the repository names debt terms among the optional CMTA modules not implemented. `SecurityInfo` is an **equity/eWpG** metadata structure. No guarantor field. |
| 42 | Debtholder representative identifier | `debt().debtIdentifier.debtHolder` | Read public; write role-restricted |  | n | — | Not present. |
| 43 | Unique identifier / hash | `tokenId()` and `terms().doc.documentHash` | Public (`view`) |  | partial | Public / admin (`ModifySecurityInfo`) | Unique id = issuance policy id + `SecurityInfo.isin` + `crypto_security_register`. No document-hash field. |
| 44 | Issuance date | `debt().debtInstrument.issuanceDate` | Read public; write role-restricted |  | partial | Admin (`ModifySecurityInfo`) | `SecurityInfo.term_of_issue` (free-form `ByteArray`) can encode issuance terms/date; not a typed date, not validator-enforced. The *ledger* records the genesis transaction's slot, which is the authoritative on-chain issuance moment. |
| 45 | Currency of payments | `debt().debtInstrument.currency` | Read public; write role-restricted |  | n | — | No currency field (`payment_bond` is unrelated free-form metadata). |
| 46 | Par value | `debt().debtInstrument.parValue` | Read public; write role-restricted |  | partial | Admin (`ModifySecurityInfo`) | `SecurityInfo.nominal_amount: Int` is the nominal/par value; metadata only, non-enforcing. |
| 47 | Minimum denomination | `debt().debtInstrument.minimumDenomination` | Read public; write role-restricted |  | n | — | Not present. |
| 48 | Maturity date | `debt().debtInstrument.maturityDate` | Read public; write role-restricted |  | n | — | Not present. |
| 49 | Interest rate | `debt().debtInstrument.interestRate` | Read public; write role-restricted |  | n | — | Not present. |
| 50 | Coupon payment frequency | `debt().debtInstrument.couponPaymentFrequency` | Read public; write role-restricted |  | partial | Admin (`ModifySecurityInfo`) | Only `SecurityInfo.dividend_payment_frequency` (a *dividend*, not coupon, frequency) exists as free-form metadata. |
| 51 | Interest schedule format | `debt().debtInstrument.interestScheduleFormat` | Read public; write role-restricted |  | n | — | Not present. |
| 52 | Interest payment date | `debt().debtInstrument.interestPaymentDate` | Read public; write role-restricted |  | n | — | Not present. |
| 53 | Day count convention | `debt().debtInstrument.dayCountConvention` | Read public; write role-restricted |  | n | — | Not present. |
| 54 | Business day convention | `debt().debtInstrument.businessDayConvention` | Read public; write role-restricted |  | n | — | Not present. |
##### Note
> Both shipped profiles target **equity-style regulated securities** (eWpG / Swiss CO ledger-based securities), so the CMTAT *debt* module is out of scope. The `SecurityInfo` type instead captures equity attributes — `isin`, `security_type`, `security_form`, `security_class`, `record_keeping`, `entry_type`, `without_voting_rights`, `transfer_restrictions`, `third_party_rights`, `company_consent`, `participate_settlement`, `nominal_amount`, `volume_of_issuance`, `issuer_name`, `issuer_registration`, `crypto_security_register`, `share_custodian`, `token_custodian`. All of it is **non-enforcing**: the field is opaque `Data` in the datum, parsed by no validator, and the repository states explicitly that populating it correctly and keeping it consistent with the register is an off-chain duty of the issuer and registrar. A debt profile would need a second `SecurityInfo` schema, which the shared-codebase design accommodates without touching the validators.

## Guideline for New Blockchain Implementations

### Freeze

To be compatible with [ERC-3643](https://eips.ethereum.org/EIPS/eip-3643), freeze is implemented with a single function: `setAddressFrozen(targetAddress, frozenStatus)`.

For non-EVM blockchains, implementations MAY separate this into two distinct functions:

```solidity
freeze(address targetAddress)
unfreeze(address targetAddress)
```

##### Note

> This implementation uses the **two-function** form via the denylist mint validator: `AddToDenylist { user_pkh, .. }` and `RemoveFromDenylist { user_pkh, .. }`, both gated on a power user holding `is_admin` plus their signature. "Frozen status" is **presence of a node** in an ordered linked list keyed by the 28-byte credential hash; absence is proven at transfer time by a *covering-node* reference input. Semantics are stricter than CMTAT's send-side freeze: a denylisted address can **neither send nor receive**, this blocks forced transfers *to* a denylisted destination (though not *from* one — see ID 17), and it now also blocks an ordinary **burn** of that holder's tokens, since a burn re-enters the sender-side transfer logic. Two design points: the denylist keys on the **bare hash**, so sanctioning a hash sanctions both the verification-key and script credential forms (the conservative direction — the KYC path deliberately distinguishes them); and the covering node is authenticated once per *run* of adjacent parties citing it, so builders should keep parties that share a node adjacent in the action lists.

### CMTAT Extended

In the table below, the CMTAT framework extended features are mapped to Solidity features.

| CMTAT Functionalities | CMTAT Solidity corresponding features | CMTAT Allowlist | CMTAT Light | CMTAT Debt | CMTAT Standard | Present in implementation being approved (`y/n`) | Implementation details |
|---|---|---|---|---|---|---|---|
| On-chain snapshot | `snapshotModule` and `snapshotEngine` | ✔ | ✘ | ✔ | ✔ | n | Not implemented (see IDs 25–30). |
| Forced transfer | `forcedTransfer` | ✔ | ✘ | ✔ | ✔ | y | `third_party_transfer_logic_validator` — power user `can_force_transfer`; destinations denylist-absent (+ KYC per flag); source unchecked; **not** blocked by pause; blocked once deactivated; must actually seize at least one token-bearing input. |
| Forced burn | `forcedBurn` | ✘ | ✔ | ✘ | ✘ | y | Negative mint through `minting_authority_validator`, gated on `can_burn`. No holder consent is needed, but a burn is **not** exempt from the compliance gates: it spends a programmable-base UTxO, so the sender-side transfer logic runs over it. This is where the CF `security-token` substandard differed — it short-circuited the transfer hook for mint and burn outright; `fn-bafin-cardano-sc` did not short-circuit either, but its assessment could only infer the consequence, because the base-layer behaviour it depended on was not vendored there. Retiring a sanctioned or uncooperative holder's position therefore requires routing through the seizure path as well, i.e. `can_burn` **and** `can_force_transfer`. |
| Freeze partial token | `freezePartialTokens` / `unfreezePartialTokens` | ✔ | ✘ | ✔ | ✔ | n | Not implemented (see ID 18). |
| Integrated whitelisting/allowlisting | CMTAT Allowlist | ✔ | ✘ | ✘ | ✘ | y | Built-in KYC allowlist: trusted-entity attestations + MPF membership tree, independently toggleable per side, plus the integrated denylist. See IDs 15/16, 21. |
| External whitelisting/allowlisting | CMTAT with rule whitelist | ✘ | ✘ | ✔ | ✔ | y | KYC is attested by **external** trusted entities (off-chain Ed25519 issuers registered in the TEL). The transfer and seizure logic scripts are themselves externally pluggable rule sets over the CIP-113 core, re-pointable via `UpgradeRegistryNode`. |
| RuleEngine / transfer hook | CMTAT with RuleEngine | ✘ | ✘ | ✔ | ✔ | y | The CIP-113 model **is** a transfer hook: `transfer_logic_validator` (withdraw-0) runs on every programmable-token spend and is this deployment's RuleEngine equivalent; `third_party_transfer_logic_validator` and the minting proxy/authority pair are the seizure and issuance equivalents. |
| Upgradeability | CMTAT Upgradeable version | ✔ | ✔ | ✔ | ✔ | partial | **New in this codebase; neither predecessor had any upgrade path at all.** There is no delegatecall on Cardano, but two in-place upgrade paths exist: (a) **the minting authority** — `minting_logic_script.ak` is a deliberately near-empty permanent proxy that delegates to whatever `GlobalStateDatum.minting_script_credential_hash` names, swappable by the admin via `RotateMintingScript` (single-signature by design, so a compromised authority need not co-sign its own replacement); and (b) **the transfer and seizure logic** — re-pointable through the CIP-113 registry node via `UpgradeRegistryNode`, admin-signed, with this deployment's own invariants (`global_state_cs` pinned, unfracking kept forbidden, both fields kept `Script` credentials) re-asserted on the way through. **`LockUpgrades` is a one-way switch that closes both paths permanently.** What remains immutable: this proxy, `global_state_spend_validator`, and the two linked-list validators. |
| Fee payer / gasless | CMTAT with ERC-2771 module | ✔ | ✘ | ✘ | ✔ | n | No ERC-2771 meta-transaction module. Fee sponsorship is achievable off-chain (a sponsor supplies fee and collateral inputs to the user's transaction) but is not codified on-chain. |

##### Note

> **Upgradeability** here is a *proxy-and-registry* model rather than an EVM storage proxy, and the trade-off is stated explicitly by the implementation: because the permanent proxy checks nothing beyond "the named authority ran", a replacement authority inherits the full burden of correctness, and five invariants (no registry node spent during a supply change; GlobalState must be spent; only `security_asset_name` mintable; not deactivated; registration structure) are enforced there or nowhere. `LockUpgrades` is the on-chain form of "this token's rules can no longer change" — meaningful to holders and regulators because an admin-key compromise cannot undo it — at the cost of also giving up the ability to patch a buggy authority. One nuance the implementation records rather than blocks: because `UpgradeRegistryNode` may read GlobalState as a *spent* input, an admin can pair a final upgrade with `LockUpgrades` (or with `RotateAdmin`, or with `DeactivateContract`) in the same transaction and have it see the pre-state. It grants no power the admin did not already have across two transactions, but "locked at slot N" implies "no upgrade *after* N", not "none *at* N". **Gasless/sponsorship** stays off-chain. The KYC proof design is **network-bound** (`network_id`, immutable after genesis) to prevent cross-network attestation replay.

### Forced Burn and Forced Transfer

In the standard burn function, tokens from a frozen wallet MUST NOT be burnable. CMTAT offers `forcedTransfer` to force a transfer or a burn.

If `forcedTransfer` is not available, implementations MAY implement only `forcedBurn` (as in CMTAT Light). Implementations MAY also implement both. In that case, only `forcedBurn` SHOULD burn tokens, and `forcedTransfer` SHOULD NOT burn tokens.

With the CMTAT Solidity version, when `forcedTransfer` is available, `forcedBurn` is not implemented to reduce contract code size. This limitation MAY not apply to other blockchains.

##### Note

> This implementation provides **both**, and satisfies the "tokens from a frozen wallet MUST NOT be burnable in the standard burn function" requirement — which the CF `security-token` substandard did not, and which `fn-bafin-cardano-sc` was only expected to, on an unverified assumption about the base layer. The reason is structural rather than a new check: destroying existing supply spends a programmable-base UTxO, and the CIP-113 base layer gates every such spend on this deployment's transfer logic (via `TransferAct`), which requires a sender-side denylist-absence proof, sender KYC where enabled, and an unpaused protocol. Neither base-layer escape applies — `TokenDoesNotExist` needs a registry node covering the policy and a registered policy has none, and `UnfrackingAct` is unavailable because registration pins `unfracking_logic_script` to the empty vkey.
>
> The consequence is that the two CMTAT paths are **not** cleanly separated the way the guideline suggests. A burn of an *unsanctioned, cooperative* holder's tokens while unpaused is a pure `forcedBurn` (`can_burn`, no destination, does burn). A burn of a *sanctioned or uncooperative* holder's tokens, or any burn during a pause, must be carried on the seizure path — so `forcedTransfer` **does** burn in that shape, and requires `can_force_transfer` in addition to `can_burn`. Grant both roles together to whoever is expected to execute court- or regulator-ordered cancellations. Code size is not a binding constraint on Cardano, so keeping the two validators separate remains intentional.

### Implementation Details

| Functionalities | CMTAT Solidity | Access Control (CMTAT Solidity) | Note | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|
| Mint while pause | ✔ | Role-restricted (minter/issuer authorized) | Dedicated cross-chain mint cannot be performed while paused. | y | Power user `can_mint` | **Allowed by explicit decision.** `verify_mint_or_burn` reads `deactivated` but never `transfers_paused`; a fresh mint spends no programmable-base UTxO, so the transfer logic never runs over it. Follows the CMTAT reference implementation, where `mint` goes through `_update` rather than the pause-checked transfer path. Pinned by the test `mint_succeeds_while_transfers_are_paused_by_design`. Two accepted consequences are documented: position sizes can change while the register is otherwise static (every change still signed, on-chain, role-gated and cap-enforced), and a holder minted to mid-pause cannot move the tokens until the pause lifts — so off-chain procedure must not mint to third parties during a pause. |
| Burn while pause | ✔ | Role-restricted (burner/issuer authorized) | Dedicated cross-chain burn cannot be performed while paused. | n (diverges) | Power user `can_burn` **+** `can_force_transfer` | **Blocked in practice.** The validator itself does not read the pause flag on the burn path, but the burn spends a programmable-base UTxO and the base layer therefore runs this deployment's paused transfer logic over it. The available route during a pause is the seizure path, which has no pause gate and no source-side checks — hence the additional `can_force_transfer` requirement. This corrects the pause exemption as originally documented; see `security-fixes.md`, *Burning needs more authority than `can_burn`*. |
| Self-Burn for everyone | ✘ | Not permitted | Token holders cannot burn their own tokens; only authorized addresses can burn. | n (not permitted) | — | Matches CMTAT: holders cannot self-burn. Every burn requires a `can_burn` power-user signature. |
| Self-Burn for authorized addresses | ✔ | Role-restricted (authorized burner) |  | y | Power user `can_burn` | Authorized burn via `minting_authority_validator` negative mint, with GlobalState spent so the cap is credited back. |
| Standard burn on a frozen address | ✘ | Not permitted in standard burn path | Requires `forcedTransfer` or `forcedBurn`. | n (not permitted) | — | **Matches CMTAT.** The CF `security-token` substandard permitted it, a divergence recorded in its own assessment; `fn-bafin-cardano-sc` was expected to match CMTAT here but could not confirm it. A sanctioned holder cannot produce the denylist-absence proof the burn's sender-side transfer logic demands, so the standard burn path fails; the seizure path is the required route. |
| Burn tokens with `forcedTransfer` | ✔ | Role-restricted (operator/compliance authorized) | See notes above. | y (required in some cases) | Power user `can_force_transfer` + `can_burn` | The seizure withdrawal only requires that at least one *input* carries the security token; a transaction with no token-bearing outputs has no destinations to vet, so combining it with a negative mint burns the seized position. This is the only route to retire a sanctioned holder's tokens, and it diverges from the guideline's "only `forcedBurn` SHOULD burn". |

### Self-Burn

Only the issuer and authorized addresses (not the token holder) can burn a token in CMTAT Solidity, which reflects legal requirements in several jurisdictions.

Once issued, a security can only be cancelled by its issuer, not its holder. Since the token represents the security, the same rule applies. An investor who wants to exit should transfer to the issuer, who can then cancel when legally permitted.

You MAY still add self-burn in your version if it fits your legal or business context.

##### Note

> This implementation **follows the CMTAT legal model**: token holders cannot self-burn. Cancellation is a power-user (`can_burn`) operation only. An exiting investor transfers to an authorized address — itself denylist- and KYC-gated — which then burns. Note that the burning transaction must still clear the sender-side gates for whoever currently holds the tokens, so a cooperative exit is a two-step flow (transfer to treasury, then burn from treasury) rather than a burn directly out of the investor's UTxO.

## Supplementary features

Features present in this implementation beyond the CMTAT baseline, grouped by what they govern.

### Lifecycle and governance

The token has a defined end state and two bounded upgrade paths, each closable on purpose.

- **Irreversible decommissioning switch** (`DeactivateContract`) with a terminal guard that makes the GlobalState UTxO permanently unspendable, satisfying CMTAT ID 14 and supporting both the *cancelled* and *immobilised* readings of the CMTA spec.
- **One-way upgrade lock** (`LockUpgrades`) — an on-chain, admin-compromise-proof commitment that the token's mint and transfer rules are final.
- **Swappable minting authority behind a permanent proxy** (`RotateMintingScript`) — an eUTXO analogue of a proxy upgrade, with the five invariants any replacement must preserve documented in-code.
- **Governed transfer-logic upgrade** (`UpgradeRegistryNode`) that re-asserts this deployment's own invariants over the base layer's freer registry semantics.
- **Rotatable admin with dual-signature handover** (`RotateAdmin`).

### Compliance gating

Who may hold and move the token is decided by off-chain attestations checked on-chain, with each side of a transfer independently controllable.

- **Independently toggleable sender and receiver KYC** (`SetRequiresSenderKyc` / `SetRequiresReceiverKyc`).
- **Tiered, TTL-bound, off-chain KYC attestations** (`tier_user` / `tier_institutional` / `tier_vlei`), verified on-chain via Ed25519 or an MPF membership proof, **network-bound** and **policy-bound** to prevent replay, with the TTL checked against the transaction's validity-range upper bound (a non-`Finite` bound is rejected).
- **Credential-form binding**: a KYC attestation names whether it was issued for a verification key or a script, and is rejected for the other form — closing a holder-identification defect under eWpG §14. The denylist deliberately goes the other way, sanctioning both forms of a hash.

### State integrity

Supply, roles and the GlobalState datum are constrained at genesis and on every mutation, so no single transaction can leave the protocol in a state its rules did not authorise.

- **On-chain supply cap** (`mintable_amount`) enforced atomically with each mint, backed by two structural guards: non-`MintSecurity` branches forbid concurrent security-token mints, and admin actions may not move the security token at all in the same transaction (closing pre-state atomicity bypasses).
- **Revocation that sticks**: every power-user node read is pinned to the list's spend address, so a node NFT removed from the list cannot keep authenticating stale role flags.
- **Datum growth caps** (≤ 64 trusted entities, ≤ 4 096 B `security_info`, ≤ 512 B per entity metadata) enforced at genesis and on every mutation.
- **Genesis sanitisation** of the initial datum: born unpaused, born unlocked, born active, 28-byte credential hashes, in-range `network_id`, well-formed membership root, sorted duplicate-free TEL, and two *distinct* linked-list policy ids both minted in the genesis transaction.
- **Griefing-hardened credential registration**: all four withdraw-0 validators whitelist `RegisterCredential` only, so a third party cannot de-register a stake credential and brick the protocol.

### Token metadata

Two carriers, neither parsed by any validator.

- **CIP-68 reference NFT** for token metadata, minted once at registration and pinned to the admin credential.
- **`SecurityInfo` regulatory metadata** block for eWpG/German and Swiss disclosure.

### Operational evidence

What the repository measures and pins, as distinct from what it enforces.

- **Measured execution budget**: `aiken bench` scaling benchmarks for the transfer and seizure paths, published per-party costs, and conservative per-transaction party maxima at 25 % of the shared CIP-113 budget.
- **Regression suite** (`validators/regression.ak`) pinning every fixed defect, with positive controls paired to each negative test.
- **eUTXO-native batch mint/transfer** — a single transaction can create or move tokens across many holders, subject to the budget limits above.

## Reference

Implementation being approved:

| Component | Repository | Version / Commit |
|---|---|---|
| Contracts (assessed) | https://github.com/cardano-foundation/cpt-rwa-ch-de-cmta-reference | commit `ff5624e` (2026-08-24) |
| Aiken package | `ft/rwa` | v0.0.0; Aiken `v1.1.23`; Plutus V3 |
| CIP-113 specification | https://github.com/HarmonicLabs/CIPs/blob/master/CIP-0113/README.md | Nuzzi, Coppola, Gargiulo, Di Sarro |
| CIP-113 base layer | (deployed base layer) | guarantees verified against commit `018415d` |
| Predecessor (assessed separately) | https://github.com/cardano-foundation/cip113-programmable-tokens-platform | `security-token` substandard, commit `bab6fc8` — see [`README-cip113-security-token.md`](./README-cip113-security-token.md) |
| Upstream origin (assessed separately) | https://github.com/FluidTokens/fn-bafin-cardano-sc | commit `67ab7d9` — see [`README-fn-bafin-cardano-sc.md`](./README-fn-bafin-cardano-sc.md). The CF substandard's `aiken.toml` names this origin as `easy1staking-com/fn-bafin-cardano-sc` (unverified path). |
| Penetration tests | `documents/pentesting/` | FT Labs, 23 and 26 June 2026 (commits `dd2b754`, `1beeed6`) — do **not** cover the assessed commit |
| Internal security review | `documents/security/security-fixes.md` | 11 defects (2 critical), fixed and regression-pinned |

Assessment template submodules (unchanged from the template — the CMTAT Solidity reference this Cardano implementation is compared against):

| Submodule | Repository | Version | Commit |
|---|---|---|---|
| CMTAT | https://github.com/CMTA/CMTAT | `v3.2.0` | `49544f4de1993008acfc9e848d0bf03bd31d8579` |
| SnapshotEngine | https://github.com/CMTA/SnapshotEngine | `v0.3.0-1-g19e0b56` | `19e0b569bf5823aa8cec5760f080a932a9ac940e` |
| RuleEngine | https://github.com/CMTA/RuleEngine | `v3.0.0-rc2-2-g9c0aa70` | `9c0aa70aae08047e4062beab0f89f92bd60252c0` |
| Rules | https://github.com/CMTA/Rules | `v0.3.0` | `91c21c1191e84ff938892267ec443b0d1bb9efb0` |
