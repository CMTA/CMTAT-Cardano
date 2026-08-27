# Agent guide — CMTAT on Cardano

> **Note — keep in sync:** `AGENTS.md` and `CLAUDE.md` must always be **identical**.
> Any edit to one must be applied verbatim to the other.

> **Note — commit messages:** After each group of modifications or each feature added, always provide a **one-line GitHub commit message** (Conventional-Commits style, e.g. `feat: ...`, `fix: ...`, `docs: ...`).
>
> **Never put `!` in a commit message** — not as the breaking-change marker (`feat!: ...`), not anywhere else. In an interactive bash, `!` inside double quotes triggers history expansion, so `git commit -m "feat!: ..."` aborts with `bash: !: unrecognized history modifier`. Signal a breaking change with an uppercase `BREAKING CHANGE:` line in the commit body instead, and keep the subject line free of `!`.

> **Note — no tool names in the changelog:** never name an assistant tool, skill or slash command in `CHANGELOG.md`. The changelog records what changed in *this project*, for readers who have no idea what tooling produced it. A line ending "the `<some-skill>` skill gained the corresponding check" documents the author's toolbox rather than the release, and it rots independently of the repository — the tool can be renamed or deleted, leaving a dangling reference to something the reader could never have seen. Describe the change and its effect; if the tooling matters, record it in the audit or analysis report instead.
>
> This is about **tool identities, not the word "Claude"**: files committed to the repository — `CLAUDE.md`, `AGENTS.md`, `CLAUDE_AUDIT.md`, `CLAUDE_ANALYSIS*.md` — are cited freely, because a reader can open them.

> **Note — do not hard-wrap prose in `CHANGELOG.md`:** one line per bullet or paragraph, and let the editor soft-wrap. Markdown collapses a single newline into a space, so a hard-wrapped bullet renders identically — the cost is invisible in the published changelog and paid entirely in the repository. Changing one word reflows every following line, so a one-word correction arrives as a multi-line diff in which a reviewer cannot see what actually changed; and because the wrap column depends on whoever wrote the entry, the file drifts into a mix of styles that reads as damage. Keep the line structure only where it is semantic: fenced code blocks, tables and blockquotes.

> **Note — long changelog entries get sub-bullets:** past roughly three sentences, a bullet stops being scannable — the defect, its blast radius, the fix, the precedent and the caveat all run together, so a reader looking for any one of them has to parse all five. Lead with one sentence naming *what changed*, then one sub-bullet per distinct claim: impact, fix, behaviour-change warning, cost, migration note. A useful trigger is length — compare against the file's own median bullet and split anything several times longer, since that length almost always means several claims in one paragraph. Sub-bullets follow the same no-hard-wrap rule: one line each.

## What this project is

A **documentation and analysis** repository, not a code repository: it holds filled **CMTA equivalency assessments** and long-form **articles** about CMTAT-equivalent security tokens on Cardano. Nothing here compiles or deploys; the code being analysed lives in four pinned git submodules. Every deliverable is a Markdown file at the repository root, plus PlantUML diagrams under `assets/`.

## Key concepts

- **CMTAT** is CMTA's EVM/Solidity security-token framework. **CIP-113** is Cardano's programmable-token proposal. The assessments map the second onto the first, item by item, using CMTA's own fillable template.
- **Three Cardano codebases**, in lineage order: `fn-bafin-cardano-sc` (upstream) → the CF platform's `security-token` substandard (port) → `cpt-rwa-ch-de-cmta-reference` (standalone successor, Swiss + German profiles). Each has its own assessment.
- **Assessments are filled copies of the CMTA template `v0.2.0`.** Keep its section order and its 54 numbered IDs intact — never renumber, merge or drop a row. Fill only the last three columns (`y/n`, access control, implementation details), plus `##### Note` blocks.
- **Answer vocabulary in use:** `y`, `n`, `partial`, `n (partial)`, `n (implicit)`, `y (diverges)`, `y (metadata)`, `n (not permitted)`. Reuse an existing form rather than inventing one.
- **Articles are local only.** They are deliberately *not* published to the access-denied website, so they carry no YAML front matter and use relative `./assets/...` image paths.
- The **CIP-113 base layer** (registry + programmable-logic base) is a deployment prerequisite that lives in none of the submodules. Several behaviours documented here are properties of *composing* a codebase with that base layer, not of either side alone.

## File tree

```
.
├── README.md                                  # project intro + index of every file here
├── README-cpt-rwa-ch-de-cmta-reference.md     # assessment: cpt-rwa (commit ff5624e) — the current codebase
├── README-cip113-security-token.md            # assessment: CF security-token substandard (commit bab6fc8)
├── README-fn-bafin-cardano-sc.md              # assessment: fn-bafin upstream (commit 67ab7d9)
├── 2026-07-17-cmtat-cardano-cip113.md         # article: how the substandard rebuilds CMTAT on eUTXO
├── 2026-08-26-cardano-cmta-swiss-german-profiles.md  # article: what the successor changed
├── tree.txt                                   # registry mapping each article to its .puml sources
├── assets/article/blockchain/cardano/         # .puml sources + rendered .png, one pair per diagram
├── CMTAT-equivalency-assessment/              # submodule: blank template + CMTAT Solidity reference
├── cip113-programmable-tokens-platform/       # submodule: codebase 2
├── cpt-rwa-ch-de-cmta-reference/              # submodule: codebase 3
└── fn-bafin-cardano-sc/                       # submodule: codebase 1
```

Each article has four diagrams in `assets/article/blockchain/cardano/`: one mindmap (embedded in the Conclusion) and three supporting concept/workflow/state diagrams (embedded in-section). No `.puml` source is ever pasted into an article.

## Submodules (pinned)

| Path | Pinned commit | Notes |
|---|---|---|
| `CMTAT-equivalency-assessment/` | `ad5904e` (`v0.2.0-1-gad5904e`) | Template plus CMTAT `v3.2.0`, RuleEngine, Rules, SnapshotEngine |
| `cip113-programmable-tokens-platform/` | `bab6fc8` | `security-token` substandard, Aiken `v1.1.21`, pkg `cip113/security-token` |
| `cpt-rwa-ch-de-cmta-reference/` | `ff5624e` | Aiken `v1.1.23`, pkg `ft/rwa` |
| `fn-bafin-cardano-sc/` | `0b9641d` | Aiken `v1.1.22`, pkg `ft/bafin`; the **assessed** commit is `67ab7d9`, five commits earlier |

## Common commands

```sh
git submodule update --init --recursive                              # fetch the analysed sources
plantuml -tpng assets/article/blockchain/cardano/<name>.puml         # render one diagram
aiken check -D                                                       # inside a codebase submodule
python3 ~/.claude/skills/check-markdown-linebreaks/check_linebreaks.py CLAUDE.md AGENTS.md
```

Rendering needs PlantUML (developed against 1.2026.2). `aiken check` must run with the compiler version that submodule pins, from inside its directory.

## Conventions

- **Never invent a URL.** A repository name found in a description field is not a verified path. `easy1staking-com/fn-bafin-cardano-sc` came from the CF substandard's `aiken.toml` and was linked for months without resolving; it is now quoted as plain text and the code is cited under `FluidTokens/`. Do not re-link it.
- **Pin every cited revision.** Full 40-character SHA plus the date of the reading, per repository. Cite what a local clone actually contains.
- **Do not overstate audit status.** All three codebases are unaudited; each assessment must keep its warning block, and `cpt-rwa`'s pentest reports predate its assessed commit.
- **Article style:** `[TOC]` goes after the introduction; one `## Annex` with `###` subsections; FAQ and References are their own `##` sections after it; no body-level `#`; at most one mid-sentence em dash in the whole body; alt text on every image.
- **Diagram style:** mindmaps need a `<style>` block or they render grey. Use flat skinparams (`skinparam cardBackgroundColor #EEF5FF`) — the nested one-line form `skinparam card { ... }` is a parse error. Activity-step colour goes *after* the `;` as `<<#EEF5FF>>`. Always open the rendered PNG: `plantuml` exits 0 while drawing warnings and parse errors into the image.
- **Register new diagrams in `tree.txt`**, one entry per article, tagged `mindmap` / `concept` / `workflow` / `state`.
- **Blank line before every Markdown table**, otherwise kramdown renders literal pipes. Five pre-existing violations are inherited from the CMTA template and are left alone deliberately.
- There is **no `CHANGELOG.md`** in this repository today. The changelog notes above apply if one is added.
