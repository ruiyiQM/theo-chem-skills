---
name: esc-method-development
description: >
  Methodology for developing new electronic structure methods: turning a
  physics question into a well-defined numerical object, implementing it
  incrementally inside a large legacy code, and landing it without breaking
  existing paths. Use when planning or starting implementation of a new
  method, operator, constraint, observable, or theory extension in quantum
  chemistry software — whether reproducing/extending a published method or
  implementing an original derivation. Do not use for verification gates and
  acceptance testing (see esc-verification), routine calculations, or input
  preparation.
---

# Electronic Structure Method Development

Method development in production codes fails in predictable ways: the math
object was never precisely defined, the implementation tangles with legacy
code, or the change lands as one untestable mega-commit. This skill encodes
the discipline that avoids all three.

Core principle: **define the object before writing code, implement in
verifiable increments, keep the host code out of your abstractions.**

## 0. Locate the source of truth

Two entry points, different first steps:

- **Reproducing/extending a published method** (the common case): extract the
  exact protocol from the paper *before* coding — every equation, every
  convention, every parameter the authors used but did not highlight.
  Ambiguities ("which population partition?", "which sign convention?") are
  resolved by re-derivation against the paper's own numbers, not by guessing
  what seems natural. If the SI contradicts the main text, note it explicitly.
- **Original derivation**: write the full derivation down first — target
  observable, operators, approximations, expected limits. If you cannot state
  what the method reduces to in a known limit, it is not ready to implement.

Either way, the deliverable of step 0 is a written definition, not a mental
model.

## 1. Define the numerical object precisely

Turn the physics into an interface-level specification:

1. **Input grammar**: exact keyword syntax and argument order, consistent with
   existing conventions of the host code.
2. **Operator semantics**: what constrained/modified observable, applied to
   which regions, with what sign/pattern per component. A short table beats
   paragraphs — e.g. one row per region: potential pattern, spin symmetry.
3. **Semantic boundaries**: what this object is NOT. State explicitly which
   related-but-different features it must not be confused with or simplified
   into. Two keywords sharing infrastructure does not make them the same
   physical operator.
4. **Bookkeeping vs physics**: distinguish completeness requirements
   (e.g. partitions that must cover the whole system) from actual physical
   content, so nobody later mistakes bookkeeping for a third interaction
   channel.

## 2. Design for coexistence with the host code

You are a guest in a large legacy codebase:

- **Minimal intrusion**: touch the fewest source files possible; reuse the
  existing solver, SCF hooks, and I/O machinery instead of parallel ones.
- **Narrow guards**: any conditional workaround (e.g. a sync guard) fires only
  when the specific combination that needs it is active — not globally.
- **Versioned data formats**: anything another tool reads gets a version
  number and explicit validation on read.
- **Postprocessing stays outside**: coupling extraction, fitting, plotting are
  standalone scripts in a utilities/tests layer with their own unit tests —
  not buried in compiled source. The kernel may be shared; the analysis is not.
- **Regression tests travel with the feature**: extending an existing test
  suite beats building a private one.

## 3. Implement in verifiable increments

Each increment = one commit = one provable property:

1. Plan the commit sequence up front (definition/parser → core operator →
   derived quantities → guards → docs). Each step should compile, run, and be
   checkable on a minimal system.
2. Work on a feature branch with an isolated build directory when builds are
   expensive; record branch, HEAD, and binary hash as you go.
3. After each increment run its smoke check before proceeding. A failing
   earlier increment invalidates everything stacked on it.
4. Keep a running "commits this round" list with one-line summaries — this is
   what makes handoffs and PR descriptions writable later.

## 4. Validate at the boundaries of the method

While implementing, continuously check the limits (this overlaps
esc-verification §5, kept here because it drives design decisions):

- Does the unconstrained/zero-strength limit reproduce the parent method?
- Do known symmetries hold (spin symmetry, equivalence of equivalent regions)?
- Does the smallest analytically-known system give the analytical answer?

Design-flaw discoveries at this stage are cheap; after merging they are
migrations.

## 5. Document as you go

The method is not done until someone else can use it:

- Keyword reference with exact semantics (§1 content, promoted to user docs).
- One worked example: minimal input + expected key outputs.
- Known limitations stated as limitations, with the diagnostic runs that
  demonstrated them referenced if they exist.
- For extensions of published work: which numbers were reproduced, how closely,
  and the decomposed explanation of any residual disagreement.

## Agent output protocol

Before writing implementation code, produce:

1. Source-of-truth summary: paper protocol extracted, or derivation outline,
   with ambiguities listed and resolutions chosen.
2. Object specification (§1): grammar, semantics table, boundary statements.
3. File-by-file implementation plan as a commit sequence (§3).
4. Risk list: which existing behaviors could break, and the regression tests
   that will catch it.

## Anti-patterns

- Coding from memory of a paper: conventions live in the SI; extract them.
- "It's just a small addition" landing as a 15-file unreviewable commit.
- Two operators sharing a solver silently collapsing into one conceptual
  operator in docs or follow-up changes.
- Postprocessing logic accreting inside compiled source where it cannot be
  tested or reused.
- Declaring done at "compiles and runs my case": missing limits checks,
  missing keyword docs, missing example.
