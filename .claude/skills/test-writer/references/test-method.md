# Test Method — Objects, Relations & the Three Steps

The conceptual model behind test design. **Platform-independent** — the mechanism is
the same for embedded, cloud, mobile, or hybrid systems; only the tier *names* and
the seam *mechanisms* change. The coverage matrix, the test specs, and the FSD's
§x.0 Test Architecture are all views of this one model.

> **The one purpose of testing: find all bugs before go-live.** We do *not* test
> because we can. Every test exists to catch a way a requirement can fail (or a way
> the code can misbehave); a test that traces to no such failure is removed.

---

## Objects

### Requirement
A statement in the FSD of something that must hold — an `FR`/`NFR` item or a stable
**clause** (per the FSD's convention). The **source of truth** for *what* must be
tested. Testing never invents intent — it traces back to a requirement.

### Behaviour
What a single test case checks. Two kinds:
- **Failure mode** — a specific way a requirement can fail ("a bad-checksum frame
  must be rejected, not accepted").
- **Input-space slice** — a partition or sequence of *normal* inputs that exercises
  the logic ("export → balanced → import converges to the cap"). Designed with
  **equivalence classes**, **boundary values**, and **state-transition sequences**.

A requirement is proven by covering **both** its failure modes and its input space.

### Test case
The atomic unit of verification. One test case:
- **verifies** exactly one requirement's **behaviour**,
- runs at exactly one **tier**,
- has exactly one **interface** as its **subject** (the SUT — the thing it asserts
  about),
- **touches** zero or more **devices** as collaborators it must control or observe.

Tier and subject are properties of the *test case*, not of the interface.

### Interface
An external-facing module whose wire format / handler logic **we own and test** (an
L1 component — a decoder, a driver, an API handler, an adapter). It is the
**subject** of a test case. Each belongs to one **layer** and fronts at most one
**device**. (See `../../fsd-writer/references/test-architecture.md` for layering.)

### Layer
The strict one-way dependency stack (L0 foundation → L1 interfaces → L2 application
logic; open-ended to L0..Ln). The **ownership rule** draws the L0/L1 line: *did we
implement and test the protocol?* — a library/managed client = foundation (L0); a
hand-written decoder/driver/handler = interface (L1). L0 is tested **transitively**.

### Device (external)
The external party an interface talks to — a sensor, a peer service, a relay, a
broker, a server, a network link. A device is a **black box**: tests assert only
that *we* read / encode / decide correctly, never that the device is correct.
**Controllability lives here.**

### Tier
The execution environment a test case runs in — cost-ordered, named per platform
(embedded: **host / target / bench**; cloud: **unit / integration / staging**;
mobile: **unit / instrumented / e2e**; plus **other** = non-runnable: 3rd-party /
CI / review-only). A property of the **test case** (one tier each), chosen as the
*cheapest tier where the bug can manifest **and** is feasible*.

Tier is a property of the *behaviour* (fixed); **maturity** (is the test written
yet) is separate. "Cheap tier first" is scheduling — the full-system test is not a
more advanced fast-tier test, it catches a *different* bug.

### Controllability
A property of a **device**: *can we command it, and how?* — resolved against a tier
into a **mode**. It decides which tiers can carry provocation / input-sweep tests
vs. observe-only validation.

| Mode | Meaning |
|---|---|
| **Drive** | We initiate outward via the real channel (publish / request / write / push). |
| **Feed** | The device can't be commanded, so we supply its *inbound* data through a built-in test hook (a **seam**) — synthesized inputs at the fast tier, a fake-data build flag at the integrated tier. We impersonate the source that sends to us. |
| **Emulate** | Substitute a fake device *outside* the system under test (full-system tier). |
| **Observe** | Real device, uncommandable — watch only (validation, not provocation). |
| **Rig** | Driven by the test infrastructure (link drop, reboot, power-cycle, clock skew). |
| **inherits** | (Not a device mode.) Application logic owns no device; it takes the controllability of the devices it orchestrates. |

The mode is the device-fact **resolved against the tier**: real device present at the
full-system tier → `Observe`; no real device at the fast tier → `Feed`. **If a
behaviour is hard to test, that changes *where* we test it, not *whether*.** Example:
we can't make a real sensor emit a corrupt frame, so we can't test "bad frame
rejected" at the full-system tier — but we don't drop it, we move it to the fast tier
and *feed* a synthesized corrupt frame. Every failure mode still gets a test;
difficulty only decides the tier. If the natural tier is uncontrollable, **add a seam
or emulator to relocate the test there.**

> **Seam** (Michael Feathers' sense) is the test-engineering name for the *hook* —
> the synthesized-input entry point, the fake-data flag. The seam is the *hook*;
> `Feed` is what we do through it.

### Coverage matrix
The generated **ledger** — not a design tool. Rows = interfaces grouped by layer;
columns = tiers; each cell is the **rollup** of the test cases whose subject is that
interface at that tier (`covered/total`). A **device-controllability** column carries
the device property per row (`inherits` where the row owns no device). Generated from
the requirement registry + spec linkage, so it cannot drift from the code.

---

## Relations

```
Requirement ──verified by──▶ Test case ──checks──▶ Behaviour (failure mode │ input slice)
                                 │
                                 ├── runs at ──────▶ Tier        (exactly 1)
                                 ├── subject ──────▶ Interface   (exactly 1, the SUT)
                                 └── touches ──────▶ Device(s)   (0..n, collaborators)

Interface ──belongs to──▶ Layer            (exactly 1)
Interface ──fronts──────▶ Device           (L1: one · L0: a device/infra · L2: none)
Device    ──has─────────▶ Controllability  (a mode per Tier; the interface's column rolls this up)

Coverage matrix = rollup of Test cases by (Interface subject × Tier)
                + device-controllability column rolled up from the fronted Device
```

| Relation | Cardinality | Note |
|---|---|---|
| Requirement → Test case | 1 → 0..n | Proven by its failure-mode + input-slice cases. |
| Test case → Requirement | n → 1 | Every test traces up to a requirement (no orphans). |
| Test case → Behaviour | 1 → 1 | One failure mode *or* one input slice per case. |
| Test case → Tier | n → 1 | Cheapest **feasible** tier; lives on the case. |
| Test case → Interface (subject) | n → 1 | The single SUT. |
| Test case → Device (collaborator) | n → 0..n | Devices it must Drive/Feed/Emulate/Observe to set up the scenario. |
| Interface → Layer | n → 1 | Ownership rule. |
| Interface → Device | 1 → 0..1 | L1 fronts one device; L2 fronts none (orchestrates many). |
| Device → Controllability | 1 → (mode per tier) | Intrinsic device fact, resolved per tier. |

**Two roles an interface plays in a test case** — the distinction a flat matrix
hides:
- **Subject (SUT)** — the one interface whose behaviour we assert.
- **Collaborator** — every *other* device the case must control/observe to create the
  scenario, each with its own controllability mode.

---

## The method (how the objects are used)

Three steps; the matrix is the ledger they write to.

1. **Top-down (from requirements).** For each requirement, enumerate its
   **behaviours** — failure modes ∪ input-space slices — and write a **test case**
   for each. Decompose hierarchically: a top-level (L2) behaviour fans into
   contributing behaviours that surface lower-layer (L1, then L0) test cases. Cover
   both how a requirement fails *and* its normal input space.
2. **Bottom-up (from code).** Walk the code; every branch no test reaches is a lead,
   resolved **upward** — (a) a forgotten test for a specified behaviour, (b) a
   behaviour missing from the FSD, or (c) dead/defensive code. Never write a
   branch-coloring test; fix it at the level that owns it. (Fast tier: a coverage
   tool — gcov / coverage.py / …; higher tiers: a structured manual walk.)
3. **Tier assignment.** Place each case at the cheapest tier where its bug can
   manifest **and** is feasible — feasibility = **controllability** (can we create
   the condition?) + **observability** (can we see the result?). Build a seam or
   emulator to relocate a case when its natural tier is uncontrollable or too costly.

---

## Key rules

- **Purpose:** find all bugs before go-live; no test without a failure mode /
  behaviour it prevents.
- **A tier on a case is a contract:** if a behaviour belongs at a tier, that test
  case must exist (or be owed and tracked as a gap).
- **Coverage shows a line *ran*, not that it's *right*:** a test can execute a branch
  (showing as covered) without checking the result — a bug passes green. Coverage is
  necessary (to catch code nothing tests) but not sufficient; the case must still
  assert the right answer. Bottom-up flags untested branches; correctness comes from
  the top-down assertions.
- **One-way dependency:** application logic must not parse wire formats; interface /
  foundation code must not make application decisions.
- **Extractable shape:** split the pure parse/encode/policy **core** (fast-tier
  testable) from the wire/flow **shell** (integrated/full-system) so failure modes
  drop to the cheapest tier. (This is the source-layout rule from
  `../../fsd-writer/references/test-architecture.md`.)
