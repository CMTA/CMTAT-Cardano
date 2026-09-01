# CHANGELOG

Please follow [https://changelog.md/](https://changelog.md/) conventions.

## Semantic Version 2.0.0

This repository ships documents rather than code, so a version tracks what a reader relying on an assessment would conclude from it. Each assessment additionally carries its own version in its `## Document Version` section, alongside the CMTA template version it was filled from.

Given a version number MAJOR.MINOR.PATCH, increment the:

1. MAJOR version when the new version makes:
   - A change to any equivalency **answer** (`y`, `n`, `partial`, `n (partial)`, …) in a filled assessment
   - A change to an **assessed revision** — re-pinning a codebase to a different commit, since every answer is a statement about one revision
   - An upgrade to a new **CMTA template version**, which may renumber, merge or restructure the 54 items
2. MINOR version when the new version adds material in a backward compatible manner — a new assessment, article, diagram, section or note — with no existing answer changed and no revision re-pinned
3. PATCH version when the new version makes corrections that leave every answer and every pinned revision intact: wording, typos, broken links, reflows

See [https://semver.org](https://semver.org)

## Type of changes

- `Summary`: main new features/change with a description (keep it short) (not a changelog tag)
- `Added` for new features.
- `Changed` for changes in existing functionality.
- `Deprecated` for soon-to-be removed features.
- `Removed` for now removed features.
- `Fixed` for any bug fixes.
- `Security` in case of vulnerabilities.

Reference: [keepachangelog.com/en/1.1.0/](https://keepachangelog.com/en/1.1.0/)

Custom changelog tag: `Dependencies` (submodule pin moves), `Diagrams`

## Checklist

> Before a new release, perform the following tasks

- Documents: update the version in each changed assessment's `## Document Version` section, and move its mirrors with it — the commit in that assessment's `### Metadata` bullets and in its `## Reference` table, and the *Pinned at* column in [`README.md`](./README.md)
- Verify every cited revision against the working tree, since the assessments quote commits rather than branches

```bash
git submodule status
```

- Check Markdown line-break style across the repository: no file may mix hard-wrapped prose with one-line-per-block prose. The two styles in one file produce the ragged source that readers report as broken line breaks, and it is invisible in the rendered page.

- Diagrams
  - Re-render every `.puml` changed since the last release, then **open each PNG**: `plantuml` exits 0 while drawing parse errors and layout warnings into the image
  - Register any new diagram in [`tree.txt`](./tree.txt), tagged `mindmap` / `concept` / `workflow` / `state`

```bash
plantuml -tpng assets/article/blockchain/cardano/<name>.puml
```

- Keep [`CLAUDE.md`](./CLAUDE.md) and [`AGENTS.md`](./AGENTS.md) byte-identical

```bash
diff CLAUDE.md AGENTS.md
```

- Regenerate the specification PDF in [`doc/`](./doc) from the assessments, and name it for the release
- Update this changelog

## Unreleased

### Added

- Article on [absence proofs](./2026-08-31-cardano-absence-proof-sanctions-list.md): how a party proves it is not on the sanctions list, why a caller-supplied reference input is sound, and what a transaction builder must do to satisfy the check.

### Diagrams

- Four PlantUML sources for the absence-proof article, registered in [`tree.txt`](./tree.txt): a mindmap, the sorted linked list, the off-chain build, and the trust chain across the sanction and transfer paths.

## v0.1.0 - 2026-08-31

`Summary`: first release. Three filled CMTA equivalency assessments covering the Cardano CIP-113 security-token lineage, two long-form articles, and the specification PDF built from them.

### Added

- Equivalency assessment of [`cpt-rwa-ch-de-cmta-reference`](./README-cpt-rwa-ch-de-cmta-reference.md) at commit `ff5624e`, the current codebase, shipping a German (eWpG) and a Swiss (CMTA) profile over one shared set of validators.
- Equivalency assessment of the Cardano Foundation [`security-token` substandard](./README-cip113-security-token.md) at commit `bab6fc8`, and of its upstream origin [`fn-bafin-cardano-sc`](./README-fn-bafin-cardano-sc.md) at commit `67ab7d9`.
- Each assessment fills all 54 numbered items of CMTA template `v0.2.0`, plus the guideline sections, with an architecture primer explaining how eUTXO validators map onto a Solidity contract's methods.
- Article on [how the CIP-113 substandard reconstructs CMTAT on eUTXO](./2026-07-17-cmtat-cardano-cip113.md), and one on [what the successor codebase changed](./2026-08-26-cardano-cmta-swiss-german-profiles.md).
- Annex to the `cpt-rwa` assessment: a 20-term Cardano glossary, an explanation of what a validator is on this ledger, and two worked rejection flows — a sanctioned holder and a paused protocol — each with its own diagram.
- FAQ answering what a validator and a node are here, how a denylist absence proof works, and why a zero-ADA withdrawal appears in every transfer.
- Specification PDF at [`doc/CMTAT-Cardano-Specificationv0.1.0.pdf`](./doc/CMTAT-Cardano-Specificationv0.1.0.pdf).

### Fixed

- Corrected four rows of the lineage comparison table, which attributed the Cardano Foundation substandard's behaviour to `fn-bafin-cardano-sc`.
  - Affected the deactivation and upgradeability answers, and both burn rows, understating what the upstream codebase does.
  - Only the substandard short-circuits the transfer hook for mint and burn; `fn-bafin` does not, and its own assessment says so.
- Corrected the internal-review status: ten of the eleven defects were fixed, and the eleventh — pause does not stop issuance — was accepted as intended behaviour rather than fixed.
- Corrected the claim that a CMTAT freeze is send-side only, and so that this implementation's denylist is stricter than one.
  - CMTAT `v3.2.0` blocks a transfer when the spender, sender **or** recipient is frozen, and refuses to mint to or burn from a frozen address.
  - The denylist matches that rather than exceeding it; it goes further only in keying on the bare hash, which covers both credential forms of it.
- Described CMTAT as the blockchain-agnostic framework it is, with Solidity as its reference implementation, rather than as an EVM-only Solidity codebase.
- Separated each assessment's own version from the CMTA template version it was filled from; the two had shared one number.
- Recorded a deliberate deviation on the mint path: a `can_mint` operator minting to its own list-authenticated credential is not asked for a KYC attestation, where CMTAT Allowlist would require one.

### Changed

- Rewrote the `cpt-rwa` assessment's opening as a reader-facing introduction, moving provenance into Metadata and the lineage into its own section.
- Split dense implementation-details cells into notes, taking the longest table cell from 1016 to 701 characters against a median of 101.
- Reflowed every Markdown file to one line per block, so a one-word correction no longer reflows a whole paragraph in the diff.

### Dependencies

- Pinned four submodules: `CMTAT-equivalency-assessment` at `ad5904e` (template `v0.2.0` plus CMTAT `v3.2.0`), `cip113-programmable-tokens-platform` at `bab6fc8`, `cpt-rwa-ch-de-cmta-reference` at `ff5624e`, and `fn-bafin-cardano-sc` at `0b9641d`.

### Diagrams

- Ten PlantUML sources with rendered PNGs under `assets/article/blockchain/cardano/`, registered in [`tree.txt`](./tree.txt): four per article, plus the two Annex rejection flows.
