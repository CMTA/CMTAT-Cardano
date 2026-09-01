# CMTAT Equivalency Assessment — Cardano CIP-113 `security-token` Substandard

> **This is a filled copy of [`CMTAT-equivalency-assessment/README.md`](./CMTAT-equivalency-assessment/README.md) (assessment template `v0.2.0`).** The *implementation being approved* is the **`security-token`** substandard of the Cardano Foundation **CIP-113 Programmable Tokens Platform**, i.e. the Aiken validators under `cip113-programmable-tokens-platform/src/substandards/security-token/`. It is a port of [`FluidTokens/fn-bafin-cardano-sc`](https://github.com/FluidTokens/fn-bafin-cardano-sc) targeting the German BaFin / Swiss regulated-securities use case. The substandard's own `aiken.toml` records the origin as "ported from easy1staking-com/fn-bafin-cardano-sc"; that path is unverified, and the history containing the assessed upstream commit is hosted under FluidTokens.

## Table of Contents

- [Document Version](#document-version)
- [How to Use This Document](#how-to-use-this-document)
- [General Note](#general-note)
- [Warning](#warning)
- [Summary](#summary)
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
- [Reference](#reference)

## Document Version
`v0.2.0` (this assessment), filled from CMTA assessment template `v0.2.0`

Note:

- versions with the `rc` suffix are draft versions.
- version before `1.0` are also draft versions

Both numbers are below `1.0`, so by the rule above this assessment and the template it follows are each still a draft.

## How to Use This Document
- Use the **CMTAT Function Equivalency Table** as the fillable assessment checklist.
- Use **Guideline for New Blockchain Implementations** as reference guidance when designing or mapping non-Solidity implementations.

## General Note
- The listed functionalities are the **minimal set** required for each module.
- The key words "MUST", "MUST NOT", "REQUIRED", "SHOULD", and "MAY" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/info/rfc2119) and [RFC 8174](https://www.rfc-editor.org/info/rfc8174).

## Warning
An implementation MAY satisfy the CMTAT standard while still failing to meet the criteria required for tokenized shares under Swiss law at the underlying-ledger level. In particular, compliance with CMTAT does not, by itself, demonstrate that decentralization-related legal criteria are satisfied.

> **Additional warning specific to this implementation.** The CIP-113 platform is flagged by its own authors as **Research & Development / not production-ready / pending professional security audit**. This assessment is a *functional-mapping* exercise only; it does not constitute a security review or a legal opinion.

Parts of this project, including this document, were written with the help of the AI coding assistant Claude Code (Anthropic).

## Summary

| Answer | Mandatory (17) | Optional (37) |
|---|---:|---:|
| Present (`y`) | 11 | 2 |
| Partial | 5 | 5 |
| Absent (`n`) | 1 | 30 |

**One mandatory item is not met:** ID 14, deactivate contract, answered `n (partial)` — an indefinite pause and a list teardown exist, but no irreversible decommissioning switch. The five mandatory partials are name, ticker, legal-documentation reference, total supply and balance, each a consequence of eUTXO having no accounts rather than a missing feature.

Of the 30 absent optional items, 26 belong to four modules the codebase does not implement: snapshots, distributions, credit events and debt terms. The remaining four are the allowance model (ID 11) and partial freeze (ID 18), neither of which has an eUTXO equivalent, and the conditional-transfer pair (IDs 19 and 20), where approval is an off-chain attestation rather than an on-chain queue. Forced transfer and a KYC allowlist are present.

Read the [Warning](#warning) before relying on any answer: this code is unaudited, and several behaviours depend on the CIP-113 base layer rather than on this codebase alone.

## Architecture Primer (read first)

This implementation is **not** an ERC-20-style contract with named methods. It is a set of **Aiken (Plutus V3) validators** operating on Cardano's **eUTXO** ledger, following the **CIP-113 programmable-token** pattern. Understanding the following mapping conventions is required to read the tables:

- **The token** is a **native Cardano asset**. Its identity is the CIP-113 *issuance policy id* (derived at runtime from the on-chain registry/directory node) plus a compile-time `security_asset_name: ByteArray`. There is no `name()/symbol()/decimals()/totalSupply()/balanceOf()` — those are ledger- or off-chain-level concepts on Cardano.
- **Every transfer** triggers a **withdraw-0 "transfer logic" stake validator** (`transfer_logic_validator`) that must succeed for the spend to be valid — this is the CIP-113 equivalent of a CMTAT RuleEngine / transfer hook.
- **Global mutable state** lives in a single **GlobalState NFT-authenticated UTxO** (`GlobalStateDatum`): pause flag, remaining-mintable supply cap, admin key hash, trusted-KYC-issuer list, denylist/power-user policy ids, membership root, and BaFin `security_info` metadata.
- **Three separable authorities** replace OpenZeppelin's single role registry:
  1. **Admin** — one `admin_credential_hash` (28-byte key hash) in `GlobalStateDatum`, rotatable via `RotateAdmin` (dual-signature: old + new). Gates list mutations (denylist, power users, trusted entities, membership root, security info, receiver-KYC toggle).
  2. **Power users** — nodes in the *power-users linked list*, each carrying independent capability flags: `is_admin`, `can_mint`, `can_burn`, `can_pause`, `can_force_transfer`. Each privileged action requires that power user's signature.
  3. **Trusted entities (KYC providers)** — Ed25519 vkeys in `GlobalStateDatum.trusted_entity_vkeys` ("TEL"). They sign **off-chain** KYC attestations; they are not on-chain transactors.
- `must_be_signed_by_credential` accepts **either** a transaction signature (key hash in `extra_signatories`) **or** a script withdrawal keyed by `Script(hash)` — so any authority MAY be a script (multisig / DAO).

## CMTAT Function Equivalency Table

### Metadata
- Implementation language: **Aiken** (on-chain, Plutus V3 / eUTXO). Off-chain tooling: **Java / Spring Boot** (transaction builders) and **TypeScript / Next.js** (reference frontend, Mesh SDK).
- Implementation version: `security-token` substandard **v0.0.0**; Aiken compiler **v1.1.21**; `plutus = "v3"`; platform repo commit **`bab6fc8`** (`cip113-programmable-tokens-platform`, 2026-06-05). Deps: `aiken-lang/stdlib v3.1.0`, `anastasia-labs/aiken-design-patterns v1.6.0`, `aiken-lang/merkle-patricia-forestry v2.1.0`.

### Token Attributes
#### Mandatory
| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 1 | Name attribute | ERC20 `name` | Public (`view`) |  | partial | Compile-time parameter; off-chain metadata public | No `name()` accessor. The Cardano **native asset name** (`security_asset_name: ByteArray`, a validator parameter) is the on-chain name. Human-readable name/issuer live in `GlobalStateDatum.security_info` (`SecurityInfo.issuer_name`) and can be published off-chain via CIP-26/CIP-68 token registries. |
| 2 | Ticker symbol attribute | ERC20 `symbol` | Public (`view`) |  | partial | Compile-time parameter; off-chain metadata public | No separate symbol field. The native asset name doubles as the ticker; a distinct ticker MAY be published off-chain (CIP-26). |
| 3 | Reference to legally required documentation | `terms` | Public (`view`) |  | partial | `ModifySecurityInfo` → admin (`admin_credential_hash`) | No CMTAT `terms` (doc URI + hash) structure. `GlobalStateDatum.security_info` (`SecurityInfo`) carries BaFin fields: `isin`, `term_of_issue`, `crypto_security_register`, `issuer_registration`, `record_keeping`, custodians, etc. It is **metadata only — not enforced by any validator**. There is no on-chain document hash field. |
| 4 | No fractions | ERC20 `decimals` | Public (`view`) | - Decimals must be set to zero unless governing law permits fractions.<br />- CMTAT Solidity allows configurable decimals at deployment | y | Ledger-enforced (integer-only) | Cardano native assets are **integer-only at the ledger level**; there is no on-chain `decimals`. "No fractions" therefore holds by construction. Any display decimals would be purely off-chain (CIP-26) and cannot create on-chain fractional balances. |

For CMTAT reference implementations, decimals SHOULD be configurable rather than defaulting to zero, to support use cases beyond tokenized shares in Switzerland.

##### Note

> On Cardano the ledger does not model decimals; a security token is intrinsically indivisible on-chain, which aligns with the Swiss "no fractions" requirement but means fractionalization (if ever needed) would require an off-chain convention plus a larger `security_asset_name` supply. `SecurityInfo.nominal_amount` and `volume_of_issuance` document the intended economic denomination.

#### Optional
| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 5 | Token ID attribute | `tokenId` | Public (`view`) | Optional parameter. | partial | Public (ledger-derived); ISIN via admin `ModifySecurityInfo` | The unique on-chain identity is the **issuance policy id** (28-byte hash, derived from the CIP-113 registry node). A human-facing unique id (`isin`) is stored in `SecurityInfo`. There is no standalone `tokenId` accessor. |

For CMTAT reference implementations, `tokenId` SHOULD be included.

##### Note

> The issuance policy id is globally unique and immutable, functioning as the de-facto `tokenId`; `SecurityInfo.isin` provides the regulatory identifier.

#### Token module

##### Mandatory

| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 6 | Know total supply | ERC20 `totalSupply` | Public (`view`) |  | partial | Public (on-chain state + ledger query) | No `totalSupply()`. On-chain, `GlobalStateDatum.mintable_amount` is a **remaining-mintable cap** (decremented on mint, incremented on burn); `SecurityInfo.volume_of_issuance` records the intended cap. Circulating supply = sum of on-chain UTxOs, obtainable via a chain indexer (Blockfrost/Yaci). |
| 7 | Know balance | ERC20 `balanceOf` | Public (`view`) |  | partial | Public (ledger query) | No `balanceOf()`. In eUTXO a holder's balance is the sum of the security-token quantity across UTxOs controlled by their credential; queried off-chain via the ledger/indexer. |
| 8 | Transfer tokens | ERC20 `transfer` | Token holder (`msg.sender`) |  | y | Token holder (owns/spends the UTxO), **gated** by KYC proof + denylist-absence + not-paused | `transfer_logic_validator.withdraw` (`transfer_logic_script.ak`) runs on every prog-token spend. Redeemer `TransferLogicScriptWithdrawRedeemer`. Per sender: valid trusted-entity **KYC proof** + **denylist absence**. Per destination: denylist absence (+ KYC if `requires_receiver_kyc`). Plus global pause gate. |
| 9 | Create tokens | `mint` / `batchMint` | Role-restricted (issuer/minter authorized) |  | y | **Power user with `can_mint`** (must sign) | `minting_logic_validator.withdraw` (`minting_logic_script.ak`), redeemer `MintingLogicScriptWithdrawRedeemer`. Positive `minted_amount` requires a referenced power-user node signed the tx and `power_user_data.can_mint`. GS is spent concurrently via `MintSecurity`, enforcing the supply cap `mintable_amount − minted ≥ 0`. Batch = a single tx can mint to many outputs (eUTXO-native). |
| 10 | Cancel tokens | `burn` / `batchBurn` / `burnFrom` | Role-restricted (issuer/burner authorized) | Implementations SHOULD use a dedicated issuer/authorized burn path for forced cancellation scenarios. | y | **Power user with `can_burn`** (must sign) | Same `minting_logic_validator.withdraw`; negative `minted_amount` requires `power_user_data.can_burn`. There is **no holder self-burn**. Because burn is a power-user forced operation, it doubles as the "forced cancellation" path (equivalent to `burnFrom`), and it can burn tokens held by a denylisted address (see [Implementation Details](#implementation-details)). |
#### Optional

| ID   | Requirement | CMTAT Solidity corresponding feature            | Access Control (CMTAT Solidity) | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 11   | Approve     | ERC20 `approve(address spender, uint256 value)` | Token holder                    | Grants a delegate permission to transfer a specific amount of tokens from the token account. This is optional, but implementations SHOULD include it since secondary market capability may depend on delegated approval to automate trading and settlement for regulated entities. Issuers SHOULD consult relevant trading and settlement venues if listing is contemplated. | n | — | **Absent.** eUTXO has no allowance model. There is no `approve`/`transferFrom` analogue; delegation is expressed instead through Cardano native/multisig scripts at the holder's credential, or via the admin **forced transfer** (ID 17). |

##### Note

> Secondary-market settlement would be built off-chain (the platform ships a Java transaction-builder service); atomic swap / DvP is expressed as a single multi-party transaction rather than a stored on-chain allowance.

### Pause module (mandatory)

| ID   | Requirement         | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity)           | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 12   | Pause tokens        | `pause`                              | Role-restricted (pauser/admin authorized) | Pause must prevent all transfers until `unpause` is called.  | y | **Power user with `can_pause`** (must sign) | `global_state_spend_validator` action `PauseTransfers { transfers_paused: True, .. }` sets `GlobalStateDatum.transfers_paused = True`. Enforced via `is_paused` in `transfer_logic_validator` **and** `third_party_transfer_logic_validator`. Caveat: pause blocks transfers **and forced transfers**, but **not** mint/burn (those short-circuit the transfer logic). |
| 13   | Unpause tokens      | `unpause`                            | Role-restricted (pauser/admin authorized) |                                                              | y | **Power user with `can_pause`** (must sign) | Same `PauseTransfers` action with `transfers_paused: False`. |
| 14   | Deactivate contract | `deactivateContract`                 | Role-restricted (admin authorized)        | Must permanently disable the token (except in upgradeability patterns where deactivation behavior is explicitly defined). | n (partial) | Admin (list `Deinit`); power user `can_pause` (indefinite pause) | **No dedicated permanent kill-switch.** Closest equivalents: (a) an **indefinite pause** (`can_pause` power user), and (b) admin-signed `Deinit` teardown of the power-user / denylist linked-list roots. There is no single action that irreversibly deactivates the token; validators are immutable scripts, so "deactivation" is a governance/operational choice, not a contract state. |

#### Enforcement

#### Mandatory

| ID   | Requirement | CMTAT Solidity corresponding feature                         | Access Control (CMTAT Solidity)               | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 15   | Freeze      | `freeze` or `setAddressFrozen(true)` *(inferred from extracted PDF text)* | Role-restricted (compliance/admin authorized) | Must block transfers to and from a given address. Single-function implementations are acceptable if they set a frozen status. | y | **Admin** (`admin_credential_hash`, must sign) | Implemented as a **denylist** (blocklist), not per-address flags. `denylist.ak` `mint` redeemer `AddToDenylist { user_pkh, .. }` inserts a linked-list node keyed by the user PKH. **Presence blocks both sending and receiving** (via `verify_denylist_absence` covering-node absence proofs in both transfer validators), matching CMTAT freeze. Uses two functions (add/remove) rather than a single `setAddressFrozen`. |
| 16   | Unfreeze    | `unfreeze` or `setAddressFrozen(false)` *(inferred from extracted PDF text)* | Role-restricted (compliance/admin authorized) | Single-function implementations are acceptable if they clear a frozen status. | y | **Admin** (`admin_credential_hash`, must sign) | `denylist.ak` `mint` redeemer `RemoveFromDenylist { user_pkh, .. }` removes the node. |

#### Optional

| ID   | Requirement        | CMTAT Solidity corresponding feature                         | Access Control (CMTAT Solidity)                  | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 17   | Enforce a transfer | `forcedTransfer(address from, address to, uint256 value)`    | Role-restricted (operator/compliance authorized) | Enforcement transfer is performed via `forcedTransfer`.      | y | **Power user with `can_force_transfer`** (must sign) | `third_party_transfer_logic_validator.withdraw` (`third_party_transfer_logic_script.ak`), redeemer `ThirdPartyTransferLogicScriptWithdrawRedeemer`. **No source checks** (source may be denylisted / KYC-expired — deliberate admin override), but each **destination** must be KYC'd and denylist-absent; the pause gate still applies. This is the seize / recovery capability. Does **not** burn (destination required). |
| 18   | Partial freeze     | `freezePartialTokens(address account, uint256 value)` / `unfreezePartialTokens(address account, uint256 value)` | Role-restricted (operator/compliance authorized) | Intended only to block a sold amount to avoid double-spend during settlement. | n | — | **Absent.** Denylist and pause are all-or-nothing (full address block / full global pause). There is no amount-scoped freeze. |

#### Transfer restriction (optional)

| ID   | Requirement                   | CMTAT Solidity corresponding feature                         | Access Control (CMTAT Solidity)                         | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 19   | Conditional transfer request  | `RuleConditionalTransferLight.detectTransferRestriction(...)` and `approvedCount(...)` | Public (`view`)                                         | Request is represented by a transfer restricted until approval count is non-zero. | n (implicit) | — | **No explicit on-chain request/hold workflow.** Every transfer is *implicitly* conditional: it requires a fresh, TTL-bound **trusted-entity KYC attestation** for the sender (and receiver if `requires_receiver_kyc`) plus denylist absence. Pre-approval happens off-chain (a trusted entity signs the attestation) rather than via an on-chain pending-approval queue. |
| 20   | Conditional transfer approval | `RuleConditionalTransferLight.approveTransfer(...)` | Role-restricted (compliance/approver authorized)        | Approval is consumed on transfer via `transferred(...)`; cancellation via `cancelTransferApproval(...)`. | n (implicit) | Trusted entity (off-chain Ed25519 attestation) | The "approval" is the off-chain KYC attestation (`AttestationProof`) signed by a trusted entity in the TEL; it is validated per-transaction (`verify_attestation_proof`) and bound to `user_pkh`, `security_policy_id`, `network_id`, and a `valid_until_ms` TTL. There is no on-chain single-use approval consumption / cancellation. |
| 21   | Assign to whitelist           | CMTAT Allowlist / Rules whitelist setters and `isAddressListed` | Role-restricted for setters; public (`view`) for checks | CMTAT Allowlist and Rules whitelist are alternative whitelist implementations. | y | Admin manages TEL + membership root; **trusted entities** issue attestations | Two allowlist mechanisms in `lib/kyc/verify.ak`: (a) **Attestation** — Ed25519 proof by a TEL issuer (`AddTrustedEntity`/`RemoveTrustedEntity`/`UpdateTrustedEntity`, admin-gated); (b) **Membership** — Merkle-Patricia-Forestry inclusion proof against `GlobalStateDatum.member_root_hash` (admin-updated via `UpdateMemberRootHash`). Sender always gated; receiver gated iff `requires_receiver_kyc` (admin `SetRequiresReceiverKyc`). |

##### Note

> The KYC/allowlist model is **capability-based and off-chain-attested** rather than an on-chain address set: a valid, unexpired issuer signature (or MPF membership proof) is presented at transfer time. This keeps per-address on-chain state small and supports tiered KYC (`tier_user` / `tier_institutional` / `tier_vlei`). The denylist (ID 15/16) is the complementary on-chain blocklist.

#### Access Control

| ID   | Requirement      | CMTAT Solidity corresponding feature                         | Access Control (CMTAT Solidity)                 | Notes                                                        | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 22   | Grant role       | `grantRole(bytes32 role, address account)` | Role admin (`DEFAULT_ADMIN_ROLE` or role admin) | Used for roles such as `ALLOWLIST_ROLE`, `DEBT_ROLE`, `OPERATOR_ROLE`, `COMPLIANCE_MANAGER_ROLE`. | y | **Admin** (`admin_credential_hash`, must sign) | Roles are power-user nodes. `power_users.ak` `mint` redeemer `AddPowerUser { new_power_user_key, .. }` inserts a `PowerUser { is_admin, can_mint, can_burn, can_pause, can_force_transfer }`. KYC-issuer role = `AddTrustedEntity` on GS. Both admin-gated. |
| 23   | Revoke role      | `revokeRole(bytes32 role, address account)`                  | Role admin (`DEFAULT_ADMIN_ROLE` or role admin) | AccessControl role removal.                                  | y | **Admin** (`admin_credential_hash`, must sign) | `RemovePowerUser { power_user_key_to_remove, .. }` (power users) and `RemoveTrustedEntity { vkey }` (KYC issuers). Flags can also be re-scoped in place via spend redeemer `ModifyPowerUser { wanted_user_state, .. }`. Admin itself is rotated via `RotateAdmin` (dual-signature old+new). |
| 24   | Role attribution | `hasRole(...)` / `getRoleAdmin(...)` | Public (`view`)                                 | In CMTAT `AccessControlModule`, `DEFAULT_ADMIN_ROLE` is treated as having all roles. | y | Public (on-chain datum inspection) | Role membership is read on-chain by referencing a power-user node UTxO (`get_element_info` / `run_element_with`) or the GS `admin_credential_hash`; anyone can inspect these UTxOs. No `hasRole` view function; there is no single super-role that implicitly holds all others (admin gates *list mutation*; capability flags gate *operations*). |

##### Note

> Authorization is intentionally decomposed into three authorities (see [Architecture Primer](#architecture-primer-read-first)) rather than a single OpenZeppelin `AccessControl` registry: **admin** (list governance), **power-user flags** (mint/burn/pause/force-transfer operations), and **trusted entities** (KYC issuance). Note `PowerUser.is_admin` exists as a flag but the effective admin authority is the GS `admin_credential_hash` — the flag is not consulted by the validators.

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
> No on-chain snapshot mechanism exists. On Cardano, historical balances at a given slot can be reconstructed off-chain from the ledger/indexer, but this is not part of the substandard.

#### Dividend (optional)

| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 31 | Distribution create parameters |  |  |  | n | — | **No distribution logic.** Only a descriptive `SecurityInfo.dividend_payment_frequency` (and `payment_bond`) metadata field exists; no create/deposit/claim validators. |
| 32 | Distribution set eligibility |  |  |  | n | — | Not implemented. |
| 33 | Distribution set deposit |  |  |  | n | — | Not implemented. |
| 34 | Distribution claim deposit |  |  |  | n | — | Not implemented. |
| 35 | Distribution schedule |  |  |  | n | — | Not implemented. |
| 36 | Distribution unschedule |  |  |  | n | — | Not implemented. |
##### Note
> No direct CMTAT Solidity equivalent is defined for these; they are implementation-specific (cf. the CMTA IncomeVault prototype). This substandard implements none of them on-chain — dividends would be a separate future module.

#### Credit Events (optional)
| ID | Requirement | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 37 | Flag as default | `setCreditEvents(...)` → `flagDefault` | Role-restricted | Managed in `ICMTATCreditEvents.CreditEvents`. | n | — | **No credit-events structure.** No `flagDefault`/`flagRedeemed`/`rating` fields in `GlobalStateDatum`. Could only be represented informally inside the free-form `security_info` `Data`. |
| 38 | Remove default flag | `setCreditEvents(...)` `flagDefault = false` | Role-restricted |  | n | — | Not implemented. |
| 39 | Flag as redeemed | `setCreditEvents(...)` → `flagRedeemed` | Role-restricted |  | n | — | Not implemented. |
| 40 | Set rating | `setCreditEvents(...)` → `rating` | Role-restricted |  | n | — | Not implemented. |
##### Note
> Credit-event state is not modeled. Admin can rewrite the opaque `security_info` blob (`ModifySecurityInfo`), which could carry ad-hoc status, but there is no typed/enforced credit-event field.

### Debt (optional)
| ID | Attribute | CMTAT Solidity corresponding feature | Access Control (CMTAT Solidity) | Notes | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|---|
| 41 | Guarantor identifier | `debt().debtIdentifier.guarantor` | Read public; write role-restricted | Debt module. | n | — | No debt module. `SecurityInfo` is an **equity/BaFin** metadata structure, not a debt instrument. No guarantor field. |
| 42 | Debtholder representative identifier | `debt().debtIdentifier.debtHolder` | Read public; write role-restricted |  | n | — | Not present. |
| 43 | Unique identifier / hash | `tokenId()` and `terms().doc.documentHash` | Public (`view`) |  | partial | Public / admin (`ModifySecurityInfo`) | Unique id = issuance policy id + `SecurityInfo.isin` + `crypto_security_register`. No document hash field. |
| 44 | Issuance date | `debt().debtInstrument.issuanceDate` | Read public; write role-restricted |  | partial | Admin (`ModifySecurityInfo`) | `SecurityInfo.term_of_issue` (free-form `ByteArray`) can encode issuance terms/date; not a typed date, not validator-enforced. |
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
> This substandard targets **equity-style regulated securities (BaFin)**, so the CMTAT *debt* module is largely out of scope. The `SecurityInfo` type instead captures equity attributes — `isin`, `security_type`, `security_form`, `security_class`, `record_keeping`, `entry_type`, `without_voting_rights`, `transfer_restrictions`, `third_party_rights`, `company_consent`, `participate_settlement`, `nominal_amount`, `volume_of_issuance`, `issuer_name`, `issuer_registration`, `crypto_security_register`, `share_custodian`, `token_custodian` — all as **non-enforcing** metadata (GS comment: security_info is "not directly used in the contracts").

## Guideline for New Blockchain Implementations

### Freeze

To be compatible with [ERC-3643](https://eips.ethereum.org/EIPS/eip-3643), freeze is implemented with a single function: `setAddressFrozen(targetAddress, frozenStatus)`.

For non-EVM blockchains, implementations MAY separate this into two distinct functions:

```solidity
freeze(address targetAddress)
unfreeze(address targetAddress)
```

##### Note

> This implementation uses the **two-function** form via the denylist mint validator: `AddToDenylist { user_pkh, .. }` and `RemoveFromDenylist { user_pkh, .. }` (both admin-signed). "Frozen status" is represented by **presence of a node** (keyed by the payment key hash) in an ordered linked list; absence is proven at transfer time by a *covering-node* reference input (`verify_denylist_absence`). Semantics match CMTAT's freeze, which is not send-side only — `ValidationModule` blocks a transfer when `spender`, `from` or `to` is frozen, and refuses to mint to or burn from a frozen address. A denylisted address here can likewise **neither send nor receive** the security token, and this also blocks forced transfers *to* a denylisted destination (but not forced transfers *from* one — see ID 17).

### CMTAT Extended

In the table below, the CMTAT framework extended features are mapped to Solidity features.

| CMTAT Functionalities | CMTAT Solidity corresponding features | CMTAT Allowlist | CMTAT Light | CMTAT Debt | CMTAT Standard | Present in implementation being approved (`y/n`) | Implementation details |
|---|---|---|---|---|---|---|---|
| On-chain snapshot | `snapshotModule` and `snapshotEngine` | ✔ | ✘ | ✔ | ✔ | n | Not implemented (see IDs 25–30). |
| Forced transfer | `forcedTransfer` | ✔ | ✘ | ✔ | ✔ | y | `third_party_transfer_logic_validator` — power user `can_force_transfer`; destination KYC + denylist-absent; source unchecked; respects pause. Does not burn. |
| Forced burn | `forcedBurn` | ✘ | ✔ | ✘ | ✘ | y | Burn is inherently a power-user (`can_burn`) forced operation — no holder consent needed — and can burn tokens held by a denylisted address (mint/burn short-circuits the KYC/denylist gate). Implemented via `minting_logic_validator` (negative mint), distinct from `forcedTransfer`. |
| Freeze partial token | `freezePartialTokens` / `unfreezePartialTokens` | ✔ | ✘ | ✔ | ✔ | n | Not implemented (see ID 18). |
| Integrated whitelisting/allowlisting | CMTAT Allowlist | ✔ | ✘ | ✘ | ✘ | y | Built-in KYC allowlist: trusted-entity attestations + MPF membership tree, plus the integrated denylist. See IDs 15/16, 21. |
| External whitelisting/allowlisting | CMTAT with rule whitelist | ✘ | ✘ | ✔ | ✔ | y | KYC is attested by **external** trusted entities (off-chain Ed25519 issuers registered in the TEL); the substandard itself is also a pluggable "rule set" over the CIP-113 core. |
| RuleEngine / transfer hook | CMTAT with RuleEngine | ✘ | ✘ | ✔ | ✔ | y | The CIP-113 model **is** a transfer hook: `transfer_logic_validator` (withdraw-0) runs on every prog-token spend and is the substandard's RuleEngine equivalent. |
| Upgradeability | CMTAT Upgradeable version | ✔ | ✔ | ✔ | ✔ | n (partial) | Plutus validators are **immutable** (no proxy pattern). Mutable *state* lives in the GS datum; admin/roles/KYC-issuers/security-info are all updatable, and admin is rotatable (`RotateAdmin`). "Upgrading logic" requires deploying new validators and re-pointing the CIP-113 registry — not an in-place proxy upgrade. |
| Fee payer / gasless | CMTAT with ERC-2771 module | ✔ | ✘ | ✘ | ✔ | n | No ERC-2771 meta-transaction module. Fee sponsorship would be handled off-chain at the transaction-building layer (collateral/fee inputs supplied by a sponsor), not by an on-chain trusted-forwarder. |

##### Note

> **Upgradeability** on Cardano differs fundamentally from EVM proxies: there is no delegatecall. The chosen model is *immutable logic + mutable NFT-authenticated datum state + registry re-pointing*. **Gasless/sponsorship** is achievable off-chain (a sponsor adds fee/collateral inputs to the user's transaction) but is not codified on-chain. The KYC-proof design is **network-bound** (`network_id`: preview/preprod/mainnet/yaci) to prevent cross-network attestation replay.

### Forced Burn and Forced Transfer

In the standard burn function, tokens from a frozen wallet MUST NOT be burnable. CMTAT offers `forcedTransfer` to force a transfer or a burn.

If `forcedTransfer` is not available, implementations MAY implement only `forcedBurn` (as in CMTAT Light). Implementations MAY also implement both. In that case, only `forcedBurn` SHOULD burn tokens, and `forcedTransfer` SHOULD NOT burn tokens.

With the CMTAT Solidity version, when `forcedTransfer` is available, `forcedBurn` is not implemented to reduce contract code size. This limitation MAY not apply to other blockchains.

##### Note

> This implementation provides **both**, and follows the recommended split: **`forcedTransfer`** = `third_party_transfer_logic_validator` (destination required, **does not burn**, `can_force_transfer` power user); **`forcedBurn`** = the `can_burn` negative-mint path in `minting_logic_validator` (**does burn**, no holder consent, works on denylisted holders). Code-size is not a binding constraint on Cardano, so keeping the two paths separate is intentional and matches the CMTAT guideline ("only `forcedBurn` SHOULD burn").

### Implementation Details

| Functionalities | CMTAT Solidity | Access Control (CMTAT Solidity) | Note | Present in implementation being approved (`y/n`) | Access Control (implementation being approved) | Implementation details |
|---|---|---|---|---|---|---|
| Mint while pause | ✔ | Role-restricted (minter/issuer authorized) | Dedicated cross-chain mint cannot be performed while paused. | y | Power user `can_mint` | Mint is **allowed while paused**: the transfer logic short-circuits on `is_mint_or_burn`, and the GS `MintSecurity` branch does not read `transfers_paused`. |
| Burn while pause | ✔ | Role-restricted (burner/issuer authorized) | Dedicated cross-chain burn cannot be performed while paused. | y | Power user `can_burn` | Burn is **allowed while paused** for the same reason (short-circuit). Note: forced *transfer* (seize), by contrast, **is** blocked while paused. |
| Self-Burn for everyone | ✘ | Not permitted | Token holders cannot burn their own tokens; only authorized addresses can burn. | n (not permitted) | — | Matches CMTAT: holders cannot self-burn. Every burn requires a `can_burn` power-user signature. |
| Self-Burn for authorized addresses | ✔ | Role-restricted (authorized burner) |  | y | Power user `can_burn` | Authorized burn via `minting_logic_validator` negative mint. |
| Standard burn on a frozen address | ✘ | Not permitted in standard burn path | Requires `forcedTransfer` or `forcedBurn`. | y (diverges) | Power user `can_burn` | **Divergence:** because there is no holder "standard burn" at all, *all* burns are forced/authorized burns, and a `can_burn` power user **can** burn tokens held by a denylisted address (the KYC/denylist gate is short-circuited for mint/burn). This is effectively CMTAT's `forcedBurn`, not the prohibited standard-burn-on-frozen. |
| Burn tokens with `forcedTransfer` | ✔ | Role-restricted (operator/compliance authorized) | See notes above. | n | — | `forcedTransfer` here (`third_party_transfer_logic`) **cannot** burn — it requires a destination output. Burning is done via the separate `forcedBurn` (`can_burn`) path instead. |

### Self-Burn

Only the issuer and authorized addresses (not the token holder) can burn a token in CMTAT Solidity, which reflects legal requirements in several jurisdictions.

Once issued, a security can only be cancelled by its issuer, not its holder. Since the token represents the security, the same rule applies. An investor who wants to exit should transfer to the issuer, who can then cancel when legally permitted.

You MAY still add self-burn in your version if it fits your legal or business context.

##### Note

> This implementation **follows the CMTAT legal model**: token holders cannot self-burn. Cancellation is a power-user (`can_burn`) operation only. An exiting investor transfers to an authorized address (KYC/denylist gated), which then burns.

## Supplementary features

> Features present in this implementation beyond the CMTAT baseline:
> - **Tiered, TTL-bound, off-chain KYC attestations** (`tier_user` / `tier_institutional` / `tier_vlei`), verified on-chain via Ed25519 (`AttestationProof`) or MPF membership proof (`MembershipProof`), and **network-bound** to prevent cross-network replay.
> - **Toggleable receiver-KYC** (`SetRequiresReceiverKyc`) — the issuer can require KYC of recipients or gate only senders.
> - **Rotatable admin with dual-signature handover** (`RotateAdmin`) — mitigates silent takeover and lost-key soft-brick.
> - **On-chain supply cap** (`mintable_amount`) enforced atomically with each mint via the GlobalState UTxO.
> - **eUTXO-native batch mint/transfer** — a single transaction can create/move tokens across many holders.
> - **Griefing-hardened credential registration** — the withdraw-0 stake credentials whitelist `RegisterCredential` only (reject `UnregisterCredential`) to prevent a de-registration DoS.
> - **BaFin `SecurityInfo` regulatory metadata** block for German/EU regulated-securities disclosure.

## Reference

Implementation being approved:

| Component | Repository | Version / Commit |
|---|---|---|
| Platform (off-chain + substandards) | https://github.com/cardano-foundation/cip113-programmable-tokens-platform | commit `bab6fc8` (2026-06-05) |
| `security-token` substandard | `src/substandards/security-token/` | pkg `cip113/security-token` v0.0.0; Aiken `v1.1.21`; Plutus V3 |
| CIP-113 core on-chain validators | https://github.com/cardano-foundation/cip113-programmable-tokens-2 | (separate repo) |
| CIP-113 specification | https://github.com/cardano-foundation/CIPs/pull/444 | active development (supersedes CIP-143) |
| Upstream port origin | https://github.com/FluidTokens/fn-bafin-cardano-sc | BaFin Cardano SC. Named `easy1staking-com/fn-bafin-cardano-sc` in this substandard's `aiken.toml` description (unverified path). |

Assessment template submodules (unchanged from the template — the CMTAT Solidity reference this Cardano implementation is compared against):

| Submodule | Repository | Version | Commit |
|---|---|---|---|
| CMTAT | https://github.com/CMTA/CMTAT | `v3.2.0` | `49544f4de1993008acfc9e848d0bf03bd31d8579` |
| SnapshotEngine | https://github.com/CMTA/SnapshotEngine | `v0.3.0-1-g19e0b56` | `19e0b569bf5823aa8cec5760f080a932a9ac940e` |
| RuleEngine | https://github.com/CMTA/RuleEngine | `v3.0.0-rc2-2-g9c0aa70` | `9c0aa70aae08047e4062beab0f89f92bd60252c0` |
| Rules | https://github.com/CMTA/Rules | `v0.3.0` | `91c21c1191e84ff938892267ec443b0d1bb9efb0` |
