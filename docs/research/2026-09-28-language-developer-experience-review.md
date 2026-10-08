# Moriarty language developer experience review

Date: 2026-09-28. Scope: `experiments/moriarty-language`, profile
`moriarty-financial-agreement-source/5` and its CLI. This is a read-only review
and proposal. It changes no profile, contract, test or acceptance gate.

The question: what would it take to make the language easy enough that a
non-technical person could use it? And would any of those changes break an
assumption the language depends on?

## Short answer

The core language should stay strict. Exact types, closed diagnostics, fixed
bounds and explicit profiles are the reasons Moriarty can be trusted with money.
Most of the ease-of-use work belongs in layers on top of that core. Those
layers break nothing:

- a human-readable diagnostic renderer
- a developer front door for the CLI
- scenario files
- a plain-English contract renderer
- term-sheet templates

Two proposals do touch the language: a built-in declaration prelude and
unit-suffixed literals. Both are possible without changing Core contract `/4`,
but each needs a new source profile and a design vote. Two ideas conflict with
a stated rule and should be done outside the language, or not at all:
inferring the profile inside the machine CLI, and generating replay IDs inside
the language.

The largest problem found is not syntax. It is feedback: errors carry no line
number and no message. That fix is small, touches no semantics, and falls under
an existing roadmap item (MC01, "source diagnostics").

## What was tested

All commands were run from `experiments/moriarty-language` on 2026-09-27/28.

### Error feedback

A one-character typo (`pre.phas` for `pre.phase`) in `loan-lifecycle.mori`:

```
$ node src/cli.ts check --profile moriarty-financial-agreement-source/5 typo.mori
{"status":"Rejected","code":"TYPE_NAME","span":{"end":"1513","kind":"source","start":"1505"},"nodePath":["1","0","0"],"workUsed":"0"}
```

Writing `100` where an amount is expected:

```
{"status":"Rejected","code":"TYPE_MISMATCH","span":{"end":"1669","kind":"source","start":"1524"},"nodePath":["2","0"],"workUsed":"0"}
```

The second span covers 145 bytes, the whole `emit` statement, not the offending
field. Neither output has a line, a column, a message or a suggestion. The
parser builds messages internally (`SuccessorSyntaxError` in
`src/successor/frontend.ts:39` takes a `message`), but the CLI drops them.

### CLI entry

```
$ node src/cli.ts --help
{"status":"CliRejected","code":"CLI_USAGE","input":"arguments"}
$ node src/cli.ts check typo.mori
{"status":"CliRejected","code":"CLI_USAGE","input":"arguments"}
```

The first line of every source file already says `profile
"moriarty-financial-agreement-source/5";`, but `--profile` is still required.
Simulation needs three files (`.mori`, snapshots JSON and state JSON), and the
snapshots must be canonical JSON. A successful `simulate` prints about 2 KB of
JSON on one line.

### The example program

`spec/successor/examples/loan-lifecycle.mori` is 192 lines:

- **Lines 4–51 (a quarter of the file)** declare the field records for
  Transfer, Repay, Originate and Accrue. The kernel fixes those shapes. The
  compiler rejects any other field set (`requireProtectedBinding` in
  `src/successor/financial-agreement-source-compiler.ts`). Only the record
  names can vary.
- **The same idea of "100 Cash" is spelled three ways:**
  `amount<Cash>(100)`, `quantity<Units<Cash,1>,0>(100)`, and `u64(60)` for
  times.
- **Each `ensures` block hard-codes one run's numbers.** For example,
  `post_principal == 80` and `post_balance("Lender") == 30`. So `repay`
  succeeds only for a payment of exactly 30 from that exact state. Any other
  payment fails `ENSURES_FAILED`. The example reads as a contract but works as
  a single test case.
- **The caller has to supply plumbing IDs** (`transferId`, `allocationId`,
  `originationId`) such as `"D1"`.
- **What happens depends on files you don't see.** The borrower's balance after
  origination is 110 because `loan-lifecycle.state.json` starts them at 10. You
  can't tell that from the source.

### Relational postconditions already work

To check whether the hard-coded numbers are a language limitation, I replaced
the nine hard-coded `repay` postconditions with two relational ones:

```
ensures post_outstanding<Cash>("Loan1") == outstanding<Cash>("Loan1") - nominal;
ensures post_balance<Cash>("Lender") == balance<Cash>("Lender") + amount<Cash>(payment);
```

`check` accepted the file. The unchanged lifecycle driver then produced the
same outcomes as the original:

| Call | Result |
| --- | --- |
| originate | `FundedExpressionPrepared` |
| accrue | `FundedExpressionPrepared` |
| accrue again | `Rejected DUPLICATE` |
| repay | `FundedExpressionPrepared` |
| settle | `FundedExpressionPrepared` |
| settle again | `Rejected GUARD_FAILED` |

So contracts that hold for any payment can be written today. This is an
authoring and documentation gap, not a language gap. The scratch files are
outside the repository and nothing in it was changed.

## Assumptions that constrain any change

These are the language's stated rules that a usability change could break.
Each is quoted or paraphrased from the named file.

| ID | Assumption | Source |
| --- | --- | --- |
| A1 | Rejection is the closed record `Rejected(code, span, nodePath, workUsed)`. "No wrapping private data in errors." The CLI adds "no exception stack, OS path/error detail or file contents" to diagnostics. | `spec/successor/semantic-contract.md` (E), `spec/successor/expression-cli.md` |
| A2 | "The --profile value is the literal profile identifier, not a path or inferred source header." There is no "profile inference". | `expression-cli.md` |
| A3 | Exact positional CLI forms. Extra or reordered flags reject `CLI_USAGE`. Check prints exactly the API record. | `expression-cli.md`; pinned by `tests/expression-cli.test.mjs` and `tests/financial-agreement-cli*.test.mjs` |
| A4 | Profiles are closed. `/1`–`/4` "keep their exact APIs, reserved names and evaluation order". New behaviour gets a new profile, and a new Core contract if constructors change. Changing a maximum changes the profile digest. | `financial-agreement-source-v5.md`, `semantic-contract.md` (V) |
| A5 | Exact type equality with no implicit casts. Quantity (nominal debt) and Amount (tokens) stay distinct: "distinct nominal debt and tokens". | `static-semantics.md` (S0), `semantic-contract.md` |
| A6 | A literal's type is fixed statically. `None`'s type is never inferred from a later use. Today, lowering is context-free: a bare integer always becomes `LitUInt` width 128 (`src/successor/financial-expression-source-types.ts:118`). | `static-semantics.md` (L, BUILD) |
| A7 | Protected operation fields are fixed by the kernel binding. Record names may be chosen by the author. | `financial-agreement-source.md` |
| A8 | Only `!=` and `emit {…}` lower to two Core nodes. "No other source form introduces more than one Core node." Source declarations add no work. The work bound B counts Core occurrences. | `static-semantics.md` (CMP), `financial-agreement-source.md` |
| A9 | Spans are real UTF-8 byte offsets. Synthetic Core carries explicit synthetic spans `[0,0)` and "cannot claim a source diagnostic". | `semantic-contract.md` (E), `lexical.md` |
| A10 | Fixed simultaneous bounds, such as 65,536 source bytes and 256 declarations. | `financial-agreement-source-v5-grammar.ebnf`, `semantic-contract.md` (V) |
| A11 | Snapshots are canonical JSON with exactly `Pre`, `Args`, `Obs` and `workInitial`. The state file is a local validated projection, not authenticated ledger state. | `expression-cli.md`, `financial-agreement-source-v5.md` |
| A12 | Transfer, allocation, origination and accrual IDs are replay keys. They are tracked in `used*Ids` and rejected on reuse (`DUPLICATE`). Evaluation is deterministic. | `financial-lifecycle.ts`, lifecycle demo |
| A13 | Contract properties, intent refinement, transition validity and history compliance are mandatory acceptance obligations. | `AGENTS.md` |
| A14 | The integer lexical form is `0` or `[1-9][0-9]*`, with no decimal point or suffix. `%` is not a token. Keywords are global and cannot be identifiers. | `lexical.md` |
| A15 | Elaboration/evaluator correspondence is one of the six SP03.3 obligations. Every new source form widens what that correspondence has to cover. | `semantic-contract.md` (theorem scope) |
| A16 | Every accepted change needs independent fresh audits on the frozen candidate. Consequential design decisions need two agreeing substantive votes. | `AGENTS.md`, develop skill |

## Proposals and their impact

Each proposal lists what would change, which files are involved, and which
assumptions it keeps, strains or breaks.

### P1. Human-readable diagnostics

**What changes.** Add a renderer that turns a `Rejected` record plus the
original source into this:

```
loan.mori:58:20  error TYPE_NAME
  There is no state field called "phas".
     |     next.phase = pre.phas + 1;
     |                  ^^^^^^^^
  Did you mean "phase"?
```

- Line and column come from the byte span and the source bytes. Because spans
  are UTF-8 byte offsets, the column must count scalars, not bytes, for
  multibyte text.
- Each code needs a message template. The successor sources contain about 140
  distinct upper-case code-like strings. The real diagnostic catalog is a
  subset of these, and a test should require a template for every code.
- "Did you mean" needs the names in scope. The `Rejected` record doesn't carry
  them. The renderer can re-run the public `parse`/`check` on the same source
  to collect declared names, then rank them by edit distance.
- Syntax errors already carry messages internally
  (`src/successor/frontend.ts:39`), so those can be reused.

**Where it lives.** New `src/successor/diagnostic-render.ts`. It is exposed
only through a human mode (P2), never through the API record.

**Also worth fixing.** Some spans are wider than the error they report. The
amount-for-integer mistake above points at the whole `emit` statement. Narrowing
a span to the offending field changes a diagnostic value, so it is a change to
the profile's diagnostics, not to presentation. It needs its own tests and a
new profile, or an explicit amendment of `/5`. See A4.

**Impact on assumptions:**

- **A1: keeps, if the renderer is only a view.** The machine record stays
  closed. The renderer echoes source text to the author's own terminal. That
  text must never go into records, receipts, logs or anything `review` ingests.
- **A3: keeps, if human output is opt-in.** Changing the default output breaks
  the CLI contract and every CLI test. An added flag is still a CLI contract
  amendment, because unknown flags currently reject.
- **A9: needs handling.** A synthetic `[0,0)` span must render as "no source
  location", not as line 1, column 1.

**Verdict.** No language assumption breaks. The CLI contract needs a scoped
amendment.

### P2. A developer front door for the CLI

**What changes.** Add a separate entry point, for example `src/dev-cli.ts`
exposed as `moriarty`, with these commands:

- `moriarty check file.mori`: reads the profile from the header, prints
  `using profile …/5`, and calls the same API with that explicit profile.
- `moriarty --help` and `moriarty <command> --help`.
- `moriarty new loan`: writes a scaffold that passes `check`.
- `moriarty run scenario.mori-scenario` (see P5).
- Human output by default, with `--json` for the exact current records.

The machine CLI `src/cli.ts` stays byte-for-byte as it is.

**Why a separate entry.** A2 exists so that a source file cannot choose its
own semantics. The caller pins the profile. That matters for admission,
backends and review. It doesn't matter for an author checking their own draft.
Keeping inference out of `cli.ts` preserves A2 where it matters.

**Impact on assumptions:**

- **A2: breaks if inference is added to `cli.ts`; keeps with a separate entry.**
  This is a design decision for A16. The record should state that `moriarty`
  is an authoring convenience and that pipelines must use the pinned CLI.
- **A3: keeps.** `cli.ts` is untouched.

**Verdict.** Safe as a separate tool. It must not replace the pinned CLI in
scripts, CI or review.

### P3. A built-in declaration prelude

**What changes.** Add a new profile, `…/6`, in which the four protected
operations and their records are built in. An agreement only states what
varies: units, assets, state and actions. About a quarter of the example file
disappears.

**Where it changes.**

- New `financial-agreement-source-v6*.ts` files and a v6 grammar.
- The compiler injects the canonical records before `requireProtectedBinding`.
- New spec and README sections, and a `readme-grammar.test.mjs` update.

**Impact on assumptions:**

- **A7: keeps.** The binding is identical and only the fixed shape is supplied.
- **A4: requires a new profile.** `/1`–`/5` must not change. The Core contract
  can stay `/4`, because no constructor changes.
- **A8: keeps.** Declarations add no work.
- **A9: needs rules.** Prelude declarations have no source bytes. Any
  diagnostic about them needs a defined target: the `agreement` span or a
  synthetic span. It must never be a fabricated offset.
- **A10: needs a decision.** Do prelude declarations count toward the 256
  declarations? I recommend they don't count against source bounds, while the
  elaborated schema still meets every aggregate bound.
- **Name collisions need a rule.** For example, an author declaring their own
  `TransferFields`: reject, or allow shadowing.

**Evidence to require.** Elaborating the `/6` form of `loan-lifecycle` must
produce a schema and Core byte-identical to the `/5` form, once the canonical
names match. Without that, the prelude has quietly changed semantics.

**Verdict.** Doesn't break the language. It is a consequential source design
change, so it needs a vote and fresh audits (A16), and it widens the A15
correspondence.

### P4. Readable literals and contextual literal typing

**What changes.** Add sugar that elaborates to exactly the Core nodes used
today:

| Sugar | Today | Core node |
| --- | --- | --- |
| `100 Cash` | `amount<Cash>(100)` | `LitAmount` (1 node) |
| `owed 100 Cash` (spelling to be decided) | `quantity<Units<Cash,1>,0>(100)` | `LitQuantity` (1 node) |
| `60` where a `UInt64` is expected | `u64(60)` | `LitUInt` width 64 (1 node) |

**Impact on assumptions:**

- **A5: breaks unless there are two spellings.** A single `100 Cash` for both
  amounts and nominal quantities would erase the token/debt distinction the
  language exists to keep. Authors must still say which one they mean. Only
  the type syntax becomes easier to write.
- **A6: needs a design change.** Today, lowering is context-free and runs
  before type checking (`financial-expression-source-types.ts:118`). Typing
  `60` as `UInt64` from the expected field type needs type-directed
  elaboration (bidirectional checking). The type is still fixed statically,
  so this is not an implicit cast. But the pipeline changes, and so does where
  some diagnostics fire. An out-of-range literal would move from a runtime
  range check to `TYPE_LITERAL`. Stage ordering (A1, A15) must be restated and
  retested.
- **A14: breaks for decimals and percentages.** `1.5 Cash`, `10%` and `60s`
  need new lexical forms, and `%` is not a token. Decimal literals also need an
  exact scale rule (for example, `1.50` means scale 2 and mantissa 150) and an
  answer to whether `1.5` and `1.50` are the same type. Leave these out of the
  first version.
- **A8: keeps.** Each sugar form is exactly one Core node.
- **A4: requires a new profile,** plus formatter rules for canonical output of
  the new forms.

**Verdict.** The unit-suffix and contextual-integer parts are compatible, with a
design vote. The decimal and percent parts need lexical changes and should come
later.

### P5. Scenario files instead of hard-coded postconditions

**What changes.**

- **Rewrite the examples.** Use relational `ensures`, which already work (see
  above), so each action states a rule that holds for any valid input.
- **Add a scenario format.** One readable file sets the parties, starting
  balances and a sequence of calls with expectations:

  ```
  scenario "partial repayment" {
    start Lender 100 Cash, Borrower 10 Cash
    originate
    accrue at 1060
    repay 30 Cash
    expect Loan1 outstanding 80 Cash
  }
  ```

- **Add a runner.** It compiles each step to the exact canonical snapshot and
  state JSON, calls the public `evaluate`, and chains `post`/`financialPost`
  into the next call. `examples/loan-lifecycle.mjs` already does this by hand.
  The runner also supplies fresh replay IDs (see P6).

**Impact on assumptions:**

- **A11: keeps.** The runner emits exactly the admitted canonical inputs. The
  state stays a local projection and must be labelled as such, never as ledger
  state.
- **A13: improves.** A contract whose `ensures` fit only one run doesn't
  express the contract's properties. Relational postconditions do.
- **Language: no change.**

**Verdict.** No language change. This is the largest usability gain for the
least risk.

### P6. Automatic IDs

**Inside the language: breaks A12.** IDs are replay keys. A source-level
"fresh ID" would have to be either nondeterministic, which breaks determinism,
or derived from a counter or hash. A derived ID is a new kernel rule, changes
what the proof has to bind, and touches `financial-lifecycle.ts` and the K
side.

**In tooling: keeps everything.** The scenario runner (P5) or the front door
(P2) generates fresh IDs and passes them as arguments, as the lifecycle demo
already does.

**Verdict.** Do it only in tooling.

### P7. Term-sheet templates

**What changes.** A form or template (principal, rate, period, cap, parties)
generates a `.mori` file, and the generated file must pass `check`. No
language change is needed.

**Risks:**

- The template must not turn into a second, unreviewed language.
- The generated `.mori` file is the artifact that is hashed, reviewed and
  admitted. Keep it visible, and never admit a template directly.
- The template must not choose the profile. The generated file declares it and
  the pinned CLI checks it.

### P8. Plain-English rendering

**What changes.** A deterministic renderer from the checked program (the bound
or elaborated form, not the raw text) produces plain English, for example:
"Lender transfers 100 Cash to Borrower. The loan accrues 1/10 per 60-second
period, rounded down, capped at 110 owed. Repayment is applied to interest
first. After repayment, the outstanding debt falls by exactly the amount
paid."

**Rules:**

- Render every `requires` and `ensures`. Leaving a clause out would mislead the
  reader.
- Don't use an LLM for this step. The English is what a non-technical reviewer
  approves, so it must be a faithful function of the program.
- Label the output non-normative. It explains the contract. It doesn't
  replace audits or acceptance.
- Add tests that pin the rendering of each constructor and financial read, so
  that a change to the language shows up as a change to the English.

**Impact on assumptions:** none. It reads public elaboration output only.

### P9. Visual simulation and a playground

**What changes.** A timeline of balances, debt and allowances across a
scenario, with "what if" controls. `site/` is already a React single-page app.

- The v5 frontend APIs take strings. In the files I checked, only
  `src/registered-bounds.ts` (the atomic profile) reads the filesystem.
  Before promising an in-browser checker, confirm that the whole v5 import
  graph runs in a browser after TypeScript compilation.
- The browser must present results as local simulation. The same "not ledger
  settlement" wording the README uses applies (A11).

**Impact on assumptions:** none, if it calls the public APIs only.

### P10. LLM-assisted drafting

**What changes.** A conversation drafts `.mori` or a template (P7). The
checker accepts or rejects it, and the P8 English is what the person approves.
The language's strictness helps here: the checker is the gate, not the model.

**Impact on assumptions:** none, provided the LLM never feeds the English
rendering or any admission step.

## Summary matrix

Legend: ✓ keeps, △ keeps with a stated rule or decision, ✗ breaks as naively
stated. Blank means not relevant.

| Proposal | A1 | A2 | A3 | A4 | A5 | A6 | A8 | A9 | A10 | A11 | A12 | A13 | A14 | Language change |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| P1 diagnostics renderer | △ | | △ | ✓ | | | | △ | | | | | | None (span narrowing: profile change) |
| P2 dev front door | ✓ | △ (✗ in `cli.ts`) | ✓ | | | | | | | | | | | None |
| P3 prelude | ✓ | | | △ new profile | | | ✓ | △ | △ | | | | | New source profile, same Core |
| P4a unit/contextual literals | | | | △ new profile | △ two spellings | △ design change | ✓ | | | | | | | New source profile, same Core |
| P4b decimals, %, durations | | | | △ | | △ | ✓ | | | | | | ✗ | Lexical change |
| P5 scenarios + relational ensures | | | | | | | | | | ✓ | ✓ | ✓ | | None |
| P6 auto IDs in language | | | | | | | | | | | ✗ | | | Kernel/proof change |
| P6 auto IDs in tooling | | | | | | | | | | | ✓ | | | None |
| P7 templates | ✓ | ✓ | ✓ | ✓ | ✓ | | | | | | | | | None |
| P8 English rendering | ✓ | | | | | | | | | | | | | None |
| P9 playground | | | | | | | | | | △ labelling | | | | None |
| P10 LLM drafting | | | | | | | | | | | | | | None |

A7, A15 and A16 apply across all rows. A7 holds for every proposal. A15 widens
for P3 and P4. A16 applies wherever a profile or CLI contract changes.

## Recommended order

1. **Rewrite the examples to use relational `ensures` (P5, first half).** This
   is documentation only and fixes the "contract that works once" problem.
2. **Build the diagnostic renderer and the separate front door (P1, P2).**
   These fall under MC01's "source diagnostics" checkbox in `ROADMAP.md`. The
   machine CLI is unchanged, and the human mode is a scoped CLI contract
   amendment.
3. **Add the scenario runner with tooling-generated IDs (P5, P6 in tooling).**
4. **Add the English renderer (P8).** This is the piece that actually lets a
   non-technical person review a contract.
5. **Propose a `/6` source profile (P3 prelude, then P4a literals).** Use the
   normal design vote, fresh audits, and byte-identical Core evidence against
   `/5`.
6. **Add templates, the playground and LLM drafting (P7, P9, P10)** once 1–5
   exist. They are only as good as the diagnostics and English rendering
   underneath them.

Leave out entirely: profile inference in the pinned CLI, in-language ID
generation, and any single literal spelling that merges Amount and Quantity.

## Scope note

The current development focus is I2/MC02 complete-effect loan/swap
integration. The develop workflow defers productization unless the user
changes scope. Items 1 and 2 can run alongside that work, because they touch
no semantics. Items 3–6 are a scope change and should be recorded as one
before work starts. This review is not a design vote, an audit or acceptance
evidence for any profile.

## Open questions

1. Should span narrowing for the existing `/5` diagnostics be an amendment to
   `/5`, or wait for `/6`?
2. How should nominal quantities be spelled (`owed 100 Cash`,
   `100 Cash nominal`, or something else)?
3. Should prelude declarations count toward the 256-declaration source bound?
4. Is the scenario format a checked-in spec with its own grammar, or a tooling
   convention? A grammar is easier to review, but it is one more thing to
   freeze.
5. Who reviews the English templates for P8? Wording errors there reach
   exactly the readers least able to catch them.
