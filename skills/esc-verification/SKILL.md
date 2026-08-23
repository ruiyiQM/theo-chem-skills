---
name: esc-verification
description: Verification culture for electronic structure method development. Use when implementing or modifying numerical methods in quantum chemistry codes, writing equivalence tests, setting numerical acceptance gates, or deciding whether a change is safe to merge.
---

# Electronic Structure Verification

A numerical method change is not done when it compiles and runs — it is done when
you can prove it did not silently corrupt existing physics. This skill distills
the verification discipline used to develop constrained-DFT methods in
production electronic structure codes.

Core principle: **every code change must be shown equivalent-or-better against a
frozen baseline before it counts.** "It looks right" is not evidence; numbers are.

## 1. Freeze the baseline first

Before touching any numerics:

1. Record the exact source state (branch, commit hash), build directory, and
   executable. Pin the binary with a SHA256 hash if results must be reproducible
   across sessions.
2. Run the reference cases you will compare against *before* the change, and
   store raw outputs. Never reconstruct baselines from memory or old logs.
3. Keep tracked worktrees clean. Untracked scratch/benchmark files are not
   yours to delete — leave them alone.

## 2. Equivalence testing: the workhorse

Most method-development changes should be *provably equivalent* on cases where
old and new paths must agree (same operator, same physics, different code path).
Design equivalence tests along these axes:

- **Parallel decomposition**: 1 rank vs N ranks must give identical energies.
  Deltas here expose synchronization bugs (stale eigenvalues, unsynced packed
  matrices) that single-rank tests never see.
- **Boundary crossings**: periodic vs non-periodic limits of the same system;
  packed vs dense matrix storage; Gamma-point vs k-pointed.
- **Operator variants**: if the bug could be operator-specific, test every
  partition/operator the feature supports, not just the new one.

Set explicit acceptance thresholds per quantity, e.g. total energy delta,
state/population deltas, derived quantities (potentials, couplings). Different
quantities tolerate different noise floors — pick thresholds from observed
round-off behavior of the code, not round numbers pulled from air.

## 3. Prove no regression, then prove the fix

After fixing a numerical bug, run the *old* production results through the *new*
code. The deltas must sit at round-off level (e.g. < 5e-9 meV on couplings).
If an old result shifts beyond noise, either the fix changed physics or the old
result was contaminated — both require investigation before proceeding, never a
quiet update of stored numbers.

## 4. Validate against independent anchors

Equivalence tests prove self-consistency, not correctness. Anchor externally:

- **Finite differences**: analytic gradients/forces vs finite-difference of the
  energy. This catches missing force terms better than any internal consistency
  check.
- **Limiting cases**: non-periodic limit of a periodic implementation;
  unconstrained limit of a constrained one; zero-constraint-strength limit.
- **Literature benchmarks**: reproduce a published number with your own protocol
  before claiming agreement or discrepancy. When you disagree with literature,
  decompose the possible sources (operator choice? postprocessing? basis?) and
  test them separately — do not treat "15% off" as one undifferentiated problem.

## 5. Separate diagnostics from accepted data

Not every calculation deserves equal trust. Label explicitly:

- **Accepted**: passed all gates, may enter papers and figures.
- **Diagnostic**: exploratory runs with known limitations (loose basis, partial
  convergence). Keep them for provenance but never mix into accepted datasets.

Record for each accepted result: protocol (all parameters frozen), baseline
hashes, gate outcomes, and known limitations. A result whose protocol cannot be
reconstructed is not a result.

## 6. Diagnose by elimination, one variable at a time

When numerics look wrong (forces inconsistent, coupling decays wrongly):

1. Sweep the axes systematically: parallel layout × grid density × operator ×
   target values. One variable per run.
2. Compare against the most trusted anchor available (usually finite differences).
3. Only conclude "physics" after ruling out "bookkeeping": missing force terms,
   sync guards, partition errors, and index packing are far more common causes
   than real effects.
4. Write the diagnosis down with artifacts (paths to raw outputs, CSVs of the
   sweep) so the next person — human or AI — does not rerun the campaign.

## Anti-patterns

- Rerunning only the case that failed last time. Gates apply to the whole suite.
- Treating agreement with literature as validation without checking that your
  protocol actually matches theirs (operator definitions differ more often than
  people check).
- Letting "small" threshold violations pass because the trend looks fine.
  Trends do not excuse failing gates; investigate or lower the gate explicitly.
- Mixing accepted and diagnostic data in one table "temporarily".
