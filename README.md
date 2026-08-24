# Three Principles for Verifiable Automation

Notes from a hackathon, generalised into governance standards for automated
research and engineering pipelines.

---

## Context

In August 2026 I spent a day contributing to an open source policy as code
project at a Boston hackathon. Six pull requests and one issue came out of it.
The patches mattered less than the pattern behind them.

The most instructive defect was not in the scanner. It was in the generator that
builds the policy index. A category list had been hardcoded once and never
re derived from the rule pack, so one index file was never written at all, and
the consistency checker reported ten of ten files up to date. It was counting
only what it already knew existed.

Nobody was lied to. Everybody was reassured by a number that did not mean what it
appeared to mean.

This repository records the three principles I extracted, and now apply as gate
criteria across my own research infrastructure.

---

## Principle 1 — Coverage Before Correctness

A passing check reports what it examined. It does not report what it never
looked at.

Before trusting a green result, establish the denominator: how many items exist,
how many were checked, and what the difference is. A checker reporting ten of ten
files valid is meaningless if eleven files should have existed.

**Required fields:** `scope_manifest`, plus the gap enumerated rather than counted.

**Domain expression:**

| Domain | Failure mode |
|---|---|
| Genomics | A variant calling pipeline reports variants found. It does not report regions where depth was insufficient to call. A negative result over an uncovered region is an unexamined region, not a negative finding |
| Drug discovery | A druggability ranking ranks the targets assessed. It says nothing about targets the panel never included |
| Security | A vulnerability report is meaningful only against an asset inventory establishing what was in scope |
| Legal review | "No issues found" is textually identical whether a clause was reviewed and cleared or never parsed at all |

---

## Principle 2 — Derive Scope From Data, Never From a List

Hardcoded enumerations go stale silently, because nothing fails when the world
grows past them.

Any list of categories, scopes, columns, panels or jurisdictions should be
derived from the source of truth at run time. Report columns and totals must be
derived from the same manifest, not fixed independently.

**Test of compliance:** if a new category appears in the source of truth and the
output report is short a column or a row, the implementation is non compliant.

**Required fields:** `manifest_provenance`, being the source identifier plus a
retrieval timestamp.

**Domain expression:**

| Domain | Failure mode |
|---|---|
| Precision medicine | Actionability tables are hardcoded lists. A new therapeutic target emerges, the panel does not know, and no report comes back short a column |
| Security | Signature and rule sets served from a cached copy rather than the current feed |
| Legal review | Authority bound to a cached statute rather than the current authoritative text |
| Research synthesis | Identifier registries treated as static rather than resolved live |

---

## Principle 3 — A Check That Cannot Fail Is Not a Check

Every verification must declare, in advance, the observation that would make it
fail. If you cannot state what result would have caught the defect, you have
written reassurance, not verification.

**Required field:** `falsifier`, being the concrete result that would have caught
the defect, and which must be reachable by some input.

I caught myself writing exactly this failure twice during the session. The first
was a decoy test that altered the wrong characters and therefore proved nothing.
The second was a diff mislabelled as an idempotency test, replaced with a SHA-256
comparison across consecutive runs. Both were caught before any claim reached a
pull request body.

That is why this principle is on the list. A standard authored from a real
violation survives operational pressure better than one authored from theory.

**Worked positive example:** in a Mendelian randomisation study of CYP19A1 and
autism spectrum disorder, Bayesian colocalization returned a posterior probability
of 0.017, against the working hypothesis. A test capable of returning that value
is a test. An estimate with no negative branch is not.

---

## Two Supporting Rules

| Rule | Statement |
|---|---|
| Declare the dependency | Where a change depends on another unmerged change, state it in the first line of the deliverable. A reviewer cannot evaluate a diff whose base they do not know |
| Never edit downstream of a generator | A file bearing a "do not edit by hand" header is corrected at the generator, never at the output. Hand editing guarantees silent regression at the next run |

---

## Evidence

Contributions to [trustabl](https://github.com/trustabl), 24 August 2026,
Cursor Boston hackathon. Each item below is the applied instance of a principle
above.

| Submission | Principle | Summary |
|---|---|---|
| [trustabl-aws#2](https://github.com/trustabl/trustabl-aws/pull/2) | 1 | `trustabl.env` documented as an output but collected by neither CI integration. Built in the container and discarded |
| [trustabl-aws#3](https://github.com/trustabl/trustabl-aws/pull/3) | 3 | Scanner error exit code 2 collapsed to exit 1, erasing the documented distinction between a gated result and a failed scan |
| [trustabl-aws#8](https://github.com/trustabl/trustabl-aws/pull/8) | 2 | A downloaded release tarball placed ahead of system PATH, so a release could supply its own `jq` and control the gate verdict without touching the scanner binary |
| [trustabl-aws#9](https://github.com/trustabl/trustabl-aws/issues/9) | 3 | Two fail open paths: checksum fetch failure warns then executes; a missing `overall_score` defaults such that a broken report yields perfect readiness |
| [trustabl-rulebook#31](https://github.com/trustabl/trustabl-rulebook/pull/31) | 1 | The only two rationale gaps across 206 rules, located by coverage sweep rather than inspection |
| [trustabl-rulebook#40](https://github.com/trustabl/trustabl-rulebook/pull/40) | 1 | Five stale index files. The master table claimed 180 rules against 206 shipped |
| [trustabl-rulebook#46](https://github.com/trustabl/trustabl-rulebook/pull/46) | 2 | Root cause of the above. A hardcoded category list meant one index file was never generated, and totals columns could not widen for a new scope. Fixed by deriving columns from the scope order |

## Verification Method

Each claim was verified before submission rather than asserted:

| Claim type | Verification |
|---|---|
| Shell change | `bash -n` parse, plus a before and after exit code table |
| YAML change | Parser round trip on both files |
| Rule counts | Ground truth parsed independently from the rule pack, reconciled against every generated column (113 tool, 58 agent, 3 subagent, 20 skill, 12 repo = 206) |
| Idempotency | SHA-256 over all generated files across consecutive runs |
| Regex defect | Reproduced with a constructed decoy that the old code selected incorrectly and the new code selected correctly |

---

## Implementation

These principles are enforced mechanically by
[**coverage-gate**](https://github.com/fxmedus/coverage-gate), a CLI that wraps
existing checkers and refuses a pass verdict until coverage is established.

| Clause | Enforcement |
|---|---|
| 1 Coverage before correctness | A gap fails the run regardless of individual results, and is enumerated rather than counted |
| 2 Derive scope from data | Literal-list manifests are rejected at load. Empty resolution is an error, since a denominator of zero always passes |
| 3 A check that cannot fail | Missing, empty and identical-across-all falsifiers are flagged |

It is validated against the originating defect in both directions: FAIL naming
`claude_skill` on the unfixed tree, PASS at ten of ten on the fixed one.

## Licence

Text released under CC BY 4.0. Referenced contributions are governed by the
licences of their respective upstream repositories.

Julian Borges, MD, MS — [ORCID 0009-0001-9929-3135](https://orcid.org/0009-0001-9929-3135)
