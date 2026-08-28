# CMTAT Equivalency Assessment — `fn-bafin-cardano-sc` (upstream BaFin Cardano contracts)

> **This is a filled copy of [`CMTAT-equivalency-assessment/README.md`](./CMTAT-equivalency-assessment/README.md) (assessment template `v0.2.0`).** The *implementation being approved* is **[`FluidTokens/fn-bafin-cardano-sc`](https://github.com/FluidTokens/fn-bafin-cardano-sc)**, the standalone Aiken contract set for BaFin-compliant security tokens on Cardano (package namespace `ft/bafin`). It follows the **Harmonic Labs CIP-113** draft (Michele Nuzzi, Matteo Coppola, Phil DiSarro).
>
> **Provenance note.** The CF substandard's own `aiken.toml` describes itself as "ported from easy1staking-com/fn-bafin-cardano-sc". That string is the only source for that name; it has not been verified to resolve as a GitHub path, and neither this codebase nor the CF platform's Aiken sources mention it anywhere else. The history containing the assessed commit is hosted under **FluidTokens**, which is what this repository's submodule tracks and what is cited above.
>
> This is the **upstream** codebase that the Cardano Foundation CIP-113 `security-token` substandard was ported from (see [`README-cip113-security-token.md`](./README-cip113-security-token.md)). The two are functionally close; the sections below flag the divergences, chiefly around the **denylist access-control model** and the **absence of the platform integration rubber-stamps**.

## Table of Contents

- [Document Version](#document-version)
- [How to Use This Document](#how-to-use-this-document)
- [General Note](#general-note)
- [Warning](#warning)
- [Architecture Primer (read first)](#architecture-primer-read-first)
- [Divergence from the CF CIP-113 substandard](#divergence-from-the-cf-cip-113-substandard)
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

> **Additional warning specific to this implementation.** `fn-bafin-cardano-sc` is a research/reference codebase (`version = "0.0.0"`) with no published external security audit. It targets a **draft** of CIP-113 that is still under active development. This assessment is a *functional-mapping* exercise only; it does not constitute a security review or a legal opinion.

## Architecture Primer (read first)

This implementation is **not** an ERC-20-style contract with named methods. It is a set of **Aiken (Plutus V3) validators** on Cardano's **eUTXO** ledger following the **CIP-113 programmable-token** pattern. The mapping conventions below are needed to read the tables.

- **The token** is a **native Cardano asset**. Its identity is the CIP-113 *issuance policy id* (derived at runtime from the on-chain registry/directory node) plus a compile-time `security_asset_name: ByteArray`. There is no `name()/symbol()/decimals()/totalSupply()/balanceOf()`; those are ledger- or off-chain-level concepts on Cardano. Uniqueness is guaranteed because `owner_credential_hash` and `security_asset_name` are applied as validator parameters, so every deployment yields distinct script hashes and a distinct policy id.
- **Every transfer** runs a **withdraw-0 "transfer logic" stake validator** (`transfer_logic_validator`) that must succeed for the prog-token spend to be valid; this is the CIP-113 counterpart of a CMTAT RuleEngine / transfer hook.
- **Global mutable state** lives in a single **GlobalState NFT-authenticated UTxO** (`GlobalStateDatum`): `transfers_paused`, `mintable_amount` (remaining supply cap), `admin_credential_hash`, power-user and denylist linked-list policy ids, trusted-KYC-issuer list (`trusted_entity_vkeys`), MPF `member_root_hash`, `requires_receiver_kyc`, an opaque BaFin `security_info` blob, and an **immutable `network_id`** (set at genesis, carried forward unchanged by every spend branch, binding every KYC proof to one network to stop cross-network replay).
- **Three separable authorities** replace OpenZeppelin's single role registry:
  1. **Owner / master admin** — one `admin_credential_hash` (28-byte key hash) in `GlobalStateDatum`, rotatable via `RotateAdmin` (dual-signature: old + new). Gates the power-user list lifecycle, the trusted-entity list, `security_info`, the membership root, the receiver-KYC toggle, and list `Deinit` teardown.
  2. **Power users** — nodes in the *power-users linked list*, each a `PowerUser` record with independent flags `is_admin`, `can_mint`, `can_burn`, `can_pause`, `can_force_transfer`. Each privileged operation checks the relevant flag on a referenced node plus that node's signature. Note `is_admin` here is the **sanctions (denylist) authority** — a delegable compliance role, distinct from the master admin key (see Divergence note).
  3. **Trusted entities (KYC providers)** — Ed25519 vkeys in `GlobalStateDatum.trusted_entity_vkeys` ("TEL"). They sign **off-chain** KYC attestations; they are not on-chain transactors.
- `must_be_signed_by_credential` accepts **either** a transaction signatory key hash **or** a script withdrawal keyed by `Script(hash)` — so any authority MAY be a script (multisig / DAO).

## Divergence from the CF CIP-113 substandard

The Cardano Foundation `security-token` substandard is a port of this codebase. The material differences, all confirmed in source, are:

| Aspect | `fn-bafin-cardano-sc` (this, upstream) | CF `security-token` substandard |
|---|---|---|
| Denylist add/remove authority | **Power user with `is_admin`** flag + signature (delegable compliance role) | GS **master admin** (`admin_credential_hash`) |
| `network_id` | **Immutable field in `GlobalStateDatum`**, read as `gs_datum.network_id` | `env` compile-time constant |
| Mint/burn rubber-stamp | **None** — `transfer_logic`/`minting_logic` run their strict path | Adds `is_mint_or_burn` short-circuit (transfer logic returns `True` on mint/burn) |
| Registration rubber-stamp | **None** | Adds `is_registration_tx` short-circuit for platform directory registration |
| Registry-node indices | Original: `minting=2`, `transfer=3`, `third_party=4` | Swapped: `minting=3`, `transfer=2` |
| GS datum handling | Indexed getters/setters (`idx_*`, `replace_data_field` write path) | Whole-datum rebuild per branch |
| Toolchain | Aiken `v1.1.22`, pkg `ft/bafin`, commit `67ab7d9` (2026-06-23) | Aiken `v1.1.21`, pkg `cip113/security-token`, platform commit `bab6fc8` |

The practical consequence of the missing mint/burn rubber-stamp is that this codebase contains **no explicit bypass** of the transfer hook for issuance and redemption. Where the substandard deliberately lets mint and burn skip the pause and denylist checks, the upstream leaves those spends subject to whatever the CIP-113 spend/issuance layer invokes (see the [Implementation Details](#implementation-details) rows).

## CMTAT Function Equivalency Table

### Metadata
- Implementation language: **Aiken** (on-chain, Plutus V3 / eUTXO). Off-chain tooling is out of scope of this repository (contracts only; a `plutus.json` blueprint is shipped).
- Implementation version: package **`ft/bafin` v0.0.0**; Aiken compiler **v1.1.22**; `plutus = "v3"`; repo commit **`67ab7d9`** (`FluidTokens/fn-bafin-cardano-sc`, 2026-06-23). Deps: `aiken-lang/stdlib v3.1.0`, `anastasia-labs/aiken-design-patterns v1.6.0`, `aiken-lang/merkle-patricia-forestry v2.1.0`.

### Token Attributes
#### Mandatory
| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 1 | Name attribute | ERC20 `name` | Public (`view`) |  | partial | Compile-time parameter; off-chain metadata public | No `name()` accessor. The Cardano **native asset name** (`security_asset_name: ByteArray`, a validator parameter) is the on-chain name. Human-readable name/issuer live in `GlobalStateDatum.security_info` (`SecurityInfo.issuer_name`) and MAY be published off-chain (CIP-26/CIP-68). |
| 2 | Ticker symbol attribute | ERC20 `symbol` | Public (`view`) |  | partial | Compile-time parameter; off-chain metadata public | No separate symbol field; the native asset name doubles as the ticker. |
| 3 | Reference to legally required documentation | `terms` | Public (`view`) |  | partial | `ModifySecurityInfo` → master admin | No CMTAT `terms` (doc URI + hash) structure. `security_info` (`SecurityInfo`) carries BaFin fields (`isin`, `term_of_issue`, `crypto_security_register`, `issuer_registration`, `record_keeping`, custodians, …). Metadata only, not validator-enforced; no on-chain document hash field. |
| 4 | No fractions | ERC20 `decimals` | Public (`view`) | - Decimals must be set to zero unless governing law permits fractions.<br />- CMTAT Solidity allows configurable decimals at deployment | y | Ledger-enforced (integer-only) | Cardano native assets are **integer-only at the ledger level**; there is no on-chain `decimals`. "No fractions" holds by construction. |

For CMTAT reference implementations, decimals SHOULD be configurable rather than defaulting to zero, to support use cases beyond tokenized shares in Switzerland.

##### Note

> `SecurityInfo.nominal_amount` and `volume_of_issuance` document the intended economic denomination; on-chain the token is intrinsically indivisible.

#### Optional
| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 5 | Token ID attribute | `tokenId` | Public (`view`) | Optional parameter. | partial | Public (ledger-derived); ISIN via admin `ModifySecurityInfo` | The unique on-chain identity is the **issuance policy id**. A human-facing id (`isin`) is stored in `SecurityInfo`. No standalone `tokenId` accessor. |

For CMTAT reference implementations, `tokenId` SHOULD be included.

##### Note

> The issuance policy id is globally unique and immutable (guaranteed by parameterisation on `owner_credential_hash` + `security_asset_name`), functioning as the de-facto `tokenId`.

#### Token module

##### Mandatory

| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 6 | Know total supply | ERC20 `totalSupply` | Public (`view`) |  | partial | Public (on-chain state + ledger query) | No `totalSupply()`. `GlobalStateDatum.mintable_amount` is a remaining-mintable cap (decremented on mint, incremented on burn); circulating supply is recovered off-chain from the ledger/indexer. `SecurityInfo.volume_of_issuance` records the intended cap. |
| 7 | Know balance | ERC20 `balanceOf` | Public (`view`) |  | partial | Public (ledger query) | No `balanceOf()`. Balance is the sum of the security-token quantity across a holder's UTxOs, queried off-chain. |
| 8 | Transfer tokens | ERC20 `transfer` | Token holder (`msg.sender`) |  | y | Token holder (owns/spends the UTxO), **gated** by KYC proof + denylist-absence + not-paused | `transfer_logic_validator.withdraw` (`transfer_logic_script.ak`) runs on every prog-token spend. Per sender: valid trusted-entity **KYC proof** + **denylist absence**. Per destination: denylist absence (+ KYC if `requires_receiver_kyc`). Plus the global pause gate. |
| 9 | Create tokens | `mint` / `batchMint` | Role-restricted (issuer/minter authorized) |  | y | **Power user with `can_mint`** (must sign) | `minting_logic_validator.withdraw` (`minting_logic_script.ak`). Positive `minted_amount` requires a referenced power-user node that signed the tx and `power_user_data.can_mint`; GS is spent via `MintSecurity`, enforcing `mintable_amount − minted ≥ 0`. Batch mint = one tx, many outputs (eUTXO-native). |
| 10 | Cancel tokens | `burn` / `batchBurn` / `burnFrom` | Role-restricted (issuer/burner authorized) | Implementations SHOULD use a dedicated issuer/authorized burn path for forced cancellation scenarios. | y | **Power user with `can_burn`** (must sign) | Same `minting_logic_validator.withdraw`; negative `minted_amount` requires `power_user_data.can_burn`. There is **no holder self-burn**. Unlike the CF substandard, this codebase adds **no** transfer-hook bypass for burns, so a burn spend is still subject to the transfer logic where the CIP-113 layer invokes it (see [Implementation Details](#implementation-details)). |
#### Optional

| ID   | Requirement | CMTAT Solidity corresponding feature            | Access Control (CMTAT Solidity) | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 11   | Approve     | ERC20 `approve(address spender, uint256 value)` | Token holder                    | Grants a delegate permission to transfer a specific amount of tokens from the token account. This is optional, but implementations SHOULD include it since secondary market capability may depend on delegated approval to automate trading and settlement for regulated entities. Issuers SHOULD consult relevant trading and settlement venues if listing is contemplated. | n | — | **Absent.** eUTXO has no allowance model. No `approve`/`transferFrom` analogue; delegation is expressed through native/multisig scripts at the holder's credential, or via admin **forced transfer** (ID 17). |

##### Note

> DvP / atomic settlement would be expressed as a single multi-party transaction rather than a stored on-chain allowance.

### Pause module (mandatory)

| ID   | Requirement         | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity)           | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 12   | Pause tokens        | `pause`                              | Role-restricted (pauser/admin authorized) | Pause must prevent all transfers until `unpause` is called.  | y | **Power user with `can_pause`** (must sign) | `global_state_spend_validator` action `PauseTransfers { transfers_paused: True, .. }`; requires a power user with `can_pause` and that user's signature. Enforced by `is_paused` in both transfer validators. |
| 13   | Unpause tokens      | `unpause`                            | Role-restricted (pauser/admin authorized) |                                                              | y | **Power user with `can_pause`** (must sign) | Same `PauseTransfers` action with `transfers_paused: False`. |
| 14   | Deactivate contract | `deactivateContract`                 | Role-restricted (admin authorized)        | Must permanently disable the token (except in upgradeability patterns where deactivation behavior is explicitly defined). | n (partial) | Master admin (list `Deinit`); power user `can_pause` (indefinite pause) | **No dedicated permanent kill-switch.** Closest equivalents: an indefinite pause (`can_pause` power user) and master-admin-signed `Deinit` teardown of the power-user / denylist linked-list roots. Validators are immutable scripts. |

#### Enforcement

#### Mandatory

| ID   | Requirement | CMTAT Solidity corresponding feature                         | Access Control (CMTAT Solidity)               | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 15   | Freeze      | `freeze` or `setAddressFrozen(true)` *(inferred from extracted PDF text)* | Role-restricted (compliance/admin authorized) | Must block transfers to and from a given address. Single-function implementations are acceptable if they set a frozen status. | y | **Power user with `is_admin`** (must sign); flag granted by master admin | Implemented as a **denylist** (blocklist). `denylist.ak` `mint` redeemer `AddToDenylist { user_pkh, .. }` inserts a node keyed by the user PKH. Authority is **delegated**: any power user holding `is_admin` may sanction (signed by that user), so a compliance role is grantable without handing over the master admin key. Presence blocks **both sending and receiving** (covering-node absence proofs in both transfer validators) — stronger than CMTAT freeze. |
| 16   | Unfreeze    | `unfreeze` or `setAddressFrozen(false)` *(inferred from extracted PDF text)* | Role-restricted (compliance/admin authorized) | Single-function implementations are acceptable if they clear a frozen status. | y | **Power user with `is_admin`** (must sign) | `denylist.ak` `mint` redeemer `RemoveFromDenylist { user_pkh, .. }`, gated the same as adding. (Root-list `Deinit` is master-admin gated.) |

#### Optional

| ID   | Requirement        | CMTAT Solidity corresponding feature                         | Access Control (CMTAT Solidity)                  | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 17   | Enforce a transfer | `forcedTransfer(address from, address to, uint256 value)`    | Role-restricted (operator/compliance authorized) | Enforcement transfer is performed via `forcedTransfer`.      | y | **Power user with `can_force_transfer`** (must sign) | `third_party_transfer_logic_validator.withdraw`. **No source checks** (source may be denylisted / KYC-expired — the point of a recovery). Each **destination** must be denylist-absent (and KYC'd if `requires_receiver_kyc`); the pause gate still applies. Does not burn (destination required). |
| 18   | Partial freeze     | `freezePartialTokens(...)` / `unfreezePartialTokens(...)` | Role-restricted (operator/compliance authorized) | Intended only to block a sold amount to avoid double-spend during settlement. | n | — | **Absent.** Denylist and pause are all-or-nothing (full address block / full global pause). No amount-scoped freeze. |

#### Transfer restriction (optional)

| ID   | Requirement                   | CMTAT Solidity corresponding feature                         | Access Control (CMTAT Solidity)                         | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 19   | Conditional transfer request  | `RuleConditionalTransferLight.detectTransferRestriction(...)` and `approvedCount(...)` | Public (`view`)                                         | Request is represented by a transfer restricted until approval count is non-zero. | n (implicit) | — | **No explicit on-chain request/hold workflow.** Every transfer is implicitly conditional on a fresh, TTL-bound trusted-entity **KYC attestation** for the sender (and receiver if `requires_receiver_kyc`) plus denylist absence. Pre-approval is off-chain. |
| 20   | Conditional transfer approval | `RuleConditionalTransferLight.approveTransfer(...)` | Role-restricted (compliance/approver authorized)        | Approval is consumed on transfer via `transferred(...)`; cancellation via `cancelTransferApproval(...)`. | n (implicit) | Trusted entity (off-chain Ed25519 attestation) | The "approval" is the off-chain KYC attestation validated per-transaction (`verify_attestation_proof`), bound to `user_pkh`, `security_policy_id`, `network_id`, and a TTL. No on-chain single-use approval/cancellation. |
| 21   | Assign to whitelist           | CMTAT Allowlist / Rules whitelist setters and `isAddressListed` | Role-restricted for setters; public (`view`) for checks | CMTAT Allowlist and Rules whitelist are alternative whitelist implementations. | y | Master admin manages TEL + membership root; **trusted entities** issue attestations | Two allowlist mechanisms in `lib/kyc/verify.ak`: (a) **Attestation** — Ed25519 proof by a TEL issuer (`AddTrustedEntity`/`RemoveTrustedEntity`/`UpdateTrustedEntity`, master-admin gated); (b) **Membership** — MPF inclusion proof against `member_root_hash` (`UpdateMemberRootHash`, master-admin gated). Sender always gated; receiver gated iff `requires_receiver_kyc` (`SetRequiresReceiverKyc`). |

##### Note

> The KYC/allowlist model is capability-based and off-chain-attested, and is bound to the deployment's `network_id` (an immutable GS field here) to stop cross-network replay. The denylist (ID 15/16) is the complementary on-chain blocklist.

#### Access Control

| ID   | Requirement      | CMTAT Solidity corresponding feature                         | Access Control (CMTAT Solidity)                 | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 22   | Grant role       | `grantRole(bytes32 role, address account)` | Role admin (`DEFAULT_ADMIN_ROLE` or role admin) | Used for roles such as `ALLOWLIST_ROLE`, `DEBT_ROLE`, `OPERATOR_ROLE`, `COMPLIANCE_MANAGER_ROLE`. | y | **Master admin** (must sign) | Roles are power-user nodes. `power_users.ak` `mint` redeemer `AddPowerUser { .. }` inserts a `PowerUser { is_admin, can_mint, can_burn, can_pause, can_force_transfer }`. KYC-issuer role = `AddTrustedEntity` on GS. Both master-admin gated. |
| 23   | Revoke role      | `revokeRole(bytes32 role, address account)`                  | Role admin (`DEFAULT_ADMIN_ROLE` or role admin) | AccessControl role removal.                                  | y | **Master admin** (must sign) | `RemovePowerUser { .. }` (power users) and `RemoveTrustedEntity { vkey }` (KYC issuers). Flags re-scoped in place via spend redeemer `ModifyPowerUser { wanted_user_state, .. }`. The master admin itself rotates via `RotateAdmin` (dual-signature old+new). |
| 24   | Role attribution | `hasRole(...)` / `getRoleAdmin(...)` | Public (`view`)                                 | In CMTAT `AccessControlModule`, `DEFAULT_ADMIN_ROLE` is treated as having all roles. | y | Public (on-chain datum inspection) | Role membership is read on-chain by referencing a power-user node UTxO (`get_element_info` / `run_element_with`) or the GS `admin_credential_hash`; anyone can inspect these UTxOs. No `hasRole` view function. |

##### Note

> Authorization is decomposed into three authorities (see [Architecture Primer](#architecture-primer-read-first)). Relative to the CF substandard, this upstream **delegates the sanctions (denylist) authority to power users holding `is_admin`** rather than reserving it for the master admin, which lets a compliance operator be granted without exposing the master key.

#### Snapshot (optional)
| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 25 | Schedule a snapshot | `scheduleSnapshot(uint256 time)` | Role-restricted | SnapshotEngine `ISnapshotScheduler`. | n | — | **No snapshot module.** Not implemented. |
| 26 | Reschedule a snapshot | `rescheduleSnapshot(...)` | Role-restricted |  | n | — | Not implemented. |
| 27 | Unschedule a snapshot | `unscheduleLastSnapshot(...)` | Role-restricted |  | n | — | Not implemented. |
| 28 | Snapshot time | `getAllSnapshots()` / `getNextSnapshots()` | Public (`view`) |  | n | — | Not implemented. |
| 29 | Snapshot total supply | `snapshotTotalSupply(uint256 time)` | Public (`view`) |  | n | — | Not implemented. |
| 30 | Snapshot balance | `snapshotBalanceOf(uint256 time, address tokenHolder)` | Public (`view`) |  | n | — | Not implemented. |
##### Note
> No on-chain snapshot mechanism. Historical balances can be reconstructed off-chain from the ledger/indexer, but this is not part of the contracts.

#### Dividend (optional)

| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 31 | Distribution create parameters |  |  |  | n | — | **No distribution logic.** Only a descriptive `SecurityInfo.dividend_payment_frequency` (and `payment_bond`) metadata field. |
| 32 | Distribution set eligibility |  |  |  | n | — | Not implemented. |
| 33 | Distribution set deposit |  |  |  | n | — | Not implemented. |
| 34 | Distribution claim deposit |  |  |  | n | — | Not implemented. |
| 35 | Distribution schedule |  |  |  | n | — | Not implemented. |
| 36 | Distribution unschedule |  |  |  | n | — | Not implemented. |
##### Note
> No direct CMTAT Solidity equivalent is defined for these; they are implementation-specific (cf. the CMTA IncomeVault prototype). None are implemented on-chain here.

#### Credit Events (optional)
| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 37 | Flag as default | `setCreditEvents(...)` → `flagDefault` | Role-restricted | Managed in `ICMTATCreditEvents.CreditEvents`. | n | — | **No credit-events structure.** No `flagDefault`/`flagRedeemed`/`rating` fields; could only be represented informally inside the free-form `security_info`. |
| 38 | Remove default flag | `setCreditEvents(...)` `flagDefault = false` | Role-restricted |  | n | — | Not implemented. |
| 39 | Flag as redeemed | `setCreditEvents(...)` → `flagRedeemed` | Role-restricted |  | n | — | Not implemented. |
| 40 | Set rating | `setCreditEvents(...)` → `rating` | Role-restricted |  | n | — | Not implemented. |
##### Note
> Credit-event state is not modeled. The master admin can rewrite the opaque `security_info` blob (`ModifySecurityInfo`), which could carry ad-hoc status, but there is no typed/enforced credit-event field.

### Debt (optional)
| ID | Attribute | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 41 | Guarantor identifier | `debt().debtIdentifier.guarantor` | Read public; write role-restricted | Debt module. | n | — | No debt module. `SecurityInfo` is an **equity/BaFin** metadata structure, not a debt instrument. |
| 42 | Debtholder representative identifier | `debt().debtIdentifier.debtHolder` | Read public; write role-restricted |  | n | — | Not present. |
| 43 | Unique identifier / hash | `tokenId()` and `terms().doc.documentHash` | Public (`view`) |  | partial | Public / admin (`ModifySecurityInfo`) | Unique id = issuance policy id + `SecurityInfo.isin` + `crypto_security_register`. No document hash field. |
| 44 | Issuance date | `debt().debtInstrument.issuanceDate` | Read public; write role-restricted |  | partial | Master admin (`ModifySecurityInfo`) | `SecurityInfo.term_of_issue` (free-form) can encode issuance terms; not a typed date, not enforced. |
| 45 | Currency of payments | `debt().debtInstrument.currency` | Read public; write role-restricted |  | n | — | No currency field. |
| 46 | Par value | `debt().debtInstrument.parValue` | Read public; write role-restricted |  | partial | Master admin (`ModifySecurityInfo`) | `SecurityInfo.nominal_amount: Int` is the nominal/par value; metadata only. |
| 47 | Minimum denomination | `debt().debtInstrument.minimumDenomination` | Read public; write role-restricted |  | n | — | Not present. |
| 48 | Maturity date | `debt().debtInstrument.maturityDate` | Read public; write role-restricted |  | n | — | Not present. |
| 49 | Interest rate | `debt().debtInstrument.interestRate` | Read public; write role-restricted |  | n | — | Not present. |
| 50 | Coupon payment frequency | `debt().debtInstrument.couponPaymentFrequency` | Read public; write role-restricted |  | partial | Master admin (`ModifySecurityInfo`) | Only `SecurityInfo.dividend_payment_frequency` (a *dividend*, not coupon, frequency). |
| 51 | Interest schedule format | `debt().debtInstrument.interestScheduleFormat` | Read public; write role-restricted |  | n | — | Not present. |
| 52 | Interest payment date | `debt().debtInstrument.interestPaymentDate` | Read public; write role-restricted |  | n | — | Not present. |
| 53 | Day count convention | `debt().debtInstrument.dayCountConvention` | Read public; write role-restricted |  | n | — | Not present. |
| 54 | Business day convention | `debt().debtInstrument.businessDayConvention` | Read public; write role-restricted |  | n | — | Not present. |
##### Note
> This codebase targets **equity-style regulated securities (BaFin)**, so the CMTAT *debt* module is largely out of scope. `SecurityInfo` instead captures equity attributes (`isin`, `security_type`, `security_form`, `security_class`, `record_keeping`, `entry_type`, `without_voting_rights`, `transfer_restrictions`, `third_party_rights`, `company_consent`, `participate_settlement`, `nominal_amount`, `volume_of_issuance`, `issuer_name`, `issuer_registration`, `crypto_security_register`, `share_custodian`, `token_custodian`) as **non-enforcing** metadata.

## Guideline for New Blockchain Implementations

### Freeze

To be compatible with [ERC-3643](https://eips.ethereum.org/EIPS/eip-3643), freeze is implemented with a single function: `setAddressFrozen(targetAddress, frozenStatus)`.

For non-EVM blockchains, implementations MAY separate this into two distinct functions:

```solidity
freeze(address targetAddress)
unfreeze(address targetAddress)
```

##### Note

> This implementation uses the **two-function** form via the denylist mint validator: `AddToDenylist { user_pkh, .. }` and `RemoveFromDenylist { user_pkh, .. }`, both gated on a **power user holding `is_admin`** plus that user's signature (the master admin grants the flag). "Frozen status" is node presence in an ordered linked list; absence is proven at transfer time by a covering-node reference input (`verify_denylist_absence`). A denylisted address can **neither send nor receive**, and forced transfers *to* a denylisted destination are also blocked (forced transfers *from* one are allowed — ID 17).

### CMTAT Extended

In the table below, the CMTAT framework extended features are mapped to Solidity features.

| CMTAT Functionalities | CMTAT Solidity corresponding features | CMTAT Allowlist | CMTAT Light | CMTAT Debt | CMTAT Standard | Present in implementation being approved (`y/n`) | Implementation details |
|---|---|---|---|---|---|---|---|
| On-chain snapshot | `snapshotModule` and `snapshotEngine` | ✔ | ✘ | ✔ | ✔ | n | Not implemented (IDs 25–30). |
| Forced transfer | `forcedTransfer` | ✔ | ✘ | ✔ | ✔ | y | `third_party_transfer_logic_validator` — power user `can_force_transfer`; destination KYC + denylist-absent; source unchecked; respects pause. Does not burn. |
| Forced burn | `forcedBurn` | ✘ | ✔ | ✘ | ✘ | y (design-dependent) | Burn is a power-user (`can_burn`) negative mint, distinct from `forcedTransfer`. Whether it can cancel a **denylisted** holder's tokens depends on the CIP-113 spend layer: unlike the CF substandard, this codebase does not short-circuit the transfer hook for burns, so a burn spend the hook governs would require the source to pass KYC + denylist absence (implying seize-then-burn for sanctioned holders). |
| Freeze partial token | `freezePartialTokens` / `unfreezePartialTokens` | ✔ | ✘ | ✔ | ✔ | n | Not implemented (ID 18). |
| Integrated whitelisting/allowlisting | CMTAT Allowlist | ✔ | ✘ | ✘ | ✘ | y | Built-in KYC allowlist (trusted-entity attestations + MPF membership tree) plus the integrated denylist. |
| External whitelisting/allowlisting | CMTAT with rule whitelist | ✘ | ✘ | ✔ | ✔ | y | KYC is attested by **external** trusted entities (off-chain Ed25519 issuers in the TEL). |
| RuleEngine / transfer hook | CMTAT with RuleEngine | ✘ | ✘ | ✔ | ✔ | y | The CIP-113 model **is** a transfer hook: `transfer_logic_validator` (withdraw-0) runs on every prog-token spend. |
| Upgradeability | CMTAT Upgradeable version | ✔ | ✔ | ✔ | ✔ | n (partial) | Plutus validators are **immutable** (no proxy). Mutable *state* is the GS datum (roles/KYC-issuers/pause/sanctions/cap), and the admin is rotatable (`RotateAdmin`). Changing logic means new scripts + registry re-pointing. |
| Fee payer / gasless | CMTAT with ERC-2771 module | ✔ | ✘ | ✘ | ✔ | n | No ERC-2771 meta-transaction module. Fee sponsorship would be handled off-chain at tx-building time. |

##### Note

> Upgradeability follows the eUTXO model (immutable logic + mutable NFT-authenticated datum + registry re-pointing). KYC proofs are **network-bound** via the immutable GS `network_id` field to prevent cross-network replay.

### Forced Burn and Forced Transfer

In the standard burn function, tokens from a frozen wallet MUST NOT be burnable. CMTAT offers `forcedTransfer` to force a transfer or a burn.

If `forcedTransfer` is not available, implementations MAY implement only `forcedBurn` (as in CMTAT Light). Implementations MAY also implement both. In that case, only `forcedBurn` SHOULD burn tokens, and `forcedTransfer` SHOULD NOT burn tokens.

With the CMTAT Solidity version, when `forcedTransfer` is available, `forcedBurn` is not implemented to reduce contract code size. This limitation MAY not apply to other blockchains.

##### Note

> This implementation provides **`forcedTransfer`** (`third_party_transfer_logic_validator`, destination required, does not burn, `can_force_transfer`) and a **burn** path (`can_burn` negative mint, does burn). Consistent with the CMTAT guideline, only the burn path cancels tokens and the forced transfer only moves them. Note the divergence from the CF substandard: this upstream does not carve out a transfer-hook bypass for the burn path, so cancelling a sanctioned holder's tokens is expected to go through seize-then-burn rather than a direct burn on a frozen address.

### Implementation Details

| Functionalities | CMTAT Solidity | Access Control (CMTAT Solidity) | Note | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|
| Mint while pause | ✔ | Role-restricted (minter/issuer authorized) | Dedicated cross-chain mint cannot be performed while paused. | design-dependent | Power user `can_mint` | `minting_logic` and the GS `MintSecurity` branch do not read `transfers_paused`. Unlike the CF substandard, there is **no** transfer-hook bypass, so if the CIP-113 layer invokes `transfer_logic` on the mint output, the recipient must be denylist-absent and the tx not-paused. Net behavior depends on the issuance-layer invocation. |
| Burn while pause | ✔ | Role-restricted (burner/issuer authorized) | Dedicated cross-chain burn cannot be performed while paused. | design-dependent | Power user `can_burn` | Same reasoning: no explicit bypass. A burn spend governed by the transfer hook would be subject to the sender KYC/denylist checks and the pause gate. |
| Self-Burn for everyone | ✘ | Not permitted | Token holders cannot burn their own tokens; only authorized addresses can burn. | n (not permitted) | — | Holders cannot self-burn; every burn requires a `can_burn` power-user signature. |
| Self-Burn for authorized addresses | ✔ | Role-restricted (authorized burner) |  | y | Power user `can_burn` | Authorized burn via `minting_logic_validator` negative mint. |
| Standard burn on a frozen address | ✘ | Not permitted in standard burn path | Requires `forcedTransfer` or `forcedBurn`. | n (design-dependent) | — | With no transfer-hook bypass, a standard burn on a denylisted holder is expected to fail the sender denylist-absence check (matching CMTAT), so seize (force transfer) then burn is the intended flow. |
| Burn tokens with `forcedTransfer` | ✔ | Role-restricted (operator/compliance authorized) | See notes above. | n | — | `forcedTransfer` here (`third_party_transfer_logic`) cannot burn — it requires a destination output. Burning is the separate `can_burn` path. |

### Self-Burn

Only the issuer and authorized addresses (not the token holder) can burn a token in CMTAT Solidity, which reflects legal requirements in several jurisdictions.

Once issued, a security can only be cancelled by its issuer, not its holder. Since the token represents the security, the same rule applies. An investor who wants to exit should transfer to the issuer, who can then cancel when legally permitted.

You MAY still add self-burn in your version if it fits your legal or business context.

##### Note

> This implementation **follows the CMTAT legal model**: token holders cannot self-burn; cancellation is a `can_burn` power-user operation only. An exiting investor transfers to an authorized address (KYC/denylist gated), which then burns.

## Supplementary features

> Features present in this implementation beyond the CMTAT baseline:
> - **Tiered, TTL-bound, off-chain KYC attestations** (`tier_user` / `tier_institutional` / `tier_vlei`), verified on-chain via Ed25519 (`AttestationProof`) or MPF membership proof (`MembershipProof`), **network-bound** via the immutable GS `network_id` field.
> - **Delegable sanctions role** — a power user with `is_admin` can add/remove denylist entries without the master admin key.
> - **Toggleable receiver-KYC** (`SetRequiresReceiverKyc`).
> - **Rotatable master admin with dual-signature handover** (`RotateAdmin`).
> - **On-chain supply cap** (`mintable_amount`) enforced atomically with each mint via the GlobalState UTxO.
> - **eUTXO-native batch mint/transfer.**
> - **Griefing-hardened credential registration** — the withdraw-0 stake credentials whitelist `RegisterCredential` only (reject `UnregisterCredential`).
> - **BaFin `SecurityInfo` regulatory metadata** block for German/EU regulated-securities disclosure.

## Reference

Implementation being approved:

| Component | Repository | Version / Commit |
|---|---|---|
| `fn-bafin-cardano-sc` | https://github.com/FluidTokens/fn-bafin-cardano-sc | commit `67ab7d9` (2026-06-23); pkg `ft/bafin` v0.0.0; Aiken `v1.1.22`; Plutus V3. CF's `aiken.toml` names the origin as `easy1staking-com/fn-bafin-cardano-sc` (unverified path). |
| CIP-113 draft followed | https://github.com/HarmonicLabs/CIPs/tree/master/CIP-meta-assets%20(ERC20-like%20assets) | Harmonic Labs (Nuzzi, Coppola, DiSarro) |
| Derived CF substandard (for comparison) | https://github.com/cardano-foundation/cip113-programmable-tokens-platform | `src/substandards/security-token/` |

Assessment template submodules (unchanged from the template — the CMTAT Solidity reference this Cardano implementation is compared against):

| Submodule | Repository | Version | Commit |
|---|---|---|---|
| CMTAT | https://github.com/CMTA/CMTAT | `v3.2.0` | `49544f4de1993008acfc9e848d0bf03bd31d8579` |
| SnapshotEngine | https://github.com/CMTA/SnapshotEngine | `v0.3.0-1-g19e0b56` | `19e0b569bf5823aa8cec5760f080a932a9ac940e` |
| RuleEngine | https://github.com/CMTA/RuleEngine | `v3.0.0-rc2-2-g9c0aa70` | `9c0aa70aae08047e4062beab0f89f92bd60252c0` |
| Rules | https://github.com/CMTA/Rules | `v0.3.0` | `91c21c1191e84ff938892267ec443b0d1bb9efb0` |
