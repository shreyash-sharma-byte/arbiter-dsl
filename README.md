# Arbiter — a verifiable DSL for business decision logic

> **Status: design draft.** There is no implementation yet. This repository contains a language specification and the prior-art research that motivates it.

## The idea

Business rules are increasingly written by LLMs from prose specs. The problem is that generated rules *look* correct: they parse, they pass their unit tests, and they are quietly inconsistent, incomplete, or order-dependent in ways that only surface in production.

The fix is not better tests. Tests sample a handful of inputs; a decision table over eight typed columns has an input space no test suite can cover. The fix is a language whose important properties are **statically decidable**, so they can be proven for *all* inputs rather than sampled:

| Property | Claim |
|---|---|
| **Totality** | Every input in the declared domain resolves to an outcome |
| **Boundedness** | Every evaluation terminates in bounded cost |
| **Determinism** | The same input always yields the same outcome, independent of rule order |

Everything else in the design follows from preserving those three. The language deliberately excludes loops, recursion, side effects, nulls, floating point, and user-defined functions — each removal buys back a property.

**The verifier is the product. The language is just the price of admission to being able to prove anything.**

## What's in this repository

| File | Contents |
|---|---|
| [`LANGUAGE-SPEC.md`](LANGUAGE-SPEC.md) | The language specification: type system, closed expression algebra, decision and composition semantics, formal verification semantics, 31 stable error codes, EBNF grammar, hard limits, DMN/Excel interchange, and a machine-verified worked example. |
| [`PRIOR-ART-SURVEY.md`](PRIOR-ART-SURVEY.md) | Survey of the field: DMN/FEEL, Rego/OPA, Cedar, CEL, Datalog, Drools, GoRules ZEN, commercial BRMS, the decision-table verification literature, and the LLM-generation evidence. Ends with a competitive assessment. |

## Design highlights

- **Domains are mandatory.** `income: Money` proves nothing; `income: Money range 0..10_000_000 scale 2 currency EUR` is a decidable domain. Totality is only *askable* when the domain is declared.
- **Order is not expressible.** There is no rule priority, salience, or first-match. Two rules that can both fire with different outcomes are a hard error.
- **`or` is banned at rule level.** Conditions are conjunctions of atoms, which keeps every rule a hyper-rectangle so gaps and conflicts can be reported as *minimal regions* rather than point counterexamples.
- **Escape hatches are visible.** Anything unverifiable must surface as `Inconclusive` in the verdict. It can never silently pass as `Verified`.
- **Rejection over defaulting.** An unhandled case is a compile error, not a runtime fallback. `Unspecified` is a first-class outcome so authors don't fabricate a default to silence the checker.
- **Diagnostics are region-minimal and three-valued** (`Verified` / `Violated` / `Inconclusive`), because they feed an automated repair loop. Redundant or point-level diagnoses make the loop worse, not better.

## Prior art, honestly

This idea is not novel and the survey says so plainly. Google shipped Z3-backed [formal verification for CEL](https://opensource.googleblog.com/2026/08/securing-the-agentic-era-introducing-formal-verification-for-cel.html) in August 2026 for exactly this reason; GoRules ZEN 2.0 ships exhaustiveness and write-conflict checking under an MIT licence; and Calvanese et al. published the core decision-table analysis algorithms in 2016 ([arXiv:1603.07466](https://arxiv.org/abs/1603.07466)).

The remaining wedges — region-minimal diagnostics, order-independence proofs that a first-match engine structurally cannot offer, and a zero-adoption on-ramp from Excel and non-executable DMN — are documented in the survey rather than assumed away.

## License

Not yet chosen.
