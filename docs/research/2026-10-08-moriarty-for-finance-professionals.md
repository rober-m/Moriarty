# Moriarty for finance professionals

Date: 2026-10-08. Scope: how to make Moriarty usable by finance specialists
who build DeFi protocols on Midnight without writing software. This is a
read-only review and proposal. It changes no profile, contract, test or
acceptance gate. It supersedes the October 7 version of this note, which
measured the Source/5 profile before the authoring beta, the Rust toolchain
and the U0 to U7 roadmap had been pulled into this checkout. The
[September 28 developer-experience review](2026-09-28-language-developer-experience-review.md)
remains the assumption-by-assumption analysis of the Source/5 proposals.

## Summary

**Recommendation.** Keep the core exactly as strict as it is, and build the
finance-facing experience as layers above the authoring beta. That is now the
product's own position. The September 19 product contract says 'natural-language
wishes require explicit formalization and an inspectable signing display' and
that 'an exact bounded static checker can improve authoring in parallel'.

**What changed since October 7.** The repository now ships the engineer-facing
half of what the earlier note proposed: an authoring beta with a readable
checker, decimal unit literals, scenario and test files, starter templates, a
signing display, a language server and a tutorial site, plus a Rust toolchain
whose diagnostics name the mistake, point at it, and suggest the fix. Two of
the seven problems in the October 7 note are solved, and three are half
solved.

**What remains is the finance-facing half**, and nobody has started it:

- The beta speaks in atoms, scales, nonces and heads. It has no rates,
  periods, day counts or rounding words, and it does not compute interest.
- A scenario is one action against one snapshot. There is no sequence, no
  cash-flow table and no way to reconcile a run against a spreadsheet.
- There is a labeled display of a signed intent, but no rendering of an
  agreement that a non-engineer could approve.
- The only usability evidence is six model-run trials. The result file says
  'these are artificial-user trials, not a human usability study' and 'a
  measured human onboarding study remains unperformed'. No file since
  September 12 considers a finance professional or a non-engineer as a user.

This note names each gap and a way to close it. A study with finance users
comes last, on the assembled layers, because it measures the rest and is the
only item that produces evidence nobody has. None of it touches the U1/U2
implementation slice or weakens any gate.

## Where Moriarty stands

All items were checked against the repository on October 8, 2026. *Exists*
means implemented and tested locally. *Preview* means executed on Midnight
Preview with two independent audits. *Open* means specified only.

| Area | Status | What it means for this plan |
| --- | --- | --- |
| Product and roadmap | The September 19 [product contract](../MORIARTY-PRODUCT-CONTRACT.md) defines Moriarty as 'a programming language for all DeFi developers on Midnight', compiled to ZKIRv3, with developer tooling inside the product. The [roadmap](../../ROADMAP.md) is one U0 to U7 sequence. MC08 and SP12 map to U7, the independent developer release. The current focus is 'one implementation slice on U1/U2'. | Authoring work is a precursor track, like the beta. It never gates a U milestone and must never touch the U1/U2 slice. |
| Authoring beta (Exists) | [`packages/moriarty-beta`](../../packages/moriarty-beta/README.md), profile `moriarty-beta/1`. Commands: `init`, `check`, `fmt`, `inspect`, `expand`, `simulate`, `test`, `intent`, `verify-intent`, `lsp`, `mcp`. Decimal unit literals such as `10.00 USD` with a declared scale. Named accounts, assets and obligations. An `intent` block with gross cap, fee cap and net floor. Only transfer and interest-first repayment prepare locally, and the result is `PreparedUnqualified`. The eight DeFi family examples are structure-only (`SpecifiedOnly`). | This is the surface to build on. Source/5 is now described as 'the current local loan profile'. |
| Rust toolchain (Exists) | [`crates/`](../../RUST_GUIDELINES.md): lexer, parser, checker, formatter, diagnostics and a `mori` binary. `check` and `format` are implemented; the other nine subcommands are stubs. Diagnostics carry a code, a message, file:line:column, a source excerpt with labels, and a help line with corrected code. `RUST_GUIDELINES.md` states 'the surface syntax is not frozen'. | The feedback layer the October 7 note asked for exists. Vocabulary changes go into this surface through its own review, not into a new profile number. |
| Source/5 (Exists, unchanged) | The lifecycle kernel (originate, accrue, repay, settle), the pinned CLI and the loan example are byte-identical to September 12. The pinned CLI still returns a bare code and byte offsets. | Source/5 is where accrual terms (numerator, denominator, rounding, period) already exist. The beta has none. Any finance vocabulary must elaborate to terms a kernel already enforces. |
| Midnight (Preview) | The [September 17 loan run](../../deliverables/preview-loan-2026-09-17/recovery-run01/RESULT.md) finalized deploy, initialize, accrue and settle at blocks 908628 to 908968, paid 533,972,602 test-asset units to the lender with 4,500,000,000 notional outstanding, exited zero, and has both independent result approvals. The September 10 swap result stands. | The chain path is proven for fixed programs. Preview remains the hard gate. |
| Native proof | On October 2 a fixed fixture produced a finalized 6,336-byte native proof. [R4](../../wiki-llm/native-financial-kernel-2026-10-01/R4-R5-REVIEWED-SUCCESS.md) applied it 'with default Real proof verification enabled and returned Success'; R5 'independently verified the retained proof'. It is 'a manually specialized kernel with a fixed production Beta/Core handoff and trusted public development genesis', and 'no public transaction was submitted'. | Independent of authoring. The layers do not touch the proof path. |
| Open | U2 general source-to-Midnight path, U4 native history and private composition, U6 ACTUS and DeFi conformance, U7 release. | Templates with finance math should be built as the first consumer of the U6 libraries, not ahead of them. |
| Test suites (Exists) | TypeScript language experiment: 938 tests, 934 pass; the four failures are Compact lowering tests that need the Compact compiler and runtime installed. Beta package: 193 tests, 115 pass, 12 skipped; all 66 failures need the native signature verifier binary, which is not built on this machine. Rust workspace: 146 tests, all pass. | The baseline is stable enough to add layers against. |
| Usability evidence | [Six model-run trials](../../wiki-llm/beta-language-2026-09-30/DEVEX-RESULT.md) (three Sonnet 5.5, three Grok 4.7): 65 cases, 12 successful local money cases. Findings included unclear units and IDs, failed tests that hid the differing field, misleading formatting and a wrong diagnostic. Repairs shipped in later versions. No human has been observed using the tool. | Our audience is unclaimed. A study with finance users on the finished layers is the evidence that would close the loop. |

## What a finance professional meets today

Every observation below was reproduced on October 8, 2026 against the beta
starter that `mori init` writes, using both the TypeScript beta CLI and the
Rust `mori` binary built from `crates/`. All are experiment observations.

### The starter

`mori init` writes a 17-line invoice agreement, a JSON scenario, a test
manifest and a README. Three of the 17 lines are signed plumbing: key, nonce,
pre-head, source hash, policy digest, and six explicit empty fields. That is a
deliberate choice; the guide says 'there are no hidden financial defaults'.
The starter's own comment says its IDs and hashes are unverified examples.

```
const price: Qty<USD> = 10.00 USD;
const fee = 0.10 USD;
intent Payment = {
  domain: Preview, asset: USD, signer: Buyer, key: "key1", nonce: "n1", pre_head: "h0",
  valid: rounds(domain: Preview, from: 0, to: 10), gross_cap: price + fee, fee_cap: fee, net_floor: price,
  operation: transfer(from: Buyer, to: Seller, fee_to: Treasury, value: price, fee: fee),
  source_hash: "src1", policy_digest: "policy1", failure: SuccessOnly,
  observations: [], disclosures: [], retained_effects: [], retained_duties: [], delegation: None, recovery: None,
};
```

### Errors now explain themselves

A one-character typo, `Treasry` for `Treasury`, through the Rust binary:

```
mori::check::unknown_name
  × I cannot find a declaration named `Treasry`.
    ╭─[typo.mori:13:56]
 13 │   operation: transfer(from: Buyer, to: Seller, fee_to: Treasry, value: price, fee: fee),
    ·                                                        ───┬───
    ·                                                           ╰── not declared above
  help: Did you mean `Treasury`?
            operation: transfer(from: Buyer, to: Seller, fee_to: Treasury,
        value: price, fee: fee),
```

A plain `10` where `0.10 USD` was meant:

```
mori::check::number_and_amount
  × I cannot add a `USD` amount and a plain number.
    ╭─[int.mori:12:71]
 12 │   valid: rounds(domain: Preview, from: 0, to: 10), gross_cap: price + fee, fee_cap: fee, net_floor: price,
    ·                                                               ──┬──   ─┬─
    ·                                                                 │      ╰── this is a plain number
    ·                                                                 ╰── this is a `USD` amount
  help: A plain number and an amount cannot be combined. Use amounts of the
        same asset on both sides.
```

The TypeScript beta CLI gives a shorter form of the same thing:
`BETA_REFERENCE at 13:56: Unknown or forward reference Treasry`. Both read the
profile from the file, both exit nonzero, and `check --json` gives the same
diagnostics with byte spans for machines. This closes problems 1 and 2 of the
October 7 note for the beta surface.

### Scenarios and tests exist, one step at a time

The scenario is a closed JSON record: balances, allowance, replay state, work
counters and, for a repayment, one obligation. Amounts are atom strings, so a
balance of 2,000.00 USD at scale 2 is written `"200000"`. A test case binds
one source, one action, one scenario and an exact expectation. Changing the
expected owner balance from 8990 to 8991 reports:

```
"pointer": "/post/balances/0/amount", "expected": "8991", "actual": "8990"
```

`simulate` prints the result as JSON, about 4.9 KB for the starter. One action
runs against one snapshot; there is no sequence of events and no table.

### The vocabulary stops at amounts

The beta has decimal unit literals, named parties and named obligations, and
that is a real step. It has nothing above them. The guide states 'no floating
point, division, rounding or implicit asset conversion exists'. Accrued
interest is an input in the scenario, not something the language computes;
'unused source constants named principal or accrued do not constrain scenario
accounting'. The repayment amount and a cash price share one type, `Qty<USD>`,
so the owed-versus-paid distinction that Source/5 enforces through two types
is not visible in the beta surface. The eight DeFi family files check
structure only; expansion and simulation reject them as `SpecifiedOnly`.

### Source/5, for the record

The Source/5 observations in the October 7 note reproduce byte for byte: the
pinned CLI returns `TYPE_NAME` with byte offsets and no message; `--help`
rejects; the 178-line loan example spends 47 lines on fixed record shapes;
and replacing the seven hard-coded financial postconditions in `repay` with
two relational ones passes the checker, reproduces all six lifecycle outcomes
and accepts a payment of 50 that the original rejects. They matter less now,
because new work targets the beta, but they are the evidence that contracts
can state rules rather than one run.

## The principle: strict core, finance layers

The properties that make the core hard to read are the properties that make
settlement provable, and the product contract now says so in its own words.
Signed terms must be explicit and visible, which is why the starter carries
its plumbing. Amounts are exact atoms, which is why there is no rounding
until a rounding rule is a signed term with a named beneficiary. A rejection
is a closed record so that private data never leaks through an error.

So every layer below is a deterministic function over the checked program or
its public output. It never changes what the checker accepts. It never puts a
language model in the path a person approves. Where a layer needs the surface
to change, the change goes into the beta grammar, which is not frozen, through
the same review the Rust port uses, and each new form elaborates to terms a
kernel already enforces. No layer gets a new profile number: `/6` is already
used by the MIL/4 Source/6 stage syntax, and the U0 proposal separately
reserved a `/6` for the Aeon refinement grammar and deferred it until after U2.

## The gaps and how to close them

Each item is a recommendation. Evidence lines are experiment or repository
observations. The numbering is for reference, not an order of work.

### 1. The vocabulary stops at amounts

- **The gap.** Decimal unit literals, named parties and named obligations
  exist. Rates, periods, day counts and rounding words do not. Interest is a
  scenario input. One type covers both debt and cash. Three of the starter's
  17 lines are signed plumbing, by design.
- **How to close it.** 'Rate 10% per period, rounded down in the lender's
  favor.' 'Period 30 days from March 10, 2026.' '100 USD paid' for cash that
  moves and '100 USD owed' for debt that is recorded. Day count and rounding
  written as words and required, never defaulted. This goes into the beta
  grammar and the Rust checker through the review `RUST_GUIDELINES.md`
  already describes. Rates and periods elaborate to the accrual terms the
  Source/5 lifecycle kernel already enforces (numerator, denominator,
  rounding, period) once the beta exposes accrual; until then they are
  rejected as unsupported, exactly as the family sketches are today. The
  beta's own full-language horizon already proposes a rounding enum with a
  named beneficiary; this gives it a surface spelling.
- **Guardrail.** Two spellings for owed and paid become mandatory, restoring
  the distinction Source/5 keeps through types. Every new form elaborates to
  exact atoms and existing kernel terms. No new profile number.

### 2. One action, one snapshot, JSON in and out

- **The gap.** A scenario is one action against one snapshot, with amounts as
  atom strings. A test is one exact expectation. Output is JSON. There is no
  sequence of events, no table and no way to reconcile against a spreadsheet.
- **How to close it.** A scenario that names the parties and opening balances
  once, then a sequence of events with expectations: originate, accrue at a
  date, repay 30 USD, expect outstanding 80 USD. The output is a dated table
  of cash flows, debt and balances in decimals, with a CSV export. A loan
  template, `mori init --template loan`, asks the term-sheet questions and
  writes the signed plumbing as placeholders marked unverified, as the
  starter already marks its own. Tooling only: the runner chains each
  post-state into the next scenario, which the beta's single-action
  `simulate` and `test` already make possible, and renders atoms as decimals
  with the formatter the signing display already uses.
- **Guardrail.** Output is labeled local stipulation, never ledger state.
  Replay keys and heads stay explicit in the generated files; the template
  fills them, it does not hide them.

### 3. Invariants are enforced but invisible

- **The gap.** The kernel's fixed effects carry the invariants (conservation,
  caps, interest before principal), and the roadmap already says 'obligation
  and trust reports accompany U0–U2'. Nothing shows a finance professional
  which invariants held.
- **How to close it.** Invariants chosen by name: conservation of value,
  outstanding never exceeds the cap, payments sum to principal plus interest,
  no negative balance, interest before principal. The report says which hold,
  which were not checked, and why. For Source/5 the mechanism is relational
  postconditions, which already work. For the beta the kernel's fixed effects
  carry the invariants and the report makes them visible.
- **Guardrail.** A report names the predicate it checked. Unknown, timeout or
  unsupported never renders as success, which is the product contract's rule.

### 4. There is a signing display, but no agreement to approve

- **The gap.** The beta has a labeled, escape-safe display of every signed
  statement field with decimal amounts. It covers one signed intent, as rows.
  Nothing renders an agreement a non-engineer could sign off on.
- **How to close it.** Extend the display from the signed intent to the whole
  agreement and render prose: 'Buyer pays Seller 10.00 USD with a 0.10 USD
  fee to Treasury. The most that can leave Buyer's account is 10.10 USD.
  Seller receives at least 10.00 USD. The intent is valid for rounds 0 to 10
  and only succeeds completely or not at all.' For a loan: the rate, the
  period, the rounding rule and its beneficiary, and what remains owed after
  each event. Tests pin the wording of every construct so a grammar change
  shows up as a wording change.
- **Guardrail.** Every bound and every duty is rendered; leaving one out
  misleads the reader. The rendering is labeled non-normative. No language
  model produces it.

### 5. Templates have no finance math

- **The gap.** Two starters exist (transfer, repay). The eight family files
  are structure-only and reject as `SpecifiedOnly`.
- **How to close it.** A starting point for the protocols this audience
  builds: a constant-product pool, a collateralized loan, a vault with
  deposit, mint, withdraw and redeem, then ACTUS PAM, LAM and ANN. Built as
  the first consumer of the U6 libraries, so the generated `.mori` file is
  checked by the same pipeline as any program.
- **Guardrail.** Templates never choose the profile and are never admitted
  directly. A template language must not become a second, unreviewed language.

### 6. Nobody has watched a human use it

- **The gap.** The only usability evidence is six model-run trials. They
  found unclear units and IDs, failed tests that hid the differing field, and
  misleading formatting, and the fixes shipped. A human will find the next
  class, and a finance professional will find a different one.
- **How to close it.** Once the layers above exist, a study with three to
  five finance professionals who do not write software. Three tasks: read
  the agreement rendering of a loan and find a deliberately wrong rounding
  rule; write a scenario from a term sheet and read the cash-flow table
  without help; fix a typo from the error message alone. Record time, where
  they stall, and what they say the words mean. This is the validation step
  for everything above, so it comes last. A short dry run on the beta as it
  ships today would calibrate the tasks, but it is not the deliverable.
- **Guardrail.** Label every result as observation of a beta with
  `PreparedUnqualified` output. The study does not establish settlement, and
  it is not an SP12 pilot.

## What we will not do

- Start a new source profile number. `/6` is taken twice.
- Hide signed terms. Templates fill placeholders and mark them unverified.
- Let a language model write approval text or touch any admission step. The
  beta's AI tools are read-only, and that is the right boundary.
- Claim anything beyond `PreparedUnqualified` for local results.
- Touch the U1/U2 implementation slice, or let any of this be reported as
  progress on a U milestone.

## Dependencies and scope

This is not a sequence; it is what each item needs before it can start.

- Multi-step scenarios, the cash-flow table, trust reports and the agreement
  rendering need no language change and read public output only. Any of them
  can start on the beta as it is.
- Finance vocabulary needs the beta's own grammar and checker review. The
  surface is not frozen.
- The loan template needs the vocabulary, or it can only fill what the beta
  can already say.
- Templates with finance math wait for the U6 libraries.
- The usability study runs on whatever has been assembled, and last, because
  it measures the rest.

All of these sit under the roadmap's 'authoring tools and libraries' heading
as precursors, the same status the beta has. None closes or delays a U
milestone. The current implementation slice on U1/U2 is untouched.

## Decisions needed

1. Whether finance vocabulary goes into `moriarty-beta/1` as an extension or
   waits for a `moriarty-beta/2`. The Rust guidelines say the surface is not
   frozen, so the recommendation is the former.
2. Template order: DeFi first (pool, loan, vault) or ACTUS first. The
   recommendation is DeFi first, because the loan and the swap have Preview
   evidence.
3. Who owns the English wording. It should be a finance reviewer, not an
   engineer.
4. Who recruits the finance users for the study once the layers exist.

## Evidence

Commands were run on October 8, 2026 from the repository root unless noted.
The beta was run from source with `node packages/moriarty-beta/src/cli.ts`;
the Rust binary was built with `cargo build --release -p mori`.

| Claim | Source or command | Kind |
| --- | --- | --- |
| Starter is 17 lines with three plumbing lines | `mori init DIR`, then `wc -l` and grep for key, nonce, pre_head, source_hash, policy_digest and the empty fields | Experiment observation |
| Rust diagnostics for a typo and a plain number | `target/release/mori check --color never typo.mori` and `int.mori` | Experiment observation |
| TypeScript beta diagnostic for the same typo | `node packages/moriarty-beta/src/cli.ts check typo.mori` | Experiment observation |
| Test mismatch pointer | Edit the owner balance in `mori.tests.json` to 8991, then `mori test DIR` | Experiment observation |
| `simulate` output size | `mori simulate invoice.mori --action pay --scenario scenario.json \| wc -c` gives 4,909 bytes | Experiment observation |
| No rounding, division or computed accrual | `packages/moriarty-beta/GETTING-STARTED.md` | Repository observation |
| Eight family examples are structure-only | `packages/moriarty-beta/examples/README.md` | Repository observation |
| Rust `check` and `format` implemented, nine commands stubbed, surface not frozen | `crates/mori/src/commands/*.rs`; `RUST_GUIDELINES.md` | Repository observation |
| Source/5 unchanged and its observations reproduce | `git diff cd240b8e HEAD -- experiments/moriarty-language/src/cli.ts spec/successor/examples/loan-lifecycle.mori`; the October 7 commands rerun | Experiment observation |
| Relational postconditions accept a payment of 50 | Lifecycle run through `createFinancialAgreementSourceV5().evaluate` on the original and the relational copy | Experiment observation |
| Product contract and roadmap quotes | `docs/MORIARTY-PRODUCT-CONTRACT.md`; `ROADMAP.md`; `openspec/changes/consolidated-language-kernel/traceability.md`; `wiki-llm/ALIGNMENT-REVIEW-2026-10-01.md` | Repository observation |
| `/6` is used by MIL/4 Source/6 and reserved for the Aeon grammar | `experiments/moriarty-language/spec/successor/financial-agreement-source-v6.md`; `deliverables/u0-study-2026-09-28/UNIFIED-PROPOSAL.md` | Repository observation |
| Preview loan, September 17 | `deliverables/preview-loan-2026-09-17/recovery-run01/RESULT.md` | Repository observation |
| Native proof R4/R5, October 2 | `wiki-llm/native-financial-kernel-2026-10-01/R4-R5-REVIEWED-SUCCESS.md` | Repository observation |
| Model-run trials, no human study | `wiki-llm/beta-language-2026-09-30/DEVEX-RESULT.md` | Repository observation |
| Rounding with a named beneficiary is already proposed | `wiki-llm/beta-language-2026-09-30/FULL-LANGUAGE-HORIZON.md` | Repository observation |
| Test suite totals | `npm test` in `experiments/moriarty-language` and `packages/moriarty-beta`; `cargo test --release` | Experiment observation |

## What changed from the October 7 version

- The measured surface is the authoring beta and the Rust toolchain, not
  Source/5. Source/5 is kept as an appendix.
- Problems 1 and 2 (diagnostics, front door) are solved for the beta and are
  removed from the plan.
- Problem 3 (single-run contracts) becomes trust reports; the relational
  evidence stays as Source/5 evidence.
- Problem 4 (vocabulary) narrows to what the beta lacks: rates, periods, day
  count, rounding with beneficiary, owed versus paid.
- Problem 5 (hidden inputs) narrows to multi-step scenarios and a cash-flow
  table over the beta's existing formats.
- Problem 6 (nothing to approve) becomes an extension of the existing signing
  display.
- The `/6` profile proposal is withdrawn; the number is taken.
- A human usability study is added as the validation step, run last on the
  assembled layers.
- The phased plan is replaced by gaps with solutions and a dependencies
  note. No order of work is prescribed.
- Status rows for the product contract, the U0 to U7 roadmap, the September
  17 Preview run, the October 2 native proof and the new test totals replace
  the September 12 picture.

Nothing in this note is a design vote, an audit, or acceptance evidence for
any profile.
