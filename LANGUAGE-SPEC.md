# Arbiter — Language Specification v0.1

> **Working name.** "Arbiter" is a placeholder; rename freely. Everything else in this document is normative.

**Status:** design draft, pre-implementation
**Date:** 2026-10-07
**Scope:** decision logic only. No process, no state, no side effects.

---

## 0. How to read this document

This spec is written to be handed to an implementer (human or AI). Normative requirements use **MUST** / **MUST NOT** / **SHOULD**. Every rejection rule has a stable error code; an implementation that emits different codes is non-conformant, because the error codes are part of the public interface consumed by the LLM repair loop.

Sections 1–4 define *why*. Sections 5–12 define *what*. Sections 13–16 define *how it is checked*. Sections 17–19 cover interchange, the implementation guide, and the summary of normative decisions.

---

## 1. Purpose

Arbiter is a declarative language for **business decision logic**: rules that map a set of typed facts to an outcome.

Its single design objective is that three properties of a program be **statically decidable**:

| Property | Meaning |
|---|---|
| **Totality** | Every input in the declared domain produces an outcome |
| **Boundedness** | Every evaluation terminates in bounded cost |
| **Determinism** | The same input always produces the same outcome, independent of rule order |

These are the product. Syntax is incidental.

### 1.1 What Arbiter is not

Arbiter is **not** a programming language and MUST NOT grow into one. The following are permanently out of scope:

- Process, workflow, state machines, time, scheduling
- Any side effect: I/O, network, database, logging, mutation
- `while` loops, recursion, unbounded iteration
- User-defined functions callable from conditions
- String manipulation inside conditions
- Nullable values
- Floating-point arithmetic

Anything requiring the above MUST be pushed across the boundary into a host language. That boundary is the integrity of the product and the first thing that will erode under user pressure.

---

## 2. Design principles

These principles are normative: they resolve ambiguities in this spec and constrain future changes.

1. **One canonical form per operation.** There MUST be exactly one way to express any given rule. No synonyms, no optional sugar, no alternative syntaxes.
2. **Domains are declared, never inferred.** A type without a declared domain is a type error.
3. **Composition, not precedence.** Rules compose; they never override one another.
4. **Order is not semantics.** No construct may make evaluation order observable.
5. **Reject, don't default.** An unhandled case is a compile error, not a runtime fallback.
6. **Escape hatches are visible.** Any construct that cannot be verified MUST appear in the verification verdict as `Inconclusive`. It MUST NOT silently pass as `Verified`.
7. **Smallest sufficient language.** A feature is admitted only if it passes the feature admission test (§4). Every feature is a liability.

---

## 3. The central mechanism: entities and the closed world

Totality is only *askable* if the input domain is known. Therefore:

> **Every value in Arbiter has a finite or decidable domain declared in the source.**

This is what `entity` declarations and mandatory domains exist to provide. They are not documentation; they are the precondition for every proof in §13.

Consequence: the verifier analyses a **closed world**. There is no "other data that might arrive" — if it isn't declared, it doesn't exist. Facts not present in the declared entity set are a type error at the boundary.

---

## 4. Feature admission test

Every proposed future feature MUST answer all five questions before admission:

1. **Which invariant does it cost** — totality, boundedness, or determinism? If none, is it genuinely free?
2. **Is the property still decidable** in its presence?
3. **Does it have an SMT theory** that Z3 can reason about?
4. **Can an escape hatch hide inside it** — i.e. can it make something unverifiable without appearing in the verdict?
5. **What is the rejection rule** when it is misused, and what is the error code?

A feature that cannot answer all five MUST NOT be admitted.

---

## 5. Lexical structure

- Encoding: UTF-8.
- Identifiers: `[A-Za-z_][A-Za-z0-9_]*`. Case-sensitive.
- Type and entity and enum names: `UpperCamelCase`. Field, parameter, decision names: `lower_snake_case`. Additional enum members: `UpperCamelCase`.
- Comments: `//` to end of line, and `/* ... */` block. Comments are not preserved in the AST except for provenance doc-comments.
- Numeric literals: decimal integers (`1000`, `1_000_000`) and fixed-point decimals (`19.99`). **Underscores are permitted as digit separators.** No exponent notation. No floating-point literals.
- String literals: double-quoted (`"..."`), UTF-8. The only escapes are `\"`, `\\`, `\n` and `\t`. There is no interpolation, concatenation, or escaping of arbitrary Unicode.
- Positions that take `INT` (`scale`, `max_len`, `max`) hold an unsigned integer literal with no separators; `range` bounds hold a signed `number`.
- Duration literals: a `number` followed by one of `days`, `months`, `years` (`3 days`, `12 months`, `2 years`). A `Duration` field declares its unit with `unit` (§6.1, §15).
- Keywords (reserved): `entity`, `enum`, `decision`, `when`, `otherwise`, `let`, `test`, `given`, `expect`, `source`, `owner`, `version`, `allow`, `gaps`, `range`, `scale`, `currency`, `unit`, `days`, `months`, `years`, `max`, `max_len`, `values`, `default`, `in`, `and`, `or`, `not`, `implies`, `true`, `false`, `Bool`, `Int`, `Decimal`, `Money`, `Date`, `DateTime`, `Duration`, `String`, `List`, `Set`, `abs`, `min`, `count`, `all`, `any`, `none`, `sum`, `avg`, `filter`, `map`.

Whitespace and newlines are not semantically significant except as statement terminators within a block.

---

## 6. Type system

### 6.1 Primitive types

| Type | Domain requirement | Notes |
|---|---|---|
| `Bool` | none (domain is `{true,false}`) | |
| `Int` | **MUST** declare `range a..b` | Bounded. Unbounded `Int` is `E-DOMAIN-UNDECLARED`. |
| `Decimal` | **MUST** declare `range a..b` and `scale s` | Fixed-point. `scale` = digits after the point. |
| `Money` | **MUST** declare `range`, `scale`, and `currency` | `Decimal` with a currency tag. No FX conversion is defined in v0.1. |
| `Date` | **MUST** declare `range a..b` | ISO-8601. |
| `DateTime` | **MUST** declare `range a..b` | UTC only. No timezone arithmetic. |
| `Duration` | **MUST** declare `range a..b` and a unit | |
| `String` | **MUST** be constrained (see 6.4) | |

**Floating point is excluded.** IEEE-754 is non-associative and platform-dependent, which breaks determinism, and is intractable for SMT. All fractional arithmetic is fixed-point `Decimal`, which is exact.

**Nullable values are excluded.** There is no `null`, no `Option`, no absent-vs-null distinction. Absence MUST be modelled explicitly as an enum member or a sentinel value with defined meaning.

### 6.2 Enumerations

```
enum Region { EU, UK, US, APAC }
```

An enum declares an exactly-known finite domain. Enums are the **preferred** type for any field that is branched upon.

### 6.3 Entities

```
entity Applicant {
  income:     Money   range 0..10_000_000 scale 2 currency EUR
  dependents: Int     range 0..20
  region:     Region
  history:    List[Loan, max 50]
}
```

Rules:

- An entity's field set is **fixed and closed**. There is no open/extra field mechanism.
- Fields are **required**. There is no optional field.
- A reference to an undeclared type, entity, field, or decision is `E-UNDECLARED`.
- Fields MUST be totally ordered in declaration for canonical serialization.
- Entities MUST NOT be recursive, directly or transitively (`E-RECURSION`).
- An entity MUST NOT have more than `MAX_FIELDS` fields (§12).

### 6.4 Strings

A `String` field MUST be one of:

- Backed by an `enum` (preferred — use the enum type directly), or
- Constrained by `values { "a", "b", "c" }` (a finite, explicit set), or
- Constrained by `max_len N` **and not used in any condition** except equality against a literal.

**A `String` that is neither enum-backed, value-constrained, nor excluded from conditions is rejected with `E-STRING-UNBOUNDED`.** Rationale: an unbounded string domain makes totality unprovable, because a rule set cannot be shown to cover all strings.

### 6.5 Collections

```
history: List[Loan, max 50]
tags:    Set[String, max 10]
```

- **Every collection MUST declare `max` cardinality**, at most `MAX_CARDINALITY` (§12). Unbounded is `E-COLLECTION-UNBOUNDED`.
- Nesting depth of collections MUST NOT exceed `MAX_COLLECTION_DEPTH` (§12).
- Collection element types MUST themselves be bounded types.

**Permitted operations** (all bounded, all decidable). Every operation except cardinality and membership uses the **uniform binder form** `op(x in collection : expr)` — there is exactly one syntax for "apply this to every element":

| Operation | Result type | Domain computation |
|---|---|---|
| `c.count()` | `Int range 0..max` | from cardinality |
| `count(x in c : pred)` | `Int range 0..max` | from cardinality |
| `sum(x in c : expr)` | numeric | `min(0, max*min_expr) .. max(0, max*max_expr)` |
| `min(x in c : expr)` | type of `expr` | range of `expr` |
| `max(x in c : expr)` | type of `expr` | range of `expr` |
| `avg(x in c : expr)` | `Decimal` | range of `expr`; **not closed at the element `scale`** (see note below) |
| `all(x in c : pred)` | `Bool` | — |
| `any(x in c : pred)` | `Bool` | — |
| `none(x in c : pred)` | `Bool` | — |
| `filter(x in c : pred)` | `List[T, max]` | cardinality bounded by source |
| `map(x in c : expr)` | `List[U, max]` | cardinality bounded by source |
| `e in c` | `Bool` | — |

The binder `x` is a fresh name scoped to the operation. It MUST NOT shadow an enclosing binder or a declared name (`E-SHADOW`). Quantifiers MUST NOT be nested deeper than `MAX_COLLECTION_DEPTH` (§12), else `E-COLLECTION-DEPTH`.

**A `sum` over an empty collection is `0`; `min`, `max` and `avg` over an empty collection are `E-EMPTY-AGGREGATE` unless the operation supplies an explicit default** via `min(x in c : expr, default d)`. This is a totality requirement: `min` of nothing has no defensible value, and silently returning a sentinel is exactly the kind of hidden default the language rejects.

For `sum`, let `max` be the declared cardinality and `min_expr`/`max_expr` the bounds of `expr`'s computed range; the result range is `min(0, max*min_expr) .. max(0, max*max_expr)` (the `min(0, …)`/`max(0, …)` terms are required because the collection may be empty, and because a negative `min_expr` makes `max*min_expr` the true lower bound). An explicit `default` widens the result range of `min`/`max` to include the default's range.

> **UNRESOLVED:** `avg` is not closed at the element `scale`. The average of `1.00` and `1.01` is `1.005`, which is not representable at scale 2, so "`scale` of element" does not define a result. Before `avg` can be implemented exactly, the spec must fix either an explicit result `scale` argument or a deterministic rounding rule (and the rounding rule for `a / b` in §7.2, which has the same problem).

**Forbidden:** indexing by a computed index, `first`/`last`/`head` on a possibly-empty collection without a defined total result, and any recursion over a collection.

Note carefully: `filter`, `map`, `count(pred)` and `all`/`any`/`none` are **permitted**, because with a declared maximum cardinality each expands to a finite formula. The prohibition is on *unbounded* iteration and recursion, not on bounded quantification. Aggregate result ranges MUST be computed from element ranges and max cardinality so that overflow is statically checkable.

### 6.6 Derived values and `let`

```
decision ratio_check(a: Applicant) -> Decision
  let delinquencies = count(x in a.history : x = Delinquent)
  let burden        = a.dependents / max(1, delinquencies)

  when delinquencies = 0 and burden <= 2 -> Approve
  ...
```

- `let` binds a name to an expression within a decision body. Bindings are immutable, single-assignment, and MUST be unique within scope.
- A `let` MUST NOT be redefined or shadowed (`E-SHADOW`).
- `let` expressions MUST type-check and MUST NOT overflow their computed range.

### 6.7 Named outcomes

```
enum Decision { Approve, Refer, Decline, Unspecified }
```

An outcome type MUST be an `enum`. Outcomes are values, not strings, and not free-form.

`Unspecified` is not special to the language, but is **strongly recommended** as an explicit member of every outcome enum, to represent "no policy defined for this input." It is the honest alternative to a fabricated default (§10.4).

---

## 7. Expression language

The expression language is a **fixed, closed algebra**. It is not extensible.

### 7.1 Operators

| Category | Operators |
|---|---|
| Boolean | `and`, `or`, `not`, `implies` |
| Comparison | `=`, `!=`, `<`, `<=`, `>`, `>=` |
| Arithmetic | `+`, `-`, `*`, `/`, `%`, `abs`, `min`, `max` |
| Membership | `in` (enum sets, collections) |
| Quantification | `all`, `any`, `none`, `count`, `sum`, `avg`, `filter`, `map` |

**No other functions exist.** In particular there are no string functions, no date functions beyond comparison and difference, no user-defined functions, and no host-language escapes. A call to an unknown function is `E-UNKNOWN-FN`.

Rationale: every operator must have an SMT theory. An uninterpreted function cannot be reasoned about, and admitting one makes the surrounding decision unverifiable without appearing in the verdict.

> **UNRESOLVED:** the algebra lists the operators but does not type them. It does not say how `Int`, `Decimal` and `Money` mix in `+ - * / %` and comparisons; how a numeric literal takes the type of its operand; whether arithmetic between different `currency` tags is rejected; what `a / b` returns (truncating integer division, or an exact `Decimal`, and at what `scale`); what the sign of `a % b` is; what `Money` divided by `Int` produces; or what `Date − Date`, `DateTime − DateTime`, and `Date + Duration` produce (the "difference" mentioned above has no rule). The checker of §18.1 step 3 cannot be implemented until these are specified.

### 7.2 Totality requirements

**Division.** `a / b` requires that `b != 0` be statically provable over the reachable domain. If it cannot be proven, the expression is rejected with `E-DIV-NONZERO`. The canonical fix is a guard:

```
a.income / max(1, a.dependents)     // total: divisor provably >= 1
```

**Modulo.** `a % b` has the same requirement as division.

**Overflow.** Every arithmetic expression MUST have a computed result range contained within the declared range of its type. Violation is `E-OVERFLOW`, reported with the offending expression and the computed range. Overflow MUST be a static error; it MUST NOT be a runtime check.

**Signedness and `abs`.** `abs` on a signed range `-a..b` yields range `0..max(a,b)`.

### 7.3 Linear vs non-linear

`*`, `/` or `%` with **two variable operands** is classified **non-linear**. This is the primary practical threat to verification: non-linear integer arithmetic is undecidable over unbounded integers, and Z3 will frequently return `unknown` or fail to terminate. (Every operand domain here is finite and declared, so the formula is decidable by bounded enumeration in principle; the concern is cost, which is why bounded model checking with `Inconclusive` on budget exhaustion is the fallback.)

Therefore:

- Non-linear expressions are **permitted**, but MUST be recorded in the AST.
- If a non-linear term appears in a decision, the verifier MUST attempt bounded model checking, and MUST return `Inconclusive` rather than guessing if it cannot decide.
- Implementations SHOULD emit `W-NONLINEAR` naming each non-linear term so authors can restructure toward linear forms.
- **Language designers and reviewers SHOULD treat keeping the arithmetic linear as a design goal**, not an afterthought.

Multiplication by a literal constant is linear and is always preferred. `price * 3` is fine; `price * quantity` is the hard case.

### 7.4 Expression depth

Expression nesting depth MUST NOT exceed `MAX_EXPR_DEPTH` (§12), else `E-EXPR-DEPTH`.

---

## 8. Decisions

A `decision` is the only executable construct. There is **no separate `pipeline` construct** — a composition of decisions is itself a decision (§9).

### 8.1 Syntax

```
decision eligibility(a: Applicant) -> Decision
  source  "Lending Policy 4.2(a), rev 2024-11"
  owner   "credit-risk@example.com"
  version "2026-10-07.1"

  when a.income >  40000 and a.dependents <= 3       -> Approve
  when a.income >  40000 and a.dependents >  3       -> Refer
  when a.income <= 40000 and a.region in {EU, UK}    -> Refer
  when a.income <= 40000 and a.region in {US, APAC}  -> Decline
```

### 8.2 Rules

- A decision MUST contain at least one rule (`E-NO-RULES`).
- A rule is `when <condition> -> <outcome>`.
- **A condition MUST be a conjunction of atoms.** `or` and `implies` MUST NOT occur in a rule condition outside a quantified predicate — including inside a parenthesised atom, which the grammar admits. Disjunction is expressed as multiple rules, which is the canonical form and matches the decision-table model. Violation is `E-OR-AT-RULE-LEVEL`.
  - An *atom* is: a comparison, a membership test, a quantification (`all`/`any`/`none`/`count`/`sum`/…), or a boolean-valued derived value. `not` and `!=` are permitted but yield non-convex regions (§13.5).
  - Atoms MUST NOT exceed `MAX_ATOMS_PER_RULE` (§12), else `E-RULE-TOO-COMPLEX`.
- A decision MUST NOT contain two rules with identical conditions and identical outcomes (`E-DUPLICATE-RULE`).
- A decision MUST NOT have more than `MAX_RULES` rules (§12; the error code for this limit is unresolved, see §12).
- The outcome of every rule MUST be a member of the declared outcome enum (`E-TYPE-MISMATCH`).

> **UNRESOLVED:** rules have no names. Diagnostics identify them by declaration index (§16.4 reports `"rule 1"`, `"rule 4"`). This is workable, but it conflicts with the prior-art survey's recommendation that "every intermediate and rule [be] explicitly named" — a principle carried over from the Anka benchmark, where losing track of a named artifact accounted for 42% of the observed generation errors. Either rules gain an optional label, or that survey row should be read as applying to `let` bindings and decisions only. Not resolved here because it requires a grammar change.

### 8.3 Completeness

Decisions are **complete by default**. Every input in the declared domain MUST match at least one rule.

- A gap — a member of the domain matched by no rule — is **`E-GAP`**, a hard error.
- `allow gaps` MAY be declared in the metadata block. If present, gaps are reported as `W-GAP` and carry a region diagnostic, but do not fail the build.
- `allow gaps` MUST appear in the verification verdict, and a decision that declares it MUST carry a `W-ALLOW-GAPS` finding.

### 8.4 `otherwise`

`otherwise -> <outcome>` is an optional final clause that matches every input not matched by any preceding rule.

- It makes totality automatic.
- **Because it makes totality automatic, it is a potential hiding place for missing policy.** Therefore:
  - Its coverage MUST be computed as a percentage of the declared domain.
  - The verifier MUST emit `W-OTHERWISE-COVERAGE` whenever coverage exceeds the configured threshold (default 10%).
  - The clause MUST appear in the verdict.

This is the concrete implementation of Principle 6. Without it, `otherwise -> Decline` becomes a silent way to make a broken rule set pass.

### 8.5 Conflicts

Two rules **conflict** if their conditions can both be true for some input *and* their outcomes differ at that input. An outcome may be a derived value (§9), so "differ" means the two outcome expressions evaluate to different enum members for some input in the overlap.

- A conflict is **`E-CONFLICT`**, a hard error. There is no hit policy, no priority, no first-match, and no way to suppress it.
- Two rules that overlap and produce the **same** outcome are **not** a conflict. They are redundant, and produce `W-REDUNDANT` with the overlap region.

### 8.6 Order independence

Rule order MUST NOT affect the result. This is not configurable: there is no syntax to express rule priority, salience, or table order. An implementation MUST NOT introduce order-dependence through evaluation strategy.

Consequently the verifier can prove **order-independence** as a derived property, and SHOULD report it explicitly in the verdict.

### 8.7 Dead and unreachable rules

- A rule whose condition is unsatisfiable given the declared domains is **`E-DEAD-RULE`**.
- A rule whose condition is fully covered by **other** rules with the same outcome is **`W-SUBSUMED`**; the test is order-independent (§8.6), so "other", not "earlier". A partial same-outcome overlap is `W-REDUNDANT` (§8.5), not `W-SUBSUMED`.
- An outcome enum member that no rule can produce is **`W-UNREACHABLE-OUTCOME`**.

---

## 9. Composition

A decision MAY reference another decision in its `let` bindings or rule atoms.

```
enum Price { Standard, Premium, Substandard }

decision region_tier(r: Region) -> Price
  when r in {EU, UK}   -> Standard
  when r in {US, APAC} -> Premium

decision pricing(a: Applicant, d: Decision) -> Price
  let tier = region_tier(a.region)

  when d = Approve and a.income >  100000 -> Premium
  when d = Approve and a.income <= 100000 -> tier
  when d = Refer                          -> Standard
  when d in {Decline, Unspecified}        -> Substandard
```

Here `pricing` references `region_tier`, and the outcome of a rule may be a derived enum value (`-> tier`) as well as a literal member.

**Disambiguation rule for implementers.** A decision reference is syntactically identical to a function call. Because the algebra is closed (§7.1) and there are no user-defined functions, resolution is unambiguous:

- If the name is a builtin — `min`, `max`, `abs`, `count`, `all`, `any`, `none`, `sum`, `avg`, `filter`, `map` (all reserved, §5) — it is a builtin.
- Otherwise, if the name resolves to a declared `decision`, it is a decision reference, and its argument types MUST match the declaration (`E-TYPE-MISMATCH`).
- Otherwise it is `E-UNKNOWN-FN`.

There MUST NOT be a user-visible distinction in syntax between the two cases, and a decision MUST NOT be shadowed by a binding of the same name (`E-SHADOW`).

Requirements:

1. **The decision call graph MUST be acyclic.** A cycle is `E-RECURSION`, detected by static graph analysis. This is what buys termination for the whole graph; it is cheap and non-negotiable.
2. **Referenced decisions MUST be total** (or `allow gaps`, which propagates).
3. **Wiring MUST be type-correct, and each argument MUST lie inside the callee's declared parameter domain.** Type-correctness alone is not enough: an `Int` argument of range `0..100` passed to a parameter declared `range 1..10` is well-typed but outside the domain over which the callee was proven total. Out-of-domain arguments are `E-TYPE-MISMATCH`.
4. **Composition preserves the callee's totality, not the caller's.** If every referenced decision is total over its declared parameter domains and the wiring is in-domain (3), then every decision-call atom is well-defined for every input and no reference can fail to produce an outcome. That does **not** make the composite total: the composite still has to satisfy §13.1 over its own domain. This is the payoff of mandatory typed I/O (§3) and the reason domains are not optional.
5. Graph depth MUST NOT exceed `MAX_DECISION_DEPTH` (§12), else `E-GRAPH-DEPTH`.
6. A decision MUST NOT have more than `MAX_PARAMS` parameters.

A "pipeline" in the sense of staged processing is simply a root decision that references others. Intermediates are `let` bindings. There is no second construct.

---

## 10. Tests and provenance

### 10.1 Tests

```
test t1:
  given Applicant { income = 50000, dependents = 2, region = EU, history = [] }
  expect Approve
```

**Tests pin intent; they do not prove properties.** A passing test suite MUST NOT be presented as evidence of totality or conflict-freedom, and a conformant implementation MUST NOT report a decision as verified on the basis of tests. This distinction is the entire justification for the language (see §1 and the design rationale).

> **UNRESOLVED:** `test` is a top-level declaration with no reference to the decision under test (§15), so nothing links `t1` to `eligibility`. When two decisions accept an `Applicant`, an implementation cannot know which one to run. Tests need an explicit decision binding (or must nest in a decision body) before the construct is executable.

### 10.2 Generated tests

The verifier SHOULD generate, for each rule, a minimal witness input that triggers it, plus a boundary input just outside each atom. These are reported as diagnostics, not as authored tests.

### 10.3 Provenance

`source`, `owner`, and `version` are optional metadata on a decision, and SHOULD be present on any decision that implements a policy.

- `source` MUST name the clause the rules derive from.
- Provenance MUST be carried into the verification verdict so that a finding can be attributed to a policy clause and an owner.

### 10.4 Why `Unspecified` exists

Forcing totality invites fabrication: an author satisfies the checker by adding `otherwise -> Decline`, which hides an unhandled case behind what looks like a deliberate business rule. Declaring `Unspecified` as an explicit outcome makes the unknown *visible, countable, and reviewable*. Prefer an honest `Unspecified` to a dishonest default.

---

## 11. Diagnostics

### 11.1 Verdict values

Verification returns exactly one of:

| Verdict | Meaning |
|---|---|
| `Verified` | The property was proven for all inputs in the domain |
| `Violated` | A counterexample exists and is reported |
| `Inconclusive` | The property could not be decided within limits |

`Inconclusive` MUST be used instead of guessing. **A reported `Violated` MUST be a real, reproducible counterexample.** False positives are the primary failure mode of verification tooling: a checker that cries wolf is disabled permanently.

### 11.2 Diagnostic form

Every diagnostic MUST carry:

- A stable code (§14)
- Severity (`error` | `warning`)
- The minimal **region** witnessing the finding, as a normalized predicate over declared fields
- The rule(s) involved
- The decision and its provenance

Diagnostics MUST be **deduplicated** and **minimal**. A finding MUST NOT be reported more than once, and MUST NOT be reported as a superset of a smaller equivalent finding. Redundant, oversized diagnoses are actively harmful when fed to an LLM repair loop, because the model will make a cosmetic change and consume an iteration.

### 11.3 Machine-readable output

Diagnostics MUST be emittable as JSON. This is the interface to the LLM repair loop and to CI.

```json
{
  "decision": "eligibility",
  "source": "Lending Policy 4.2(a), rev 2024-11",
  "verdict": "Violated",
  "findings": [
    {
      "code": "E-GAP",
      "severity": "error",
      "region": "income in 40000..55000 and dependents in 4..20 and region = EU",
      "witness": { "income": 47000, "dependents": 5, "region": "EU", "history": [] },
      "suggested_rule": "when a.income >= 40000 and a.income <= 55000 and a.dependents >= 4 and a.region = EU -> Refer"
    }
  ]
}
```

Note the **minimal region**, not merely a point witness. A point counterexample tells the author one failing input; a region tells them the fix. Both SHOULD be emitted.

---

## 12. Hard limits

All limits are **static** and produce errors at compile time. **No limit may be enforced by a runtime timeout or a caught exception** — that is exactly the "unwanted runtime scenario" the language exists to prevent.

| Limit | Constant | Default |
|---|---|---|
| Fields per entity | `MAX_FIELDS` | 64 |
| Params per decision | `MAX_PARAMS` | 20 |
| Rules per decision | `MAX_RULES` | 10,000 |
| Atoms per rule condition | `MAX_ATOMS_PER_RULE` | 32 |
| Expression nesting depth | `MAX_EXPR_DEPTH` | 16 |
| Collection nesting depth | `MAX_COLLECTION_DEPTH` | 2 |
| Collection max cardinality | `MAX_CARDINALITY` | 1,000 |
| Decision graph depth | `MAX_DECISION_DEPTH` | 32 |
| Verification solver budget | `VERIFY_BUDGET` | 10 s per decision |

**Exceeding `VERIFY_BUDGET` yields `Inconclusive`, not an error.** It is a coverage gap in verification, and must be reported as such.

> **UNRESOLVED:** four limits have no error code: `MAX_FIELDS` (§6.3), `MAX_PARAMS` (§9), `MAX_RULES` (§12, and §8 never actually cites it), and `MAX_CARDINALITY` (§6.5). §0 makes error codes part of the public interface and this section says every limit produces a compile-time error, so each needs a stable code (one generic code, or one per limit) before an implementation is conformant. The names are a design decision, so none is invented here.

---

## 13. Verification semantics

For decision `d` with declared input domain `D` and rule set `R`:

### 13.1 Totality

```
∀ x ∈ D : ∃ r ∈ R : cond_r(x)
```

A violation is a **gap**. It MUST be computed by differencing `D` against the union of all rule regions, and reported as one finding per maximal disjoint hyper-rectangle of the remainder: a non-rectangular gap is several findings, never one bounding box over them (§11.2 forbids reporting a superset of smaller findings). When the decision has an `otherwise` clause, treat it as a final catch-all rule for this formula (§8.4).

### 13.2 Determinism (conflict-freedom)

```
∀ x ∈ D, ∀ r₁,r₂ ∈ R, r₁ ≠ r₂ : (cond_r₁(x) ∧ cond_r₂(x)) ⇒ (outcome_r₁(x) = outcome_r₂(x))
```

`outcome_r(x)` is rule `r`'s outcome expression evaluated at `x`: constant when the outcome is a literal member, and `x`-dependent when it is a derived value (§9). An `otherwise` clause is excluded from this formula: by definition it fires only when no other rule does, so it cannot conflict.

A violation is a **conflict**. The witness reported MUST be the intersection region and the two conflicting rules.

### 13.3 Boundedness

Established **structurally**, not analytically: all cardinalities declared, no recursion, depths within limits. This is why boundedness is cheap and why the collection rules in §6.5 matter.

### 13.4 Acyclicity

The decision call graph MUST admit a topological ordering. Detection is standard cycle detection; it guarantees termination of the whole graph.

### 13.5 Implementation strategy

Two complementary engines, in this order:

1. **Region algebra (primary).** A rule condition is a conjunction of atoms (§8.2), so its region is the intersection of its atoms' regions. Atoms of the form `x = c`, `x < c`, `x <= c`, `x > c`, `x >= c`, or `x in {…}` over a finite enum are iso-oriented hyper-rectangles (an enum membership set is a rectangle over the enum dimension) in N-dimensional space, and their intersection is a region. Atoms that are `x != c`, `not`, a quantified collection operator, collection membership (`e in c`), or a `let`-bound boolean whose definition contains disjunction, `min`/`max`/`abs`, or cross-field arithmetic produce **complements or unions of rectangles, not rectangles** — a single rule is therefore not always one hyper-rectangle. Totality is set-difference of the domain against the union of rule regions; conflict is pairwise intersection. The algebra MUST therefore represent orthogonal regions (unions of rectangles), or defer the atom to Z3. This produces **region-minimal diagnostics** and is fast and dependency-free on the rectangular fragment. This is the spine of the implementation.
2. **SMT via Z3 (backstop).** Used for atoms that are not hyper-rectangles — `!=`/`not`, cross-field arithmetic (`loan > 3 * income`), quantification over collections, and any case where an `unsat` *proof* is wanted rather than a search result. `z3-solver` ships as a prebuilt wheel and has no build step.

Where region algebra cannot express a condition cleanly, evaluation MUST fall back to Z3, and if Z3 cannot decide within `VERIFY_BUDGET`, the verdict is `Inconclusive`.

**Reference prior art:** Calvanese, Dumas, Laurson, Maggi, Montali, Teinemaa, *"Semantics and Analysis of DMN Decision Tables"*, BPM 2016 ([arXiv:1603.07466](https://arxiv.org/abs/1603.07466)) — formal semantics, the hyper-rectangle interpretation, and sweep-based algorithms for exactly these two tasks. Build on it; do not re-derive it.

**Do not rely on SMT alone.** A general solver returns a satisfiability verdict and a point witness. It does not natively produce the *region* to add or the *pair* of rules to fix. Region algebra is what makes diagnostics actionable.

---

## 14. Rejection rules (error catalog)

| Code | Severity | Trigger |
|---|---|---|
| `E-PARSE` | error | Syntax error |
| `E-TYPE-MISMATCH` | error | Operand or outcome type incorrect |
| `E-UNDECLARED` | error | Reference to an undefined type, entity, field, or decision |
| `E-DOMAIN-UNDECLARED` | error | Missing domain: `Int`/`Date`/`DateTime`/`Duration` without `range`; `Decimal`/`Money` without `range` or `scale`; `Money` without `currency`; `Duration` without `unit` |
| `E-STRING-UNBOUNDED` | error | `String` used in a condition without enum or value constraint |
| `E-COLLECTION-UNBOUNDED` | error | Collection without `max` |
| `E-COLLECTION-DEPTH` | error | Collection or quantifier nesting exceeds `MAX_COLLECTION_DEPTH` |
| `E-EMPTY-AGGREGATE` | error | `min`/`max`/`avg` over a possibly-empty collection without an explicit `default` |
| `E-RECURSION` | error | Cycle in entity or decision graph |
| `E-GRAPH-DEPTH` | error | Decision graph deeper than `MAX_DECISION_DEPTH` |
| `E-EXPR-DEPTH` | error | Expression deeper than `MAX_EXPR_DEPTH` |
| `E-RULE-TOO-COMPLEX` | error | More than `MAX_ATOMS_PER_RULE` atoms |
| `E-OR-AT-RULE-LEVEL` | error | `or` or `implies` in a rule condition outside a quantified predicate (including inside parentheses) |
| `E-DIV-NONZERO` | error | Divisor not provably non-zero |
| `E-OVERFLOW` | error | Computed range exceeds declared range |
| `E-SHADOW` | error | Duplicate or shadowed binding; binder or `let` shadowing a declared name; a decision shadowed by a binding |
| `E-UNKNOWN-FN` | error | Call to a function outside the closed algebra |
| `E-DUPLICATE-RULE` | error | Two rules with identical condition and outcome |
| `E-DEAD-RULE` | error | Rule condition unsatisfiable over the domain |
| `E-GAP` | error | Domain member matched by no rule, without `allow gaps` |
| `E-CONFLICT` | error | Overlapping rules with differing outcomes |
| `E-NO-RULES` | error | Decision with no rules |
| `W-GAP` | warning | Gap present under `allow gaps` |
| `W-REDUNDANT` | warning | Overlapping rules with the same outcome |
| `W-SUBSUMED` | warning | Rule fully covered by other rules with the same outcome |
| `W-UNREACHABLE-OUTCOME` | warning | Outcome member no rule can produce |
| `W-OTHERWISE-COVERAGE` | warning | `otherwise` covers more than the threshold |
| `W-NONLINEAR` | warning | Non-linear arithmetic term present |
| `W-ALLOW-GAPS` | warning | Decision declared `allow gaps` |
| `W-ORDER-DEPENDENT` | warning | Imported from a DMN table whose hit policy is order-dependent (`FIRST`, `RULE ORDER`) |
| `W-UNSUPPORTED-HIT-POLICY` | warning | Imported from a DMN table whose hit policy has no native equivalent (`COLLECT`) |

---

## 15. Grammar (EBNF)

```ebnf
program        ::= declaration*

declaration    ::= enum_decl | entity_decl | decision_decl | test_decl

enum_decl      ::= "enum" IDENT "{" IDENT ("," IDENT)* "}"

entity_decl    ::= "entity" IDENT "{" field+ "}"
field          ::= IDENT ":" type field_constraint*
field_constraint ::= "range" number ".." number
                   | "scale" INT
                   | "currency" IDENT
                   | "unit" UNIT
                   | "values" "{" literal ("," literal)* "}"
                   | "max_len" INT

type           ::= "Bool" | "Int" | "Decimal" | "Money"
                 | "Date" | "DateTime" | "Duration" | "String"
                 | IDENT                                     (* enum or entity *)
                 | "List" "[" type "," "max" INT "]"
                 | "Set"  "[" type "," "max" INT "]"

decision_decl  ::= "decision" IDENT "(" params? ")" "->" IDENT
                   metadata* body
params         ::= param ("," param)*
param          ::= IDENT ":" type
metadata       ::= "source" STRING | "owner" STRING | "version" STRING | "allow" "gaps"
body           ::= let_binding* rule* otherwise?             (* "at least one rule" is a semantic check, §8.2 *)
let_binding    ::= "let" IDENT "=" expr
rule           ::= "when" condition "->" expr
otherwise      ::= "otherwise" "->" expr

condition      ::= atom ("and" atom)*                        (* no "or"/"implies" at this level *)
atom           ::= cmp_expr "in" "{" literal ("," literal)* "}"   (* enum/set membership *)
                 | cmp_expr "in" cmp_expr                    (* collection membership *)
                 | not_expr                                  (* comparison, negation, quantification, or boolean-valued name *)
collection_op  ::= quantifier "(" IDENT "in" expr ":" expr ("," "default" expr)? ")"
quantifier     ::= "all" | "any" | "none" | "count"
                 | "sum" | "min" | "max" | "avg" | "filter" | "map"

expr           ::= implies_expr
implies_expr   ::= or_expr ("implies" implies_expr)?         (* right-associative *)
or_expr        ::= and_expr ("or" and_expr)*
and_expr       ::= not_expr ("and" not_expr)*
not_expr       ::= "not" not_expr | cmp_expr
cmp_expr       ::= add_expr (comp_op add_expr)?
comp_op        ::= "=" | "!=" | "<" | "<=" | ">" | ">="
add_expr       ::= mul_expr (("+" | "-") mul_expr)*
mul_expr       ::= unary (("*" | "/" | "%") unary)*
unary          ::= "-" unary | postfix
postfix        ::= primary ("." method ("(" args? ")")?)*
method         ::= IDENT | "count"                          (* reserved builtins are legal only here; §6.5 `c.count()` *)
primary        ::= literal
                 | "(" expr ")"
                 | IDENT "(" args? ")"                       (* builtin or decision call; resolved by §9 *)
                 | collection_op
args           ::= expr ("," expr)*

test_decl      ::= "test" IDENT ":" "given" IDENT "{" assignments "}" "expect" IDENT
assignments    ::= assignment ("," assignment)*
assignment     ::= IDENT "=" literal

literal        ::= number | STRING | "true" | "false" | IDENT
                 | duration
                 | "[" (literal ("," literal)*)? "]"          (* list literal, e.g. history = [] *)
duration       ::= number UNIT
number         ::= ["-"] digit (digit | "_")* ["." digit+]
UNIT           ::= "days" | "months" | "years"
```

Terminals: `IDENT` and `digit` are the lexical classes of §5; `INT` is an unsigned integer literal; `STRING` is a double-quoted literal; `UNIT` is one of `days`, `months`, `years`; `..` is the range token. A reserved builtin name is legal as a `method` (after `.`) and nowhere else, so `c.count()` parses while `count` may not be used as a declared name. There is no separate production for `min`/`max`/`abs`: they are `IDENT "(" args? ")"` calls resolved as builtins by name (§9). Two places need one token of lookahead rather than LL(1): a `quantifier "("` call is a `collection_op` when the token after the binder is `in`, and a `number` is a `duration` when a `UNIT` follows it.

`or` (and `implies`) is legal inside `expr` (for `let` bindings and quantified predicates) but **not** in a rule condition: the checker rejects any `or`/`implies` node in a `condition` outside a `collection_op` predicate with `E-OR-AT-RULE-LEVEL`. The grammar cannot enforce this by itself, because a parenthesised `expr` is reachable from an `atom`. Note also that "conjunction of atoms" does **not** by itself make every rule a hyper-rectangle: `!=`, `not`, `in {…}` over an ordered type, collection membership and quantification, and `let`-bound booleans containing disjunction, `min`/`max`/`abs`, or cross-field arithmetic yield non-rectangular regions (§13.5).

---

## 16. Worked example, with diagnostics

### 16.1 Source

```
enum Region   { EU, UK, US, APAC }
enum Loan     { Current, Delinquent, WrittenOff }
enum Decision { Approve, Refer, Decline, Unspecified }

entity Applicant {
  income:     Money  range 0..10_000_000 scale 2 currency EUR
  dependents: Int    range 0..20
  region:     Region
  history:    List[Loan, max 50]
}

decision eligibility(a: Applicant) -> Decision
  source  "Lending Policy 4.2, rev 2024-11"
  owner   "credit-risk@example.com"
  version "2026-10-07.1"

  let delinquencies = count(x in a.history : x = Delinquent)

  when a.income >  40000 and a.dependents <= 3 and delinquencies = 0  -> Approve
  when a.income >  40000 and a.dependents <= 3 and delinquencies > 0  -> Refer
  when a.income >  40000 and a.dependents >  3                        -> Refer
  when a.income <= 40000 and a.region in {EU, UK}                     -> Refer
  when a.income <= 40000 and a.region in {US, APAC}                   -> Decline

test t1:
  given Applicant { income = 50000, dependents = 2, region = EU, history = [] }
  expect Approve
```

### 16.2 Verification output — clean

This rule set is total and conflict-free. Verify by hand:

- `income > 40000 ∧ dependents <= 3` → rules 1 and 2, which partition on `delinquencies` (nothing is both `= 0` and `> 0`).
- `income > 40000 ∧ dependents > 3` → rule 3.
- `income <= 40000` → rules 4 and 5, which partition `Region` completely.

The union is the entire declared domain, and no two rules with differing outcomes can both fire. A conformant implementation therefore reports:

```json
{
  "decision": "eligibility",
  "source": "Lending Policy 4.2, rev 2024-11",
  "owner": "credit-risk@example.com",
  "verdict": "Verified",
  "properties": {
    "bounded":           "Verified",
    "acyclic":           "Verified",
    "order_independent": "Verified",
    "total":             "Verified",
    "conflict_free":     "Verified"
  },
  "findings": [
    {
      "code": "W-UNREACHABLE-OUTCOME",
      "severity": "warning",
      "region": "(whole domain)",
      "note": "outcome 'Unspecified' is not produced by any rule"
    }
  ]
}
```

Note what is being demonstrated: **a clean rule set produces zero errors.** The single warning is genuine — `Unspecified` is declared but no rule yields it, which usually means the author intended to handle a case and forgot. Warnings do not fail the build.

### 16.3 A variant with a planted gap and conflict

```
decision eligibility_draft(a: Applicant) -> Decision
  source  "Lending Policy 4.2, rev 2024-11 (draft)"
  owner   "credit-risk@example.com"
  version "2026-10-07.draft"

  when a.income >  40000 and a.dependents <= 3    -> Approve
  when a.income >  40000 and a.dependents >  3    -> Refer
  when a.income <= 40000 and a.region in {EU, UK} -> Refer
  when a.income >  50000                          -> Decline
```

Two defects, both real:

- **Gap.** Rules 3 and 4 do not cover `income <= 40000 ∧ region ∈ {US, APAC}`. Rule 4 begins above 50000 and cannot help.
- **Conflict.** Rule 4 (`income > 50000`) overlaps rules 1 and 2, and its outcome `Decline` disagrees with both `Approve` and `Refer`.

### 16.4 Verification output — defects

```json
{
  "decision": "eligibility_draft",
  "source": "Lending Policy 4.2, rev 2024-11 (draft)",
  "owner": "credit-risk@example.com",
  "verdict": "Violated",
  "properties": {
    "bounded":           "Verified",
    "acyclic":           "Verified",
    "order_independent": "Verified",
    "total":             "Violated",
    "conflict_free":     "Violated"
  },
  "findings": [
    {
      "code": "E-CONFLICT",
      "severity": "error",
      "region": "income > 50000 and dependents <= 3",
      "witness": { "income": 60000, "dependents": 2, "region": "EU", "history": [] },
      "rules": ["rule 1", "rule 4"],
      "outcomes": ["Approve", "Decline"]
    },
    {
      "code": "E-CONFLICT",
      "severity": "error",
      "region": "income > 50000 and dependents > 3",
      "witness": { "income": 60000, "dependents": 5, "region": "EU", "history": [] },
      "rules": ["rule 2", "rule 4"],
      "outcomes": ["Refer", "Decline"]
    },
    {
      "code": "E-GAP",
      "severity": "error",
      "region": "income <= 40000 and region in {US, APAC}",
      "witness": { "income": 20000, "dependents": 1, "region": "US", "history": [] },
      "suggested_rule": "when a.income <= 40000 and a.region in {US, APAC} -> Unspecified"
    },
    {
      "code": "W-UNREACHABLE-OUTCOME",
      "severity": "warning",
      "region": "(whole domain)",
      "note": "outcome 'Unspecified' is not produced by any rule"
    }
  ]
}
```

Three things to notice, all of which are conformance requirements rather than presentation choices:

1. **Regions, not just points.** The conflict is reported as `income > 50000 and dependents <= 3` — the whole offending region — not merely the single witness `income = 60000`. A point tells the author one failing input; a region tells them the fix.
2. **No duplicate or superset findings.** Rule 4 conflicts with both rule 1 and rule 2, and these are reported as two distinct, disjoint regions rather than one vague "rule 4 overlaps the table" message. Prior art gets this wrong: Calvanese et al. document a commercial DMN editor that "often reports the same rule overlap multiple times."
3. **Independent property verdicts.** `bounded`, `acyclic` and `order_independent` stay `Verified` while the other two are `Violated`. Collapsing all five into a single pass/fail destroys the information a repair loop needs.

The `W-UNREACHABLE-OUTCOME` warning from §16.2 persists here (the same `Decision` enum still has no rule producing `Unspecified`); errors and warnings are independent findings. Note the suggested rule uses the parameter-qualified source syntax (`a.income`), not the bare field names of the region notation.

### 16.5 Repaired

```
decision eligibility_draft(a: Applicant) -> Decision
  source  "Lending Policy 4.2, rev 2024-11 (draft)"
  owner   "credit-risk@example.com"
  version "2026-10-07.draft2"

  let delinquencies = count(x in a.history : x = Delinquent)

  when a.income >  40000 and a.dependents <= 3 and delinquencies = 0 -> Approve
  when a.income >  40000 and a.dependents <= 3 and delinquencies > 0 -> Refer
  when a.income >  40000 and a.dependents >  3                       -> Refer
  when a.income <= 40000 and a.region in {EU, UK}                    -> Refer
  when a.income <= 40000 and a.region in {US, APAC}                  -> Unspecified
```

The gap is closed, and — importantly — it is closed with `Unspecified` rather than an invented `Decline`. `W-OTHERWISE-COVERAGE` does **not** fire, because no `otherwise` clause is used; the unmodelled case is stated explicitly and can be counted and reviewed. The warning does change: no rule now produces `Decline`, so the verdict carries `W-UNREACHABLE-OUTCOME` on `Decline` instead of on `Unspecified`. That is the trade §10.4 describes — the unmodelled case is visible, and so is the outcome the repair displaced.

---

## 17. Interchange

DMN import/export is a **distribution strategy, not a compromise**. It buys execution engines, modelers, and a benchmark corpus for free.

### 17.1 Import mapping

| DMN | Arbiter |
|---|---|
| `itemDefinition` | `enum` or `entity`; DMN carries no domains, so every imported numeric field is `E-DOMAIN-UNDECLARED` until the author declares `range`/`scale`/`currency`/`unit` (domains are never inferred, §2) |
| `decisionTable` | `decision` |
| Input column + type | parameter + `type` |
| Rule input entry | atoms of the `when` condition |
| Rule output entry | `->` outcome |
| `hitPolicy="UNIQUE"` | native semantics — conflicts are errors |
| `hitPolicy="ANY"` | native semantics — agreeing overlaps are `W-REDUNDANT` |
| `hitPolicy="FIRST"` / `"RULE ORDER"` | imported, flagged `W-ORDER-DEPENDENT`; MUST NOT become native semantics |
| `hitPolicy="COLLECT"` | imported as `allow gaps` plus a `W-UNSUPPORTED-HIT-POLICY`; requires manual review |
| Completeness indicator | `complete` (default) or `allow gaps` |
| (no equivalent) | `otherwise`, `Unspecified`, provenance |

**Level-1 DMN tables** — those with unstructured or natural-language expressions, which are not executable — MUST import as far as the structure allows, with unparseable input entries surfaced as `Inconclusive` regions rather than dropped. These tables have never been executed or tested and are a primary target (§1 purpose).

Also target: **Excel decision tables**, the actual incumbent. Import MUST NOT require an engine or a running service.

### 17.2 Export

Export to DMN MUST be supported so that any conformant engine can execute the result. Export is lossy in one direction: `otherwise` and provenance have no DMN equivalent and MUST be reported as dropped.

---

## 18. Implementation guide

### 18.1 Build order

1. **Region algebra core.** Interval and rectangle intersection and set-difference over typed atoms. No dependencies, pure logic. This is the spine; everything else is packaging.
2. **Parser + AST.** Recursive descent or a parser generator. AST nodes immutable, with source locations.
3. **Type and domain checker.** Compute ranges for every expression; enforce §6 and §7.
4. **Diagnostic engine.** Minimal, deduplicated, ranked regions; JSON output; three-valued verdicts.
5. **DMN import.** Validate against the DMCommunity corpus immediately.
6. **Z3 backstop.** Cross-field arithmetic and `unsat` proofs; bounded model checking; `Inconclusive` on budget exhaustion.
7. **Excel import.**
8. **LLM repair loop.** Prose → Arbiter → verify → feed minimal regions back → repair.

### 18.2 Module layout (suggested)

```
arbiter/
  lexer, parser, ast        # surface
  types/, domains/          # §6, range computation
  checker/                  # §7, §14 error catalog
  region/                   # interval sets, hyper-rectangles, difference
  verify/                   # §13: totality, conflicts, acyclicity
  smt/                      # Z3 backstop
  diagnostics/              # §11, JSON emission
  import/dmn/, import/xlsx/
  export/dmn/
```

### 18.3 Test strategy

Three levels, and they test different things:

1. **Unit tests** on region algebra — the highest-value tests in the codebase. Property-based: for random rule sets, `union(rules) ∪ gaps = domain` must hold.
2. **Corpus tests** — the [DMCommunity challenges](https://dmcommunity.org/challenges/) (87 published decision-modeling problems). Success is measured by whether reported regions are minimal and correct, not merely whether a violation was detected.
3. **Conformance tests** — planted defects (a gap, a conflict, a dead rule, a cycle, an overflow, a division by zero) MUST each produce exactly the expected error code, and clean inputs MUST produce zero findings. **False positives are a release blocker**, not a nuisance.

### 18.4 Explicit non-goals for v0.1

Do not build: an editor, a visual modeler, a runtime engine, a service, a workflow layer, or a host-language binding. The deliverable is a **checker** plus the interchange needed to point it at existing rule sets.

---

## 19. Summary of normative decisions

| Decision | Choice | Rationale |
|---|---|---|
| Execution model | Decision only | Analyzability (§1.1) |
| Domains | Mandatory, declared | Totality is only askable with known domains (§3) |
| Numbers | Fixed-point only, ranged | Exactness and SMT-representability (§6.1) |
| Nulls | None | Removes an entire class of edge cases (§6.1) |
| Strings in conditions | Enum or value-constrained only | Unbounded strings make totality unprovable (§6.4) |
| Collections | Bounded, quantifiable, non-recursive | Boundedness is structural (§6.5) |
| Functions | Closed algebra | Every operator needs an SMT theory (§7.1) |
| Arithmetic | Linear preferred, non-linear flagged | Non-linear int arithmetic is undecidable (§7.3) |
| Rules | Conjunction of atoms | Canonical form; clean region algebra (§8.2) |
| Order | Not expressible | Determinism by construction (§8.6) |
| Overlap | Always an error | No priority, no first-match (§8.5) |
| Totality | Mandatory; `allow gaps` opt-out | Reject rather than default (§8.3) |
| `otherwise` | Permitted, coverage reported | Escape hatches must be visible (§8.4) |
| Composition | Acyclic, typed, pure | Termination and free totality (§9) |
| Limits | Static, never runtime | No unwanted runtime scenarios (§12) |
| Diagnostics | Region-minimal, deduplicated, 3-valued | Usable by an LLM repair loop (§11) |
| Tests | Pin intent only | Tests sample; verification proves (§10.1) |
| Interchange | DMN and Excel, both directions | Distribution without adoption (§17) |
