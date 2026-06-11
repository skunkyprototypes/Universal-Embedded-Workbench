---
name: test-writer
description: >
  Designs the test suite for a project from its FSD. Starts from the FSD's
  requirements/clauses, derives every test case top-down (failure modes +
  input-space slices), checks bottom-up against code branches for gaps, and
  assigns each case to the cheapest feasible tier. Emits test specs that mirror
  the FSD chapters and feed a generated coverage matrix. Pairs with the
  fsd-writer skill, which declares the test architecture and the requirements.
  Triggers on "write tests", "design tests", "test cases", "test plan",
  "test spec", "test coverage", "test design", "what tests", "failure modes".
---

# Test Writer Skill

Turns an FSD into a test suite by **design, not by guesswork**. It reads the
requirements, derives the cases that prove them, finds the gaps the code reveals,
and places each case at the cheapest tier where it can fail — producing test specs
that mirror the FSD's chapter structure and feed a generated coverage matrix.

## 1. Purpose

**Testing has one job: find all bugs before go-live.** We do not test because we
can. Every test exists to catch a way a requirement can fail (or a way the code can
misbehave); a test that traces to no such failure is removed. The skill makes that
discipline mechanical — see `references/test-method.md` for the full model.

## 2. Where it fits

This skill is the test-authoring half of the `fsd-writer` pipeline:

- **`fsd-writer`** declares the §2.4 component **layering**, the §x.0 **test
  architecture** (tiers + layer→tier mapping + an empty coverage matrix), and the
  **requirements** (FR/NFR items or stable clause IDs) inside each component chapter.
- **`test-writer`** (this skill) reads those requirements and the codebase, and
  **authors the test cases** — the `specs/*.md` that mirror the chapters.
- A **traceability tool** crosses requirement → component → tier with the spec
  linkage to generate the coverage matrix + gap report. Status is computed, never
  hand-written.

Inputs: the **FSD** (Parts scheme — see `../fsd-writer/references/canonical-fsd-structure.md`)
and the **codebase** (for the bottom-up pass). Read both before designing.

## 3. Invocation

### Mode A — Design the suite from an FSD
```
/test-writer
<path to FSD, or "the current project">
```
1. Read the FSD; list every requirement (clause/FR/NFR) by component chapter.
2. Run the **three-step method** (Section 4).
3. Write the test specs (Section 6), one file per component chapter.
4. (Re)generate the coverage matrix + gap report via the traceability tool.

### Mode B — Evolve the suite for a delta
```
/test-writer update
<a new/changed requirement, a new failure mode, or "close the gaps in <component>">
```
Touch only the affected component's spec; add cases with the next available IDs;
never renumber existing test IDs; regenerate the matrix.

## 4. The method (three steps)

The matrix is the ledger these steps write to. Full detail + the object model in
`references/test-method.md`.

1. **Top-down (from requirements).** For each requirement, enumerate its
   **behaviours** — *failure modes* (ways it can fail) ∪ *input-space slices*
   (equivalence classes, boundary values, state-transition sequences of normal
   inputs) — and write one **test case** per behaviour. Decompose hierarchically: a
   top-level (L2) behaviour fans into contributing behaviours that surface
   lower-layer (L1, then L0) cases. A requirement is proven only when **both** its
   failure modes and its input space are covered.
2. **Bottom-up (from code).** Walk the code; every branch no test reaches is a
   **lead**, resolved *upward* to exactly one of: (a) a forgotten test for a
   specified behaviour, (b) a behaviour missing from the FSD (add the requirement),
   or (c) dead/defensive code. Never write a branch-coloring test — fix it at the
   level that owns it. Use the fast-tier coverage tool (gcov, coverage.py, …);
   higher tiers get a structured manual walk.
3. **Tier assignment.** Place each case at the **cheapest tier where its bug can
   manifest AND is feasible**. Feasibility = **controllability** (can we create the
   condition?) + **observability** (can we see the result?). If the natural tier is
   uncontrollable or too costly, **build a seam or emulator to relocate the case** —
   difficulty changes *where* a behaviour is tested, never *whether*.

## 5. Test-case anatomy

Each test case (see `references/test-method.md`):
- **verifies** exactly one requirement's **behaviour** (one failure mode *or* one
  input slice — not both),
- runs at exactly one **tier**,
- has exactly one **interface** as its **subject** (the SUT — the thing it asserts
  about),
- **touches** zero or more external **devices** as *collaborators*, each handled
  with its own **controllability** mode (Drive / Feed / Emulate / Observe / Rig;
  L2 logic *inherits* the controllability of the devices it orchestrates).

Tier and subject are properties of the *case*, not of the interface.

## 6. Output — test specs that mirror the chapters

Write **one spec file per component chapter** (interface / feature / layer), so the
FSD, the specs, and the matrix share one spine. Within a file, one block per case:

```markdown
### TC-<AREA>-<n>-<slug> — <one-line title>

- **Verifies**: <requirement / clause ID(s)>   <!-- the traceability link -->
- **Tier**: host | target | bench | other
- **Kind**: failure-mode | input-slice (positive/negative/boundary)
- **Subject**: <the interface under test>
- **Collaborators**: <device : mode>, …        <!-- omit if none -->
- **Preconditions**: …
- **Steps**: 1. … 2. …
- **Pass**: <the concrete, asserted expected result>
```

Rules:
- **Cite the requirement ID** in every case — that, plus the code's `@fsd`/coverage
  tags, is what the traceability tool joins. No orphan tests.
- **Match the field names the project's traceability tool parses** (the example
  above is a common convention; adapt to the tool).
- **Coverage is generated** — never hand-maintain a covered/total table in the
  specs or the FSD. The matrix + gap report are produced from the IDs.
- Parameterized cases (one behaviour over many inputs) may use a table row per
  input citing the same requirement ID.

## 7. Quality checklist

- [ ] Every **Must**/**Should** requirement has ≥ 1 test (its failure modes **and**
      its input space), or an explicit, tracked gap.
- [ ] Every test case cites the requirement ID it verifies (no orphan tests).
- [ ] Each case has exactly one subject, one tier, one behaviour; collaborators have
      explicit controllability modes.
- [ ] Bottom-up done: every uncovered code branch is resolved to forgotten-test /
      missing-requirement / dead-code (not left, not branch-colored).
- [ ] Each case sits at the cheapest feasible tier; relocations via seam/emulator
      are noted, not dropped.
- [ ] **Pass** criteria assert the *right answer*, not merely that the line ran
      (coverage ≠ correctness).
- [ ] Specs mirror the FSD chapters; the matrix/gap-report regenerate clean.

Report any checklist failures before finalizing.

## 8. References

- `references/test-method.md` — objects, relations, the three-step method, the
  controllability model, and the key rules (the conceptual core).
- `../fsd-writer/references/test-architecture.md` — the layers, the test tiers
  (named per platform), and the generated coverage matrix this skill writes to.
