---
name: esc-verification
description: >
  Verification culture for electronic structure method development: change
  classification, mathematical-definition checks, equivalence testing, and
  numerical acceptance gates when implementing or modifying quantum chemistry
  code. Use when changing Hamiltonians, operators, solvers, SCF procedures,
  gradients/forces, response properties, or parallel numerical kernels in
  electronic structure implementations. Do not use for routine application
  calculations, input preparation, workflow scripting, or non-scientific
  refactoring.
---

# Electronic Structure Verification

A numerical method change is not done when it compiles and runs — it is done
when you can prove it is correct *in the way appropriate to its type*. This
skill distills the verification discipline used to develop constrained-DFT
methods in production electronic structure codes.

Core principle: **classify the change first; the classification dictates the
proof.** "It looks right" is not evidence; numbers are.

## 0. Classify the change

Before any verification work, identify what kind of change this is:

| Class | Examples | Required proof |
|---|---|---|
| A. Refactor / parallelization / optimization | reindexing, MPI redistribution, faster kernel | **numerical equivalence** to frozen baseline |
| B. Bug fix | wrong sign, missing term, sync bug | **controlled deviation**: old vs new vs independent reference, with the deviation explained by the bug |
| C. New physics / improved approximation | new operator, new functional, better solver | **validation against independent references** — baseline equivalence is neither expected nor required |

State the classification explicitly before running anything. Most wasted
verification effort comes from proving the wrong property.

## 1. Verify the definition before the numbers

The most dangerous failure in method development is not a broken kernel — it is
implementing something other than what you intended. Before numerical testing:

1. Write down the implemented equation explicitly (on paper or in the PR).
2. Check it against the derivation: units, normalization, sign conventions,
   index conventions.
3. Beware same-name-different-math traps: population partitions (Mulliken vs
   Hirshfeld vs Becke), polarizability conventions, kernel signs,
   resonant/anti-resonant splittings.
4. Construct the smallest system where the expected analytical behavior is
   known, and check against it.

## 2. Freeze the baseline

For classes A and B:

1. Record exact source state (branch, commit), build directory, and executable.
   Pin the binary with a SHA256 hash if results must be reproducible across
   sessions.
2. Run reference cases and store raw outputs *before* the change. Never
   reconstruct baselines from memory or old logs.
3. Keep tracked worktrees clean. Untracked scratch/benchmark files are not
   yours to delete — leave them alone.

If no trusted baseline exists (new project, inherited code): create a minimal
reference implementation or anchor to analytical limits, validate it, then
freeze that as the baseline.

## 3. Equivalence testing (class A)

Most method-development changes should be provably equivalent on cases where
old and new paths must agree. Design tests along the axes the change can
actually break — do not run irrelevant axes as ritual:

- **Parallel decomposition**: 1 rank vs N ranks. Exposes synchronization bugs
  (stale eigenvalues, unsynced packed matrices) invisible at 1 rank.
- **Boundary crossings**: periodic vs non-periodic limit of the same system;
  packed vs dense storage; Gamma-only vs k-pointed.
- **Operator variants**: if the touched code serves several operators or
  partitions, test each — not only the one you added.

Set acceptance thresholds per quantity from measured reproducibility envelopes:
repeat unchanged-physics calculations across rank counts, compilers, BLAS
libraries, and take the observed noise floor per quantity (energy, force,
population, coupling). Never copy thresholds between projects or quantities.

## 4. Prove no regression, then prove the fix

After fixing a numerical bug (class B), run old production results through the
new code. Deltas beyond noise floor require investigation: either the fix
changed physics or the old result was contaminated. Both block merging until
explained — never quietly update stored numbers.

## 5. Validate against independent anchors

Equivalence proves self-consistency, not correctness — two wrong
implementations can agree perfectly. Anchor externally:

- **Analytical limits**: non-periodic limit of periodic, zero-constraint limit,
  known exact results for minimal systems.
- **Finite differences**: analytic gradients vs FD of the energy. Catches
  missing force terms better than any internal check.
- **Literature benchmarks**: reproduce published numbers with your own frozen
  protocol before claiming agreement or discrepancy. When you disagree,
  decompose the sources (operator choice? postprocessing? basis? protocol?) and
  test them separately — "15% off" is not one problem, it is several hypotheses.

## 6. Convergence is a separate axis

Old-vs-new agreement means nothing if both are unconverged. For accepted
results, verify sensitivity to basis size, integration grid, k-point sampling,
cutoffs, and solver thresholds at the production setting. Record wall time and
memory while you are at it: a numerically correct but 100× slower change still
needs a performance decision before merge.

## 7. Separate diagnostics from accepted data

Label every result explicitly:

- **Accepted**: passed all gates for its class; may enter papers and figures.
- **Diagnostic**: exploratory, known limitations (loose basis, partial
  convergence). Keep for provenance; never mix into accepted datasets.

Record per accepted result: frozen protocol, baseline hashes, gate outcomes,
known limitations. A result whose protocol cannot be reconstructed is not a
result. Store sufficient provenance (inputs, key outputs, hashes); full raw
output archival is optional when sizes are prohibitive.

## 8. When a gate fails

Do **not** relax tolerances, update reference data, or ignore outliers. Instead:

1. Reproduce the failure deterministically.
2. Minimize the testcase — smallest system that still fails.
3. Diagnose by elimination, one variable per run (ranks × grid × operator ×
   target), comparing against the most trusted anchor available.
4. Rule out bookkeeping before concluding physics: missing terms, sync guards,
   partition errors, and index packing are far more common than real effects.
5. Document root cause with artifact paths so the next person — human or AI —
   does not rerun the campaign.

## Agent output protocol

When applying this skill, produce before writing code:

1. Change classification (§0) with one-line justification.
2. Verification matrix: which tests on which axes, for which class.
3. Acceptance criteria per quantity with threshold provenance.
4. After execution: gate outcomes table, deviations and their explanations,
   unresolved risks.

## Anti-patterns

- Demanding equivalence from class B/C changes — this rejects correct fixes and
  blocks real new physics.
- Treating internal consistency as physical correctness.
- Treating literature agreement as validation without checking that protocols
  match (operator definitions differ more often than people check).
- Rerunning only the case that failed last time; gates apply to the whole suite.
- Letting "small" violations pass because the trend looks fine.
- Mixing accepted and diagnostic data in one table "temporarily".
