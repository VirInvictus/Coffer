# Coffer — preliminary research

**Status:** Phase 0, research only. No code, no spec yet. This file is the
landing pad for the design investigation; it graduates into `spec.md` once the
central decisions below are made.

**One-liner.** A native GNOME (GTK4 / libadwaita) envelope-budgeting application
over a plain-text hledger journal. Actual Budget's experience, hledger's data
discipline, two-way: the journal is the source of truth and Coffer both reads
from it and writes back to it. Atrium's sibling, but for money.

The name: a coffer is both a strongbox for valuables and the recessed panel in a
coffered ceiling. Finance and architecture in one word, the same dual reading as
"Atrium."

---

## 1. Why this exists

- **Real personal motivation.** Brandon already runs hledger for personal
  finance. The journal exists; the tooling around it is a CLI and the occasional
  `hledger-web` session. A native, beautiful budgeting surface is a daily-driver
  upgrade, not a hypothetical.
- **A genuine gap.** Actual Budget is the best open-source envelope budgeter, but
  it is Electron, sync-first, and owns its own SQLite store (your data lives in
  *its* database, not in files you control). Fava is gorgeous but beancount-only
  and read-only. `hledger-web` is the closest existing two-way tool, but it is a
  utilitarian Yesod web app, not a budgeting experience. Nobody ships a native
  GNOME envelope budgeter that treats a plain-text journal as canonical.
- **New portfolio domain.** Accounting / personal finance broadens the portfolio
  past the consume-and-curate axis (Calibre-shaped library managers) and the
  productivity axis (Atrium). It demonstrates a different competence: modeling a
  formal double-entry system and rendering it calmly.
- **The Lattice philosophy, applied to money.** Lattice treats the filesystem as
  truth and never silently rewrites it. Coffer treats the hledger journal the
  same way. This is the honest counterpart to Atrium, not a copy of it (see §4).

---

## 2. The landscape (prior art)

| Tool | Stack | Model | Storage | Two-way? | What to take |
|------|-------|-------|---------|----------|--------------|
| **Actual Budget** | Electron / React, `loot-core` (TS) | Strict envelope / zero-based, reactive spreadsheet | Own SQLite per budget (+ optional CRDT sync) | N/A (owns its data) | The *experience*: assign-every-dollar, carryover, the reactive recompute, the calm month grid |
| **hledger-web** | Haskell / Yesod | Whatever the journal encodes | The journal file (canonical) | **Yes** | The reference two-way model: JSON API + add-transaction + journal edit with numbered backups |
| **Fava** | Python + JS, beancount | Reporting / analysis (not envelope) | beancount file (read-mostly) | Read-only | The *reporting surface*: charts, balance sheets, income statements, account drilldown |
| **Paisa** | Go + web | Goal + retirement planning over ledger | ledger/beancount file | Limited | Budgeting/planning UI ideas (not cloned; revisit if needed) |

The takeaway: **the experience comes from Actual, the data model comes from
hledger, and `hledger-web` already proves the two-way write is possible.** Coffer
is the synthesis nobody has shipped: Actual's UX, hledger-canonical, native GNOME.

---

## 3. Key technical findings

### 3.1 hledger already exposes the surfaces we need to read

- **Machine-readable output.** `hledger ... --output-format json` (and CSV) is
  available across the report commands (`balance`, `register`, `print`, etc.).
  Numbers come through as both a float and a native Decimal object. Caveat worth
  remembering: `register --output-format json` is documented as awkward to
  consume (upstream issue #2552); Coffer should prefer `print` and `balance`
  JSON, and treat `register` carefully.
- **`hledger-web` is the two-way reference.** Its OpenAPI spec
  (`research/hledger/hledger-web/config/openapi.yaml`) documents a clean read API:
  `/accountnames`, `/transactions`, `/accounts`, `/accounttransactions/{name}`,
  `/commodities`, `/prices`. Writing is a separate capability: a new transaction
  is added by PUTting JSON to `/add`, and with the `manage` capability enabled it
  can edit/upload/download the journal (and any included files). Critically, it
  **writes a numbered backup of the journal on every edit.** That backup
  discipline is non-negotiable for Coffer and should be copied wholesale.

### 3.2 The budget-model fork (THE central decision)

Actual is strict envelope budgeting: every dollar is assigned to a category
before it is spent, and over/under-spend carries to next month. hledger offers
two incompatible ways to express "a budget," and Coffer must pick (or bridge):

- **(A) Envelope-as-subaccounts** (the YNAB-faithful path; see
  `research/hledger-envelope-budget/README.md`). Money lives in
  `assets:cash:<bank>:budget:checking:<category>` envelope accounts. "Assigning"
  money is a real transfer posting from an `:unallocated` envelope into a
  category envelope. Spending draws from the category envelope. Available-to-spend
  is just an account balance. **Pros:** models available funds *precisely*,
  matches Actual's semantics 1:1, every assignment is a real posting in the
  journal (fully auditable, survives without Coffer). **Cons:** the journal grows
  a lot of internal transfer postings; needs auto-posting discipline; the journal
  becomes noisier.

- **(B) Goal budgeting via periodic rules** (`hledger --budget`). Budgets are
  `~ monthly` periodic transaction rules; `--budget` reports actuals against
  goals. **Pros:** clean journal, budgets are declarations not money moves.
  **Cons:** this is *not* envelope budgeting. There is no real carryover of
  available money; Coffer would have to compute YNAB-style carryover itself and
  it would not be reflected as real balances. Diverges from the Actual experience.

- **(C) Hybrid (current lean).** Make the **journal canonical with envelopes as
  real subaccount transfers (A)**, and layer **Actual's reactive-spreadsheet UX
  (the carryover math, "assign every dollar," month-to-month available) on top as
  a computed view**. "Assign $200 to Groceries" becomes a one-click op that
  appends a real transfer transaction. Carryover and available-to-budget are
  derived live from `hledger bal` output, not stored. This keeps hledger as the
  single source of truth (nothing Coffer-only is required to interpret the data)
  while delivering the Actual feel. **This is the recommended direction and the
  first thing to validate against Brandon's real journal.**

### 3.3 Writing back to plain text is the hard problem

This is Coffer's equivalent of Atrium's two-way Org vault, and it is *the* risk.
A GUI that appends and edits transactions in a hand-maintained journal must not
destroy the user's formatting, comments, account-declaration ordering, `include`
structure, or alignment. Constraints, drawn from how `hledger-web` behaves:

- **Append, don't rewrite, by default.** New transactions append to a designated
  destination file. Full-file rewrites are avoided; when unavoidable, they go
  through `hledger print` only with explicit consent (it normalizes formatting).
- **Numbered backup on every write**, exactly as `hledger-web` does.
- **Respect `include`.** A journal is usually a tree of files (per-year, per-
  account). Coffer must know which file a write belongs in.
- **Round-trip honesty.** Anything Coffer does not model (comments, tags, custom
  directives, virtual postings) is preserved verbatim, never dropped.

---

## 4. How the Atrium idiom maps (and where it inverts)

Coffer is Atrium's sibling, but the polarity flips. Worth being explicit so the
architecture is not cargo-culted:

| Atrium | Coffer |
|--------|--------|
| **SQLite is canonical**, Org vault is a projection | **The hledger journal is canonical**, SQLite (if any) is a disposable read cache/index rebuilt from `hledger` output |
| Single-writer SQLite worker is the core | The core is a **journal writer** (safe append/edit of plain text) + a **report reader** (shell out to `hledger`, parse JSON) |
| Org vault is the optional peer projection | The journal is not optional; it is the whole point |
| Calibre search grammar over tasks | Likely **not** needed v1; hledger's own query language already filters. Revisit. |
| GTK4 + libadwaita + Rust 2024 | Same stack. This transfers cleanly. |
| `inotify` watch on the vault → refresh | `GFileMonitor` / `inotify` watch on the journal → refresh (external `nvim`/`hledger` edits flow into the UI) |

**What transfers:** the GTK4 + libadwaita + Rust shell, the two-surface design
instinct, the file-watching refresh loop, the "never silently lose data" backup
discipline, the headless-core-plus-thin-GUI crate split (a `coffer-core` that
shells out to hledger and owns the writer, a `coffer` GTK binary, maybe a
`coffer-cli`).

**What does NOT transfer:** the single-writer-SQLite-as-truth pattern. There is
no authoritative database here; hledger *is* the database. This is the single
most important thing to internalize before writing a spec.

---

## 5. Open questions (carry into spec.md)

1. **Budget model:** confirm the §3.2(C) hybrid against Brandon's real journal.
   Does the envelope-as-subaccount layout fit his existing account tree, or would
   adopting it mean restructuring his journal? (If restructuring is heavy, that
   changes the calculus toward goal budgeting or a migration assistant.)
2. **hledger coupling:** shell out to the `hledger` binary (robust, version-
   coupled, parses JSON) vs. a Haskell FFI to `hledger-lib` (faster, fragile,
   build-hostile). Default: **shell out.** Confirm.
3. **Is there any SQLite at all?** A pure shell-out-per-query design may be fast
   enough for a personal journal. A cache is an optimization, not a requirement.
   Measure before adding one.
4. **Charts on GTK4.** Still the weakest part of the native story (same open
   question the read-first sketch had). Candidates: cairo direct, a Rust plotting
   crate, `gtk-chart`. No webview. Mine Fava for *what* to chart, not *how*.
5. **Reconciliation flow.** Actual's "reconcile to a real bank balance" is a
   first-class feature. hledger has balance assertions (`= $X`). Map reconciling
   onto writing balance assertions. Worth a dedicated design pass.
6. **Multi-commodity / currencies.** Brandon's journal is likely single-currency,
   but hledger is multi-commodity by nature. Decide v1 scope.
7. **Import.** Actual imports OFX/QFX/CSV from banks. hledger has `hledger import`
   + CSV rules. Coffer could wrap that. Probably post-v1.

---

## 6. Research repos (in `research/`, gitignored, depth-1 clones)

- **`hledger/`** — the tool. Read `hledger-web/config/openapi.yaml` for the
  two-way API shape, `hledger-web` source for the add/edit/backup implementation,
  and the journal parser for what round-trips. The reference for everything.
- **`actual/`** — `packages/loot-core/` is the reactive-spreadsheet budget
  engine; `packages/desktop-client/` is the envelope UX. Mine for the
  *experience* (the month grid, assign-every-dollar, carryover, reconciliation),
  not the storage (we are journal-canonical, it is SQLite-canonical).
- **`fava/`** — the reporting/visualization reference. What to chart, how to lay
  out balance sheets and income statements, account drilldown UX.
- **`hledger-envelope-budget/`** — the worked recipe for YNAB-style envelopes on
  hledger. The concrete account-layout proof for §3.2(A)/(C).

To refresh a clone: `cd research/<repo> && git pull --depth 1`. To reclaim disk,
delete `research/` entirely; nothing here depends on it at build time.

---

## 7. Suggested next steps

1. Sit down with Brandon's actual journal and test the §3.2(C) hybrid by hand:
   draft the envelope account layout, do a month of assign/spend/carryover on
   paper (or in a scratch journal), and confirm it feels right before committing.
2. Prototype the read path: a throwaway script that shells out to
   `hledger bal --budget -O json` and friends and confirms the numbers Coffer
   needs are all reachable and fast enough.
3. Prototype the write path: append a transaction safely, numbered backup,
   re-read, confirm round-trip. This is the riskiest mechanic; de-risk it first.
4. Only then write `spec.md` and a `roadmap.md` with a hard phase split:
   **read-only dashboard first** (supersedes the old read-first sketch), then the
   envelope-assignment write layer, then reconciliation, then import.

### support

if any of this is useful to you and you'd like to chip in:

```
bc1qkge6zr45tzqfwfmvma2ylumt6mg7wlwmhr05yv
```

https://liberapay.com/bdkl/
