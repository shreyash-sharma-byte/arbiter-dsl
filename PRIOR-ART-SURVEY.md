# Prior Art Survey — Business Decision DSL

**Date:** 2026-10-07
**Scope decision:** decision-only for v1 (no process/workflow semantics); workflow delegated to an engine later.
**Goal:** a DSL that an LLM compiles prose business specs into, with a verifier strong enough to catch what the LLM gets wrong.

---

## TL;DR

**The thesis is correct, and that is now the problem.** Three independent findings confirm it and simultaneously close the space:

1. **The DSL-for-LLM idea works empirically.** [Anka](#61-direct-evidence-that-constrained-dsls-beat-general-purpose-languages-for-llm-generation) (Dec 2025): a novel DSL with zero training exposure hit 99.9% parse success, and beat Python by **40 points** on multi-step tasks purely by removing syntactic choice points. ([paper](https://arxiv.org/abs/2512.23214))
2. **Formal verification of AI-authored rules is now a shipped product category.** Google's [CEL Formal Verification Framework](https://opensource.googleblog.com/2026/08/securing-the-agentic-era-introducing-formal-verification-for-cel.html) (Aug 2026) is Z3-powered, open source, and exists because "unit tests may fail to cover the infinite set of possible inputs" for AI-generated policies.
3. **Static analysis of decision tables is already built.** [GoRules ZEN 2.0](#42-gorules-zen--the-most-direct-competitor-and-already-most-of-this-idea) ships exhaustiveness checking, write-conflict detection, static type checking and decision-path coverage — MIT-licensed, Rust, six language bindings, with an AI copilot.

**So: do not build the general version.** The surviving wedges, in order of defensibility:

- **Region-minimal diagnostics** — not a point counterexample, not a redundant list (§5.2, §5.3).
- **Order-independence proofs** — GoRules keeps first-match semantics, so it structurally cannot offer this (§7.3).
- **Zero-adoption ingress** — Excel and non-executable Level-1 DMN, which no engine can even run (§4.6, §9.1).
- **Provenance and compliance traceability** tied to verification evidence (§7.3).

**And do this first, before designing anything:** run ZEN 2.0 against real tables with deliberately planted gaps and overlaps, and read its actual diagnostic output. One day of work decides whether the next six months are spent building a competitor or a complement. See [§9.0](#90-before-anything-else-the-one-day-experiment-that-settles-the-strategy).

---

## 0. The thesis this survey is testing

The bet is that **the verifier is the product, not the language**. If the DSL is only a nicer way to write rules, it loses to Excel, DMN, and a `dict` of lambdas. It only wins if it can prove things the alternatives cannot: *"these two rules can never both fire," "no rule covers this input space," "this outcome is unreachable," "rule 12 is shadowed by rule 4 and is dead code."*

That reframes the whole design question. Every feature must be judged by **does it preserve the ability to prove something**, not by whether it makes rules easier to read.

---

## 1. The one axis that decides everything: expressiveness vs analyzability

Every system in this survey sits somewhere on one tradeoff:

```
 MORE ANALYZABLE  ←──────────────────────────────→  MORE EXPRESSIVE
 DMN/S-FEEL   Cedar   Rego/Datalog   CEL   Drools   Python/JS
```

The rule that holds across all of them: **you can prove properties of a language only in the fragment you can decide.** Add loops, dynamic dispatch, mutable state, or unbounded recursion and you exit the analyzable regime permanently — not incrementally.

Concretely, what keeps a language analyzable:

| Restriction | What it buys you |
|---|---|
| No mutable state, no ordering effects | Rule interactions become a set problem, not a scheduling problem |
| Finite/decidable value domains (or SMT-representable types) | Satisfaction and coverage become decidable |
| No unbounded recursion | Termination is structural, not proven |
| Side-effect-free conditions | A rule can be evaluated in isolation, so pairwise analysis is sound |
| Total functions (every input resolves) | Gap detection is even *askable* |

Drools violates nearly all of these, which is precisely why it is powerful and why nobody can tell you what a 2,000-rule Drools base actually does.

---

## 2. Cluster A — Decision table standards (DMN / FEEL)

**What it is.** DMN (Decision Model and Notation) is the OMG standard for business decision logic. Its central construct is the decision table: typed input columns, rules as rows, output columns, plus a **hit policy** and a **completeness indicator**. Expressions use FEEL (Friendly Enough Expression Language); decision tables use the restricted **S-FEEL** subset.

**Current version.** DMN 1.7 is the latest specification — Trisotech's [Sept 2026 release notes](https://www.trisotech.com/digital-enterprise-suite-release-notes-september-18-2026/) report updating their engine "to support the latest DMN 1.7 specification," including DMN 1.7 XML import/export.

**Conformance levels** (verified against [KIE/Drools documentation](https://apache.googlesource.com/incubator-kie-docs/+/c6652f3bd85210e0da98dffe9bb71eedf1b22638/doc-content/drools-docs/src/main/asciidoc/DMN/dmn-conformance-levels-con.adoc), which summarises the OMG spec):

| Level | Adds | Executable? |
|---|---|---|
| **1** | Decision requirement diagrams, decision logic, decision tables. Any language may define expressions, including unstructured natural language. | **No** |
| **2** | **S-FEEL** expressions; fully executable decision models. | Yes |
| **3** | Full **FEEL**, the complete set of boxed expressions. | Yes |

Levels are cumulative. This matters: **Level 1 tables are not executable**, which means a large share of real-world DMN in the wild is documentation rather than running logic — and is therefore a natural target for static analysis, since it has never been executed or tested.

**Hit policies.** The most consequential design surface, documented precisely in the [Fluxnova (Camunda 7 CE fork) manual](https://github.com/finos/fluxnova-docs-manual/blob/main/content/reference/dmn/decision-table/hit-policy.md). Default is `UNIQUE`.

| Policy | Semantics | Order-dependent? | Analysis-friendly? |
|---|---|---|---|
| `UNIQUE` | At most one rule may match; **more than one is a violation** | No | Yes |
| `ANY` | Multiple may match but **must agree** on output | No | Yes |
| `FIRST` | First matching rule wins | **Yes** | No |
| `RULE ORDER` | All matching outputs, in table order | **Yes** | No |
| `COLLECT` (+ `SUM`/`MIN`/`MAX`/`COUNT`) | All matching outputs in arbitrary order, optionally aggregated | No | Partially |

Two things follow. First, `UNIQUE` and `ANY` are the only policies that encode a *checkable* property — and they are also order-independent, which is why §9.3 adopts them. Second, **engines only detect `UNIQUE`/`ANY` violations at runtime, on inputs that happen to occur.** A table can run for years with a latent conflict no request has triggered. This is the gap the whole project lives in.

**The formal analysis literature.** Covered in detail in §5. The short version: [Calvanese et al., "Semantics and Analysis of DMN Decision Tables"](https://arxiv.org/abs/1603.07466) (BPM 2016) provides the formal semantics and the two core algorithms (overlap detection, missing-rule detection) via a hyper-rectangle geometric interpretation, implemented atop the `dmn-js` editor and evaluated on a credit-lending dataset. Prior art they survey: **rough sets** (requires boolean/categorical domains, so numerics must be pre-discretized), **Prologa** (prevents overlaps/gaps by construction, supports rule merging, same discretization limit), **OpenRules** (constraint satisfaction, boolean output only), and **Signavio's editor** (detects overlaps and gaps, but its diagnoses are "unnecessarily large" and it "often reports the same rule overlap multiple times").

**Implementations and tooling.**

| Engine / tool | Notes |
|---|---|
| **Camunda 8** | Current Camunda product; DMN decision tables supported; deepening AI-agent integration via connectors. |
| **Fluxnova** (FINOS) | Community continuation of Camunda 7 CE, which is [**end-of-life**](https://github.com/camunda/camunda-bpm-platform). Same DMN engine and docs. |
| **Drools / Apache KIE** | DMN at conformance level 3; `org.kie` artifacts at 10.0.0; Apache-2.0. |
| **Trisotech** | Commercial suite; DMN 1.7 support; **AI Model Generators** producing DMN from documents. |
| **GoRules ZEN** | Accepts DMN among its formats; see §4 for why it is the most important entrant. |
| **`dmn-js` / `dmn-moddle`** (bpmn-io) | [Read and write DMN XML from JavaScript](https://github.com/bpmn-io/dmn-moddle) — the pragmatic import/export path. `dmn_python` exists on PyPI. |
| **jDMN** | Open-source Java DMN implementation. |

**What to steal.**
- The **decision table** as the unit of authoring — it is the format business people already recognise, and it makes structure explicit.
- **Typed columns with facets** — declared domains are what make region analysis possible.
- **`UNIQUE`/`ANY` semantics** as the native default; completeness as a mandate rather than an option.
- **DMN XML as an interchange format** — import/export buys engines, modelers, and the DMCommunity corpus for free (§9.2).

**What to avoid.**
- **Order-dependent hit policies** (`FIRST`, `RULE ORDER`) as anything but an import concession. They destroy analyzability.
- **XML as the authoring format.** Verbose, diff-hostile, and unpleasant for an LLM to generate compared to a compact modern syntax.
- **FEEL's quirks.** It is a large language with a long list of subtle behaviours (null handling in ranges, for-loops over null iterators, `list replace` parsing, etc. — Trisotech's bug-fix list is a decent sample).
- **The "business users will author it" fantasy.** They mostly have not; see §4 and §7.


---

## 3. Cluster B — Policy languages (Rego / Cedar / CEL / Datalog)

These are the languages that took **analyzability seriously as a design constraint** rather than discovering it later. They are the best source of ideas for what to deliberately give up.

### 3.1 Rego / Open Policy Agent — the cautionary tale about syntax

Rego is a declarative, Datalog-derived policy language under CNCF (graduated 2021, Apache-2.0). **OPA 1.0 shipped 20 December 2024** ([announcement](https://www.openpolicyagent.org/blog/announcing-opa-1-0-a-new-standard-for-policy-as-code-a6d8427ee828)), and the release notes are a case study in what happens when a rule language grows organically:

- `if` is now **mandatory** for all rule definitions, and `contains` for multi-value rules — previously optional or import-gated.
- `every` and `in` are available without imports.
- Requirements previously enforced only under `opa check --strict` are now **the default**: "Duplicate imports and imports which shadow each other are no longer allowed."

Read that as a warning. Ten years of accumulated syntax-optionality produced a language where the same intent had multiple valid spellings, and the project's own 1.0 milestone was largely about **removing choice points**. That is exactly the property §6.1 identifies as the main source of LLM generation error, and Anka's Principle 1. **A new DSL gets to start where Rego 1.0 had to spend a decade arriving: one canonical form, enforced from day one.**

The other Rego lesson is that **linting is not verification.** The [Regal](https://github.com/open-policy-agent/regal) linter ships a large rule set — 100+ checks across `bugs`, `idiomatic`, `imports`, `performance`, `style`, `testing`, and `custom` categories (`duplicate-rule`, `constant-condition`, `impossible-not`, `unresolved-reference`, `rule-length`, `prefer-snake-case`, …). This is mature, genuinely useful tooling. But scan the list: **essentially none of it asks whether the policy set is complete, whether two policies conflict, or whether an outcome is unreachable.** It checks that the code is well-formed and idiomatic, not that the *policy* is correct. That distinction is the entire opening for a business-rules verifier.

### 3.2 Cedar — the gold standard for "prove the language, not just the policies"

Cedar (AWS) is an authorization policy language whose [security documentation](https://docs.cedarpolicy.com/other/security.html) describes an unusually rigorous engineering process, and it is worth understanding as the ceiling for what a language project can claim:

1. **A formal model in Lean**, with executable definitions of Cedar's components and *properties* that Lean verifies — "confidence that our definition of the ground truth is correct."
2. **A production engine in Rust**, using only the safe subset, plus property-based testing for properties "difficult to prove in Lean" (e.g. parser/formatter round-tripping).
3. **A differential testing engine** that checks the Rust implementation matches the Lean model — so properties proven about the model transfer to production.
4. **A validator whose correctness is proven in Lean**, checking policies against a schema: unrecognised entity types and actions, actions applied to unsupported principals, improper `in`/`==`, unrecognised attributes, type mismatches, optional attributes referenced without an existence check, invalid extension-type constructor arguments.

Cedar's deliberate guarantees, stated as invariants: policies are **guaranteed to terminate and are effect-free**; Cedar "has no facilities for I/O"; and "the evaluation of one Cedar policy can't effect the evaluation of another policy." Semantics are **default-deny**, **forbid-overrides-permit**, and **skip-on-error**.

Those three guarantees are precisely the restrictions §1 identifies as necessary for analyzability — and Cedar states them as *product features* rather than limitations. **This is the rhetorical move to copy.** Also note the honest division of responsibility in their docs: "It is the responsibility of users of Cedar to correctly define policies... Although Cedar provides tools such as the policy validator, it is ultimately the user's responsibility to write policies correctly." Cedar never claims to verify that your *policy* is right — only that the *language* is sound. The business-rules verifier must be careful to make the same distinction (§9.6.3).

### 3.3 CEL — proving arbitrary rules, for the agentic era

CEL (Common Expression Language) is Google's non-Turing-complete expression language, ["fast, portable, and safe to execute in performance-critical applications"](https://cel.dev/) — evaluation in nanoseconds to microseconds, with access only to data the host application provides. It is an *embedded expression* language, not a rules language: no facts model, no decision tables, no rule sets.

Its relevance here is that **CEL is the language that actually implemented this project's thesis**, in August 2026, with the Z3-powered [CEL Formal Verification Framework](https://github.com/cel-expr/cel-java/tree/main/verifier). See §6.4 for the full analysis — including the three engineering lessons (three-pass taint tracking for zero false positives, bounded model checking, CI-native delivery) that should be treated as a requirements baseline.

CEL's own site now advertises "Formally Verifiable — prove correctness, safety invariants, and equivalence across all inputs using automated SMT solvers like Z3." Note how the marketing is framed: **verification is the feature being sold.** Not syntax, not speed — provability.

### 3.4 Datalog — why termination is a language-design property

Rego is Datalog-derived, and that inheritance is why it is analyzable at all. Datalog's contribution: a **fixed-point semantics** over a finite domain that terminates by construction, with **stratified negation** keeping semantics well-defined. Implementations: **Soufflé** (high-performance, used for static analysis), **DDlog** (Differential Datalog).

The tradeoff cuts both ways: Datalog-family languages struggle with aggregation, ordering, and complex arithmetic, and their termination guarantee depends on the domain being finite (or safely bounded). **That is the trade in miniature** — Datalog gives up expressive convenience to buy decidability, and it is why it has become the standard substrate for program analysis rather than for application logic.

### 3.5 JSON-logic — the minimal end of the spectrum

A tiny JSON-encoded expression format. Trivially analyzable (it is just data, and a small AST), trivially embeddable, and correspondingly limited: no types, no rules, no aggregation, no decision tables. Included because it is worth remembering that **a DSL can be a JSON AST**, which has real practical advantages: no parser to write, trivial to generate from an LLM, trivial to diff, and any language can read it. GoRules' JDM (§4.2) is the industrial-strength version of this idea.

### 3.6 The pattern across all of them

Every language in this cluster bought analyzability by **removing** something:

| Language | What it gave up | What it bought |
|---|---|---|
| Cedar | I/O, loops, cross-policy effects, mutable state | Termination + isolation guarantees, Lean-proven semantics |
| CEL | Turing-completeness, external data access, unbounded cost | Predictable cost, SMT-verifiability |
| Rego/Datalog | General computation, arbitrary recursion | Fixed-point semantics, decidable evaluation |
| DMN (`UNIQUE`/`ANY`) | Ordered rule priority | Checkable conflict-freedom |

**Design rule to carry forward: for every feature you are tempted to add, name the property it destroys.** If the answer is "none," it is probably not a feature worth admitting.

---

## 4. Cluster C — Rules engines and BRMS

### 4.1 Drools / Apache KIE — the power/analyzability tradeoff made flesh

Drools is the long-standing open-source production rules engine (Apache-2.0, `org.kie` artifacts at **10.0.0** on Maven Central, now under the [Apache KIE](https://central.sonatype.com/artifact/org.kie/drools-parent/10.0.0) incubator umbrella). It implements **RETE** with **forward chaining** over a working memory, with stateful and stateless sessions.

Those choices are exactly what make it unanalyzable:

- **Working memory and forward chaining** mean a rule's effect depends on which other rules have already fired — so rule interactions are a *scheduling* problem, not a *set* problem.
- **Rule salience/priority** reintroduces the order-dependence that DMN's `UNIQUE`/`ANY` and Cedar's isolation deliberately avoid.
- The result is the classic complaint that nobody, including the authors, can predict what a large rule base does. A 2,000-rule Drools base is not verifiable by any technique in §5 — the analysis is undecidable in practice, not just expensive.

Drools *does* support DMN at conformance level 3, which is the useful part: **the DMN subset is the analyzable fragment of the Drools ecosystem.** If you want Drools' execution engine, emit DMN and use that subset — do not emit DRL.

### 4.2 GoRules ZEN — the most direct competitor, and already most of this idea

This is the most important entry in the survey, and it substantially narrows the whitespace. GoRules' ZEN Engine is a cross-platform business rules engine: **MIT-licensed**, Rust core, native bindings for Node.js, Python, Go, Java/Kotlin, .NET, and Rust, decisions stored as portable JSON (**JDM — JSON Decision Model**), prebuilt binaries including a WASM/WASI target. Evaluation in microseconds. There is also an open-source React **JDM Editor**.

The **ZEN 2.0** release notes, read as a competitor analysis, are sobering. Verbatim capabilities:

> - **Policy documents**: model decisions as readable documents with typed data models, expressions, decision tables, match blocks and **assertions**.
> - **Workspace analysis**: **static type checking** across policies and graphs. **Type flow, exhaustiveness checking, write-conflict detection** and precise diagnostics, all available **before anything runs**.
> - **Per-column collect**: decision table output columns can collect across all matching rows (`tags[]`) while the rest of the table stays first-match.
> - **Pre-compiled engine** … evaluation is allocation-light and repeat-safe.
> - **Hardened runtime**: out-of-range numbers, arithmetic overflow and malformed inputs return errors or nulls instead of crashing.

Map that against this project's intended value proposition:

| Intended feature | GoRules ZEN 2.0 status |
|---|---|
| Typed fact model / data model | **Shipped** ("typed data models") |
| Static analysis before execution | **Shipped** ("all available before anything runs") |
| Gap detection | **Largely shipped** ("**exhaustiveness checking**") |
| Conflict detection | **Largely shipped** ("**write-conflict detection**") |
| Decision tables | **Shipped** |
| Portable JSON format, multi-language | **Shipped**, MIT |
| Inline tests / scenarios | **Shipped** ("scenario suites run on every change, coverage is measured against decision paths, every answer comes with a replayable trace") |
| LLM authors rules | **Shipped** ("AI copilot and MCP server that edits rules, runs tests and explains decisions") |
| Governance / release process | **Shipped** ("moves from testing to staging to production untouched. Approvals, instant rollback, and a paper trail") |
| **Region-level minimal gap/overlap diagnostics** | Unclear — "precise diagnostics" is not the same claim (see below) |
| **Order-independent semantics as the default** | **No** — "the rest of the table stays first-match" |
| **Proofs (SMT) rather than checks** | Not claimed |

Two genuine openings survive:

1. **Order-independence.** GoRules keeps first-match semantics for decision tables with per-column collect as an opt-in. First-match is exactly the property §5 says destroys order-independence analysis. A verifier that can tell you *"your table's result depends on row order; here is the input where reordering changes the outcome"* is a real, distinct check that a first-match engine structurally cannot offer.
2. **Region-level, minimal diagnostics vs. "precise diagnostics."** GoRules claims precision, not region-minimality. The Calvanese/Signavio finding (§5.3) shows that even competent tools emit redundant, oversized diagnoses. If "exhaustiveness checking" produces a point or a coarse message, there is room for a tool that emits *the minimal uncovered region and the rule to add*. **This must be verified hands-on against ZEN 2.0 before betting on it** — read the docs, run the engine, and look at the actual diagnostic output.

Anything else in the intended feature list is, on current evidence, already built, MIT-licensed, and multi-language. **Building it from scratch is not a viable plan.** The realistic options become: build on top of ZEN (an analysis/verification layer for a MIT-licensed engine with real adoption), or pick a genuinely different wedge (§9.1).

### 4.3 Commercial BRMS

The enterprise decision-management market: **IBM ODM** (formerly ILOG JRules), **FICO Blaze Advisor**, **Progress Corticon** (strong on decision tables), **InRule**, **Decisions.com**, **Pegasystems**, **FlexRule** (FlexRule OPEN v11.1, positioned as a "Decision Intelligence Platform with Decision Governance Runtime"), plus SaaS entrants **DecisionRules.io** (with an "AI Assistant") and **LinkVP** (marketing an "AI Decision Table").

Common shape: a proprietary rule language or decision-table UI, a rule repository, a management/governance console, and enterprise sales. The trajectory is not obviously healthy — this is a mature category with consolidation, long implementation cycles, and a persistent reputation for being expensive and hard to change. **Not a category to enter head-on as a new general-purpose offering.**

### 4.4 Lightweight open-source libraries

`json-rules-engine` (Node), `business-rules` (Python), `NRules` (.NET), `easy-rules` (Java), `durable_rules`. Typically MIT/Apache, unevenly maintained, and — importantly — **almost none of them offer verification of any kind.** They evaluate rules; they do not analyse them. They are useful as evidence that demand for lightweight rules exists, and as a reminder that the *analysis* layer is the unoccupied part of the stack.

### 4.5 Expert systems legacy: CLIPS and JESS

RETE-based expert-system shells from the 1980s–90s, with rule languages and truth maintenance. They faded for reasons that are directly relevant: forward-chaining rule bases became unnavigable as they grew, debugging was painful, and the systems were hard for non-specialists to maintain. **The lesson is that expressiveness is not the binding constraint — comprehensibility and trust are.** That is the same lesson the BRMS market learned expensively, and it is the argument for a small, verifiable language over a powerful one.

### 4.6 The real incumbent: the spreadsheet

The most important competitive fact in this cluster is not any engine. **It is that a very large share of business decision logic lives in Excel, and it has for decades.** The `excel2dmn` tooling exists precisely because the migration path from Excel decision tables to DMN is a known, common need, and Camunda and others publish guides for it.

Two consequences:

1. **Excel is the incumbent to displace or to import from** — not Drools, not GoRules. A decision-table format that business people already understand, with a straight import path from the spreadsheet they actually use, has a far shorter adoption path than a new syntax.
2. **Spreadsheet logic is error-prone and largely untested.** The "errors in operational spreadsheets" literature is extensive. This is not a hypothetical market: it is a large body of unverified, business-critical rules, owned by people who are not programmers, with no version control and no tests. **That is the strongest available wedge for a verification tool** — see §9.1.

### 4.7 Why BRMS projects fail — the evidence to design against

The recurring failure modes, consistent across the category:

- **Rule bases become spaghetti.** Initially clean rules accumulate exceptions, and rule interactions become impossible to reason about. This is the direct consequence of order-dependent, stateful semantics (§4.1).
- **Nobody can predict behaviour.** Once interactions dominate, the system becomes a black box, and every change is a risk.
- **Business users never author the rules.** The core promise of the BRMS category — that domain experts would write their own logic — largely did not happen. Engineers wrote them anyway, and the readable-syntax compromise made the language worse for its actual users.
- **Cost and latency.** Enterprise pricing, long change cycles, and heavy governance overhead.

**The design conclusion, and the strongest strategic argument for this project:** the LLM changes failure mode #3. For the first time the "business user writes the rules" promise is *mechanically enforceable* — the LLM is the translator, and the verifier is the safety net that makes the translation trustworthy. The business-user interface becomes prose, and the generated artifact becomes something a machine can check. That is a genuinely new configuration of an old idea, and it is the only version of the BRMS dream that has a chance of working.

But note what it also implies: **the language no longer needs to be pleasant for humans**, which means it should be optimised purely for verifiability and canonical form (§6.1). The readable-syntax compromise that doomed earlier BRMS products is now unnecessary.


---

## 5. The verification layer — what is actually achievable

This is the part that determines whether the project is real. There is a well-developed literature here, and its conclusions are sharper than most people expect.

### 5.1 The canonical formalization: decision tables as hyper-rectangles

The foundational paper is **Calvanese, Dumas, Laurson, Maggi, Montali, Teinemaa — ["Semantics and Analysis of DMN Decision Tables"](https://arxiv.org/abs/1603.07466)** (BPM 2016). It gives a formal semantics for DMN tables in classical predicate logic with data types, defines the correctness criteria, and supplies scalable algorithms for the two core tasks.

The key idea is a **geometric interpretation**:

- Each rule's input entries define an **iso-oriented hyper-rectangle** in N-dimensional space (N = number of input columns).
- **Overlapping rules** = detectably overlapping hyper-rectangles.
- **Missing rules (incompleteness)** = differencing the N-dimensional universe defined by the columns against the union of all rule hyper-rectangles.

The algorithms are sweep-based spatial-join algorithms, and were implemented on top of the `dmn-js` DMN editor and evaluated on large tables derived from a credit-lending dataset.

### 5.2 Points vs regions — the distinction that decides the design

The paper states the central limitation explicitly:

> Using a general solver to analyze decision tables **"leads to a boolean output (is the set of rules satisfiable?), and cannot natively highlight specific sets of rules that need to be added to a table (missing rules), nor specific overlaps between pairs of rules that need to be resolved."**

This needs to be stated precisely, because the naive reading ("SMT is useless here") is wrong. What SMT actually gives you:

- A **satisfiability verdict** (`unsat` = proof that a property holds).
- A **model / point witness** for the failing case — one concrete input value.

A point witness is genuinely useful. But consider the diagnostic an LLM needs to repair a rule base:

| Diagnostic form | Example | Repairable? |
|---|---|---|
| Boolean | `incomplete: false` | No |
| Point witness (SMT) | `gap at income=47000, dependents=5, region=EU` | Partially — you now know one hole and must guess its extent |
| **Region** (geometric) | `no rule covers income ∈ [40000, 55000] ∧ dependents ≥ 4 ∧ region = EU` | Yes — the fix is directly readable |
| **Region + proposed covering rule** | …`+ rule: when 40000..55000 ∧ dependents ≥ 4 ∧ EU then Refer` | Yes, directly |

So the accurate statement is: **SMT gives you a point; geometric region algebra gives you the region.** Both matter, and the split is:

1. **Region algebra first** for the common case (typed columns, comparisons, ranges, enums, conjunctions/disjunctions) — hyper-rectangle intersection and set-difference, per Calvanese et al. This is what produces *actionable* diagnostics, and it is fast and dependency-free.
2. **SMT (Z3) as the backstop** for what region algebra can't express cleanly — cross-column arithmetic conditions (`loan_amount > 3 * monthly_income`), disequality, and any case where you want an actual `unsat` *proof* rather than a search result. Also for property checking ("prove no input yields both Approve and Decline"), where a model-as-counterexample is exactly the right output.
3. **Minimization of the diagnostic** in both paths — see 5.3.

Z3 is readily available and adds no build friction: `z3-solver` ships as a prebuilt wheel (`z3_solver-5.1.0.0-py3-none-manylinux_2_27_x86_64.whl`).

### 5.2.1 Existing implementation worth studying: `sgreben/rc`

There is an open-source precedent for the SMT-only approach: **[`sgreben/rc`](https://github.com/sgreben/rc)**, a Java library ("completeness, overlap, and constraint satisfaction checker for sets of rules"), implemented on Z3, with a YAML declaration format:

```yaml
ruleSets:
  - name: my rule set
    variables: { temperature: int, temperatureGoal: int, motion: real, stateOut: STATE }
    rules:
      - name: rule 1
        when: temperature > 23 && (motion < 0.3 || brightness < 0.1)
        then: { stateOut: OFF }
```

Its API is instructive because it confirms both the power and the ceiling of the approach:

- `isComplete()` / `completenessCounterexample()` — note it returns a **counterexample**, not just a boolean.
- `isOverlapping()` / `overlaps()` — example values matched by multiple rules.
- `satisfiesConstraint(expr)` / `constraintCounterExamples(expr)` — general property checking with witnesses.

This validates the concept and gives you a reference design to compare against. It is also Java, appears to be a small/personal project at `1.0-SNAPSHOT`, and its YAML→rule compilation is a much thinner authoring surface than a proper DSL. **Steal the API shape; build the region-level diagnostics it doesn't have.**

### 5.3 Diagnostic quality is a first-class requirement

The paper's evaluation of prior art is a warning worth heeding. Signavio's DMN editor *does* detect overlapping and missing rules, but:

> "the diagnosis of overlapping and missing rules produced by Signavio is **unnecessarily large**: it often reports the same rule overlap multiple times."

For a human this is annoying. For an **LLM in a repair loop it is fatal** — you feed back a wall of 40 near-duplicate overlap reports, the model makes a cosmetic change, and you burn a loop. The verifier must emit **minimal, deduplicated, ranked** findings: the smallest condition set that witnesses each gap, and the minimal pair (or n-set) for each conflict. This is an unsolved-in-practice detail that is cheap to get right and is a real differentiator.

### 5.4 Prior art on the analysis tasks themselves

- **Rough sets** — used for completeness/consistency/non-redundancy analysis, but requires boolean or categorical domains; numeric attributes must be pre-discretized into intervals. Loses information.
- **Prologa** — a modeling tool for classical decision tables that *prevents* overlaps and gaps by construction, and supports rule merging (two rules merge when all but one input entry match and outputs agree). Same discretization limitation.
- **OpenRules** — uses constraint satisfaction to analyze decision tables. Same boolean-output limitation as SMT.
- **Signavio DMN editor** — detects overlaps and gaps; technique undisclosed; verbose diagnoses.

The gap in all of this prior art: **none of it is designed to be consumed by an automated repair loop, and none of it handles an LLM-authored rule base.** That is the opening.

### 5.5 The analysis suite worth building

Ordered by value-per-effort for v1:

| Check | What it catches | Difficulty |
|---|---|---|
| **Overlap/conflict** (two rules, disagreeing outcomes, overlapping conditions) | The classic silent-wrongness bug | Core, geometric |
| **Gap/incompleteness** (region matched by no rule) | The bug that costs money in production | Core, geometric |
| **Coverage of a required region** (does the table cover *this* obligation?) | Regulatory/contract compliance | Easy once gaps work |
| **Subsumption / dead rules** (rule can never fire, or is fully shadowed) | Rot, and LLM-generated redundancy | Moderate |
| **Unreachable outcomes** (an output value no rule ever yields) | Spec/implementation drift | Easy |
| **Determinism / order-independence** (would reordering rules change results?) | Hidden first-match-wins coupling | Easy, high value |
| **Property proofs** ("no input yields both Approve and Decline") | Compliance evidence | SMT backstop |

### 5.6 The other half: test generation

Verification proves properties; it doesn't prove *intent*. The complement is **generating test inputs from the rule structure itself** — for each rule, the minimal witness input that triggers it, plus one input just outside each boundary. This catches the failure mode static analysis cannot: a rule base that is internally consistent, complete, and simply *wrong about the policy*.

This doubles as the LLM's feedback signal. An LLM writing rules without executing them is guessing; an LLM that gets back "your rule r7 never fires because r3 subsumes it, and here is an input that falls in a gap" converges fast.

---

## 6. The LLM angle — what the evidence actually says

### 6.1 Direct evidence that constrained DSLs beat general-purpose languages for LLM generation

The most relevant paper is **["Anka: A Domain-Specific Language for Reliable LLM Code Generation"](https://arxiv.org/abs/2512.23214)** (Al Mazrouei, Dec 2025). It is a direct empirical test of the hypothesis behind this project, and the results are strong:

- The DSL was designed explicitly **for LLM generation, not human ergonomics** — the paper's framing is that "features that might annoy human programmers" improve LLM accuracy.
- Despite **zero training exposure**, Claude 3.5 Haiku hit **99.9% parse success** and **95.8% overall task accuracy** from a ~100-line in-context syntax guide.
- On **multi-step tasks the DSL beat Python 100% vs 60% — a 40-point gap.** GPT-4o-mini independently showed the same direction (+26.7 points).
- The advantage **grows with task complexity**: ~0 points on 1–2 operation tasks, +5 on 3–4, +40 on 5+.

The failure taxonomy is what makes it actionable. Python's errors were: **variable shadowing (42%)**, **operation sequencing (31%)**, **chaining confusion (27%)**. All three are *ambiguity* failures, not reasoning failures.

The design principles the paper extracts:

1. **Canonicalization** — exactly one way to express each operation.
2. **Explicit naming** — every intermediate result must be named (no anonymous state).
3. **Structural templates** — block structure scaffolds sequential generation.
4. **Verbose keywords** over symbols.
5. **Type documentation in the prompt.**

**Caveats, stated honestly:** the benchmark is data-transformation pipelines, not decision logic; two models only; no fine-tuning comparison; no user study; and the DSL did *not* help on "hard" tasks requiring nested conditionals and edge cases (−10 points there). That last point is worth taking seriously — business rules *are* the nested-conditional case. So the Anka result supports the *strategy* but does not guarantee the magnitude in this domain.

### 6.2 Where this contradicts my earlier advice

I previously told you the LLM changes the game because it frees the DSL from needing to be human-readable. The Anka evidence supports that but sharpens it: the win does not come from "the syntax can be ugly," it comes from **eliminating choice points**. Verbose, rigid, canonical syntax is not a cost you accept for LLM-friendliness — it is the mechanism that produces the accuracy gain. The implication for your design: **when in doubt, be more rigid, not more flexible.**

This also means the DSL should be designed so that an LLM *cannot* express the same rule two ways, and should be validated with the same kind of benchmark Anka built.

### 6.3 Adjacent evidence: the hybrid-architecture consensus

Industry has converged on a specific split, stated plainly by Camunda in July 2025: use deterministic rules (DMN) where **decisions are deterministic, auditing and compliance matter, and inputs are structured**; use AI agents where **input is unstructured and the task is fuzzy**. Their recommended pattern is to mix them — AI handles classification/summarization as a *preprocessing layer*, rules handle the governed decision.

Note the framing: the LLM is upstream of the rules, not the author of them. Your project inverts that, which is the more interesting bet — but it means you inherit the burden of proving the LLM's output is trustworthy, which is exactly the verifier argument.

### 6.4 The strongest external validation — and the strongest warning

In **August 2026** Google announced the **CEL Formal Verification Framework**, and the announcement is, almost verbatim, the thesis of this project:

> ["Securing the agentic era: Introducing formal verification for CEL"](https://opensource.googleblog.com/2026/08/securing-the-agentic-era-introducing-formal-verification-for-cel.html) — Sean Huh, CEL Team
>
> "We are rapidly entering an era where AI agents can autonomously draft, refactor, and deploy policies... **How do we trust AI-generated policies?** Unit tests may fail to cover the infinite set of possible inputs that occur in production; thus, an AI agent that overfits its policy to existing tests may fail spectacularly in production. **To secure automated policy authoring, we must combine heuristic testing with mathematical proofs.**"

The framework is open source ([`cel-expr/cel-java` verifier](https://github.com/cel-expr/cel-java/tree/main/verifier)), Z3-powered, and explicitly framed as "a safety net for the agentic policy." It proves three things, each with a direct business-rules analogue:

| CEL capability | Business-rules analogue |
|---|---|
| **Equivalence** — refactored policy behaves identically to the original | "The AI refactored my rule base; is it still the same policy?" |
| **Validity** — expression holds for *all* inputs, with counterexamples | Completeness / no-gap proof |
| **Invariants** via `assume`/`assert` implication | "No input yields both Approve and Decline" |

**Read this as validation.** A team with Google's resources reached your conclusion independently, and shipped it into Kubernetes admission policies. The core bet — *AI-authored rules need proofs, not tests* — is correct.

**Read the same thing as a warning.** The marquee use case is already taken, for the policy/authorization niche, by an open-source, Z3-based, free tool from Google. Do not plan to be "the formal verifier for AI-written rules" in the abstract. The remaining whitespace is specific and defensible: **business decision logic** — typed business facts, decision tables, and the *region-level* diagnostics (missing rules, overlaps, unreachable outcomes) that CEL's expression-level verifier does not produce.

**And read it as a free architecture document.** Three engineering decisions in that post are worth copying outright, because they are the difference between a verifier people use and one they disable:

1. **Zero false positives via three-pass taint tracking.** Their words: traditional verifiers are "prone to 'solver hallucinations' — reporting fake bugs when encountering custom domain-specific functions or external variables they don't fully understand." Their fix is to emit **`Inconclusive`** as a first-class third verdict whenever a finding depends on something unmapped, guaranteeing every reported **Violation** is real and reproducible. This is the single most important product lesson in this survey: **a verifier that cries wolf gets switched off in CI, permanently.**
2. **Bounded-model checking with configurable depth** to stop SMT quantifiers from hanging. Practical, unglamorous, essential — see risk §9.6.2.
3. **CI/CD as the delivery surface.** Assumptions and assertions are configured as pipeline gates, so AI-generated changes are validated on the way in rather than audited after the fact.

Adopt all three. A three-valued verdict (`Verified` / `Violated` / `Inconclusive`), a bounded-verification escape hatch, and a CI-native interface are now the expected bar, not differentiators.


---

## 7. Competitive landscape (as of 2026-10)

The market splits into two layers, and conflating them is the main strategic error available here. **Generation is crowded and commoditised. Verification is where the whitespace is — and it is narrowing from both ends.**

### 7.1 Layer 1 — Prose→rules generation (crowded, commoditised)

| Product | What it already does |
|---|---|
| **Trisotech** Digital Enterprise Suite | "AI Model Generators" generate DMN models **from documents (PDF/images)**, display reasoning steps, and were explicitly improved for "less errors and more consistently across the major AI providers." Now on **DMN 1.7** ([Sept 18 2026 release notes](https://www.trisotech.com/digital-enterprise-suite-release-notes-september-18-2026/)). |
| **GoRules** | "AI copilot and MCP server that edits rules, runs tests and explains decisions. It never deploys." |
| **DecisionRules.io** | SaaS business rules engine with an "AI Assistant." |
| **LinkVP** | Markets an "AI Decision Table" feature. |
| **FlexRule** | FlexRule OPEN v11.1, "Decision Intelligence Platform with Decision Governance Runtime." |

**Do not build here.** "Describe your policy and get rules" is table stakes, and every entrant has an existing modeler, engine, and distribution channel.

### 7.2 Layer 2 — Verification of rules (the real battlefield)

This layer was, until recently, nearly empty. It is not any more:

| Player | What they do | Overlap with this project |
|---|---|---|
| **GoRules ZEN 2.0** | **Static analysis before execution**: type flow, **exhaustiveness checking**, **write-conflict detection**, "precise diagnostics." MIT-licensed, Rust core, six language bindings. Plus scenario suites, decision-path coverage, replayable traces, and release governance. | **High — this is most of the intended product, already shipped and MIT** |
| **Google / CEL** | [CEL Formal Verification Framework](https://opensource.googleblog.com/2026/08/securing-the-agentic-era-introducing-formal-verification-for-cel.html) (Aug 2026): Z3-powered proofs of **equivalence, validity, and invariants** for AI-authored policies, with counterexamples and CI integration. Open source. | **High on technique, low on domain** — policy/authorization, not business decisions |
| **`sgreben/rc`** | Java/Z3 library: completeness, overlap, and constraint-satisfaction checking for rule sets, with counterexamples. YAML declarations. | Concept validated; small project, thin authoring surface |
| **Calvanese et al.** | The formal algorithms for overlap and missing-rule detection in DMN tables, implemented in an open-source DMN editor. | The algorithmic foundation — **reuse the ideas, they are published** |
| **Trisotech / DecisionRules / LinkVP** | Generation, with error *reduction*, not error *elimination*. Trisotech's own note says the generator was improved to produce "less errors." | Low — statistical, not proof-based |

**The honest read: this idea has been independently validated and is being actively built.** Google built the proof layer for policy. GoRules built the static analysis and governance layer for decision tables. Calvanese et al. published the algorithms. Trisotech and DecisionRules shipped the generation. Nothing about the concept is novel any more.

That is not a reason to abandon it — it is a reason to **stop planning to build the general version** and find the specific thing that is still missing.

### 7.3 What is actually still missing

Ranked by defensibility:

1. **Order-dependence detection.** GoRules keeps first-match semantics for decision tables ("the rest of the table stays first-match"). A check that says *"reordering rows changes the outcome, here is the input where it happens"* is structurally unavailable from a first-match engine.
2. **Region-minimal diagnostics.** Both prior generations of tooling (Signavio's DMN editor per Calvanese et al., and possibly ZEN's "precise diagnostics") report findings that are precise but not *minimal* — the paper documents the same overlap being reported repeatedly. Feeding oversized diagnostics to an LLM repair loop is actively harmful (§5.3). **Verify this against ZEN 2.0 hands-on before relying on it as the wedge.**
3. **The Excel/legacy-DMN on-ramp.** ZEN, CEL and GoRules all assume you author in their format, or already have machine-readable rules. The enormous body of decision logic in spreadsheets and Level-1 (non-executable) DMN has no verification path at all. **Analysis-as-a-service over artifacts you already have** requires no adoption.
4. **Provenance and regulatory traceability.** Every rule tied to a source clause, owner, and version, with the verification result as evidence. CEL gestures at "auditors clear proof of compliance"; nobody in the decision-table world is doing this properly. This is the feature that sells to regulated industries.

### 7.4 Positioning statement the survey supports

> Not **"AI writes your rules"** — everyone does that.
> Not **"we formally verify AI-written rules"** — Google does that for CEL.
> It's **"we find the holes in the decision tables you already have — Excel, DMN, or AI-generated — and prove your rules don't depend on row order."**

The defensibility comes from three things neither the CEL verifier nor ZEN 2.0 targets together:

1. **Business-decision semantics** — decision tables with `UNIQUE`/`ANY`/completeness, not bare expressions, and explicitly *not* first-match.
2. **Region-level minimal diagnostics** — "no rule covers income ∈ [40k,55k] ∧ dependents ≥ 4", not a point counterexample and not a redundant list.
3. **Zero-adoption ingress** — DMN (including non-executable Level-1 tables) and Excel import, aimed at the enterprise decision-management world rather than the cloud-policy world.

If none of those three gaps survives contact with reality — if ZEN 2.0's diagnostics turn out to be region-minimal and its order-dependence is tolerable, and if the Excel on-ramp turns out not to matter — **then the honest conclusion is that this space is taken, and the right move is to build on top of ZEN rather than against it.** Determine that early and cheaply: run ZEN 2.0 on real tables and read its actual diagnostic output. That experiment costs a day and settles the strategy.



---

## 8. Test corpora — free, real, and adversarial

**[DMCommunity Challenges](https://dmcommunity.org/challenges/)** — 87 published decision-modeling problems, openly posted, with community solutions in multiple engines. This is the best available real-world corpus and should be the v1 benchmark.

Directly relevant ones:

- [Pricing Policy](https://dmcommunity.org/challenges/2024-12-pricing-policy/) (Dec 2024) — business rules
- [Card Approval Decision](https://dmcommunity.org/challenges/2021-10-card-approval-decision/) (Oct 2021) — finance, business rules
- [Rate Loan Risk Category](https://dmcommunity.org/challenges/2021-04-loan-risk-category/) (Apr 2021) — tagged `dmn`, finance
- [Permit Eligibility](https://dmcommunity.org/challenges/2023-03-permit-eligibility/) (Mar 2023) — business rules
- [Medical Claim Processing](https://dmcommunity.org/challenges/2022-05-medical-claim-processing/) — healthcare
- [Soldier Payment Rules](https://dmcommunity.org/challenges/2023-08-soldier-payment-rules/) — business rules, scheduling
- [Compressing Decision Tables](https://dmcommunity.org/challenges/2020-09-compressing-decision-tables/) — decision-table optimization, i.e. exactly the redundancy/subsumption problem
- [Lookup Tables in Decision Models](https://dmcommunity.org/challenges/2024-04-lookup-tables-in-decision-models/) (Apr 2024)
- [Benchmark "Medical Services"](https://dmcommunity.org/challenges/2021-02-medical-services-benchmark/) — tagged `performance`, useful for scale testing

Note the tags on recent challenges: **`agentic-ai`** and **`business-rules`** appear together from Feb 2026 onward — the community is actively grappling with the AI-plus-rules boundary right now. Several challenges remain **open with no published solutions**, which is useful: it means the corpus contains problems that are genuinely hard for existing tooling.

The Calvanese et al. paper also used a **credit-lending dataset** for evaluation; that lineage of benchmark data is worth tracking down for scale testing.

---

## 9. Recommendation

### 9.0 Before anything else: the one-day experiment that settles the strategy

The survey changed the risk profile. This is no longer "is the idea good?" — it is — but **"is there anything left to build?"** GoRules ZEN 2.0 already ships static analysis with exhaustiveness and write-conflict checking under an MIT licence (§4.2); Google already ships Z3 proofs for AI-authored CEL (§6.4); Calvanese et al. already published the algorithms (§5.1).

So before writing a line of DSL, spend **one day** doing this:

1. Install `@gorules/zen-engine` (or the Python binding), take 5–10 real decision tables from the [DMCommunity corpus](#8-test-corpora--free-real-and-adversarial), and run them through ZEN 2.0.
2. Deliberately plant a **gap** and an **overlap** in one table.
3. Read the actual diagnostic output. Answer three questions:
   - Does it find both?
   - Is the output **region-level and minimal**, or a point/verbose message?
   - What happens with a **first-match table whose result depends on row order** — is that detected?

Then the strategy follows mechanically:

| Outcome | Conclusion |
|---|---|
| Diagnostics are region-level and minimal, and order-dependence is handled | **The space is taken.** Build *on top of* ZEN — provenance, Excel/DMN ingress, compliance reporting, the LLM repair loop — rather than competing on analysis. |
| Diagnostics are point-level or verbose, or order-dependence is undetected | **A real wedge exists.** Build the region-minimal, order-independent verifier from §9.4. |

One day of work, and it decides whether the next six months are spent building a competitor or a complement. **Do this before designing syntax.**

### 9.1 The strategic inversion: don't sell a language

Language adoption is the wrong wedge (§7.1). Prose→rules generation is shipping from Trisotech, GoRules, DecisionRules, LinkVP and FlexRule, all with existing engines, modelers and enterprise distribution. A new language with no runtime, no modeler and no users loses that fight on day one — and §4.2 shows the feature list is largely already built under a permissive licence.

What remains genuinely open is **analysis over artifacts that already exist**, because that requires no adoption at all:

> **A verifier for business decision logic that takes an existing Excel or DMN decision table — including non-executable Level-1 DMN that has never been run — finds the conflicts, gaps, dead rules and unreachable outcomes, and explains each one as a minimal region you can act on.**

That is sellable on day one, works on the rule bases enterprises already have, and every competitor's export format becomes your input. It also has the shortest possible path to a first user: **point it at one real spreadsheet and show the owner a hole they did not know about.** No migration, no training, no procurement of a new engine.

Only after owning the verifier should the DSL be layered on as the authoring front-end — because a verifier is the thing a DSL exists to feed, and it is the only credible reason to adopt a new syntax. The DSL becomes the fast path, not the entry requirement.


### 9.2 Interop with DMN is a distribution strategy, not a compromise

The survey supports a specific technical bet: **make DMN import/export a first-class feature, not a competitor.**

What that buys:

- **Execution for free.** Emit DMN and any engine runs it — Camunda 8, the FINOS [Fluxnova](https://github.com/finos/fluxnova-docs-manual) fork of Camunda 7 CE (note: [Camunda 7 CE is EOL](https://github.com/camunda/camunda-bpm-platform)), GoRules ZEN, jDMN, Drools DMN. You never need to build a runtime.
- **Distribution into existing toolchains.** `dmn-js` / bpmn.io already give you a modeler to render what you produce.
- **A free benchmark.** The [DMCommunity corpus](#8-test-corpora--free-real-and-adversarial) is 87 real problems, many with reference solutions in DMN.
- **Credibility.** "Standards-based" defuses the "why should I trust your format" objection.
- **The wedge from 9.1.** Importing existing DMN is how you get your first user without asking them to change anything.

The one thing to avoid: inheriting DMN's weaknesses. It is verbose XML, FEEL has a long list of gotchas, and (critically) its hit policies include **order-dependent** semantics (`FIRST`, `RULE ORDER`) that actively fight analysis. Support them on *import* for compatibility; do not make them the native semantics.

**And seriously consider building on ZEN rather than beside it.** If §9.0 concludes the space is taken, the strongest available position is an analysis and governance layer for the MIT-licensed ZEN engine: it is Rust-fast, has six language bindings, stores decisions as portable JSON (JDM), and has real adoption. Letting someone else own the runtime, the format and the bindings — while you own the verification, the provenance and the LLM repair loop — is a smaller, more defensible business than rebuilding an engine you would be years behind on.


### 9.3 Native semantics, non-negotiable

From the Anka evidence (§6) and the analyzability argument (§1), these are the choices to lock in early:

| Decision | Choice | Why |
|---|---|---|
| Evaluation order | **Order-independent.** All matching rules must agree, or it's a conflict. | Order-dependence is what makes rule bases unanalyzable |
| Totality | **Total functions.** Every input resolves to an outcome. | Gap detection is only askable if totality is required |
| Side effects | **None** in the decision layer. Effects are emitted as commands. | Rules must be evaluable in isolation for pairwise analysis to be sound |
| Syntax | **One canonical form per operation.** No synonyms, no optional sugar. | Choice points are where LLM generation fails (§6.1) |
| Naming | **Every intermediate and rule explicitly named.** | Shadowing was 42% of Python's failures in the Anka benchmark |
| Tests | **Inline, first-class syntax.** | The LLM generates rules *and* the tests that pin their intent |
| Types | **Explicit, declared in the source.** | Doubles as the documentation embedded in the LLM prompt |

Note that the last four are the *same* recommendations Anka derived empirically, and they all point one direction: **when in doubt, be more rigid, not more flexible.** Resist the urge to add sugar.

### 9.3.1 The sharpest single differentiator: static vs runtime violation

DMN's hit policies are documented precisely in the [Fluxnova (Camunda 7 CE fork) manual](https://github.com/finos/fluxnova-docs-manual/blob/main/content/reference/dmn/decision-table/hit-policy.md). The relevant ones:

| Policy | Semantics | Order-dependent? |
|---|---|---|
| `UNIQUE` (**default**) | At most one rule may match. **If more than one matches, the hit policy is violated.** | No |
| `ANY` | Multiple may match, but **all must produce the same output**, else violated. | No |
| `FIRST` | First matching rule wins. | **Yes** |
| `RULE ORDER` | All matching outputs, in table order. | **Yes** |
| `COLLECT` (+ `SUM`/`MIN`/`MAX`/`COUNT`) | All matching outputs, arbitrary order, optionally aggregated. | No (arbitrary) |

Here is the opening, stated precisely:

> **DMN discovers hit-policy violations at runtime, on the inputs that happen to occur. This design proves them at authoring time, on all inputs that could occur.**

`UNIQUE` and `ANY` are exactly the right semantics — they are order-independent and they assert the property you want. But DMN engines only *detect* a violation when a request actually hits the overlapping region. A table can sit in production for years with a conflict that no customer has triggered yet, and a gap that no request has landed in. "It works" and "it's correct" are different claims, and DMN tooling only lets you make the first one.

That is the product in one sentence. Native semantics should therefore be: **`UNIQUE`/`ANY`-style order-independence, required by construction, plus mandated totality — with the violation turned into a compile-time error.**

The corollary for import (§9.2): when you import a DMN table, you can tell the user *immediately* whether their `UNIQUE` table is actually unique across the whole input space, and whether their table is complete. That is a compelling first interaction with an existing, unmodified artifact.


### 9.4 Build order

0. **The §9.0 experiment.** One day, against ZEN 2.0 and real tables. It decides whether this is build-on or build-against, and it must come before any design work.
1. **Region algebra core.** Interval/rectangle intersection and set-difference over typed conditions. No dependencies, pure logic. This is the whole product's spine whether you build the verifier yourself or add value on top of ZEN.
2. **Minimal diagnostic emitter.** Deduplicated, ranked, smallest-witness findings — the thing Signavio reportedly gets wrong and ZEN may or may not do. Include the **three-valued verdict** (`Verified` / `Violated` / `Inconclusive`) and zero-false-positive discipline from §6.4.
3. **DMN decision-table import**, including **non-executable Level-1 tables** that no engine can run. Point it at the DMCommunity corpus on day one. If it cannot analyse real tables, nothing downstream matters.
4. **Excel decision-table import** — the actual incumbent (§4.6), and the shortest path to a first user who has never heard of DMN.
5. **Z3 backstop.** Cross-column conditions and property proofs (`z3-solver` installs from a wheel, no build step). Borrow the CEL verifier's bounded-model-checking approach to keep the solver from hanging.
6. **Order-dependence check.** "Reordering rows changes the outcome; here is the witness input." Structurally unavailable from a first-match engine, and the clearest remaining differentiator (§7.3.1).
7. **Only now: the DSL surface.** Designed last, against the real friction found in steps 1–6 — not in the abstract.
8. **The LLM loop.** Prose → DSL → verify → feed *minimal* diagnostics back → repair. This is where the Anka findings get tested in *this* domain, and where the −10-point "hard task" caveat from that paper needs watching.


### 9.5 How to know it's working

Two benchmarks, both cheap and both honest:

- **Coverage benchmark:** run the verifier over the DMCommunity challenges and report how many gaps/conflicts it finds, and whether the diagnoses are minimal and correct. Any tool can say "inconsistent"; the bar is producing *the* region a human agrees with.
- **Generation benchmark:** build an Anka-style eval — take N DMCommunity problems, have an LLM write them in the DSL from prose, and measure parse success, execution success, and task accuracy **with and without the verifier in the repair loop.** The delta between those two numbers is the entire value proposition, quantified.

If the verifier's repair loop doesn't measurably improve LLM accuracy on real specs, the thesis is wrong and you should know that in weeks, not months.

### 9.6 Honest risks

1. **Anka's result may not transfer.** It measured data pipelines; business rules are precisely the nested-conditional case where Anka showed *no* advantage and a 10-point deficit. Your core bet is unproven in your domain. Test it early (§9.5).
2. **Combinatorial blowup.** Hyper-rectangle differencing is exponential in the worst case. Calvanese et al. got scalability via sweep algorithms and evaluated on credit-lending data — that's the reference to match, but the worst case is real and must be measured on the `performance`-tagged corpus.
3. **Verification is necessary, not sufficient.** A rule base can be complete, consistent, and still wrong about the policy. The verifier catches *inconsistency with the spec as written*, never *inconsistency with reality*. Inline tests and provenance (source clause per rule) are the mitigation, and the limits must be documented or the tool will be over-trusted.
4. **The "all-rounder" ambition stays parked.** Process, state, and time semantics are deliberately out of scope (scope decision, above). The moment they come in, analyzability dies. Revisit as a *separate* language sharing the surface syntax, delegating to a workflow engine.
5. **The space may already be taken — and this is now the top risk.** GoRules ZEN 2.0 ships exhaustiveness checking, write-conflict detection, static type checking, scenario suites, decision-path coverage and a release-governance trail, MIT-licensed, with an AI copilot (§4.2). Google ships Z3 proofs for AI-authored CEL (§6.4). Calvanese et al. published the algorithms in 2016 (§5.1). **If §9.0 comes back "region-level and minimal," the correct response is to build on ZEN, not against it** — and to say so early rather than after six months of work.
6. **Validation cannot be outsourced to the verifier.** The most dangerous failure of a proof tool is inducing false confidence: a rule base that is complete, consistent and order-independent can still be wrong about the policy. Provenance (every rule tied to a source clause and owner) and inline tests are the only defence, and the tool must state its own limits loudly.


### 9.7 One-line summary

> The idea is right — Google proved it and GoRules shipped it. So don't build the general version. Run the one-day experiment (§9.0), and if the diagnostics aren't already good enough, build the verifier that finds holes in the decision tables people already have in Excel and DMN, with region-minimal explanations and an order-independence proof that no first-match engine can offer. Let the language be the last thing you design, not the first.

