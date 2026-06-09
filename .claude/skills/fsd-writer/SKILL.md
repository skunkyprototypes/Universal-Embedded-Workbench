---
name: fsd-writer
description: >
  Generates or updates a Functional Specification Document (FSD) for any kind of
  system — embedded, cloud back-end, mobile, networking, SDR, or hybrid. Converts
  rough project descriptions into structured FSDs with requirements, test cases,
  and traceability. Supports initial generation and incremental evolution, and
  loads optional domain packs (e.g. ESP32) for domain-specific detection and test
  libraries. Triggers on "FSD", "fsd", "write FSD", "create FSD", "generate FSD",
  "new FSD", "update FSD", "evolve FSD", "functional spec", "specification document".
---

# FSD Writer Skill

A general-purpose skill that turns a rough, unstructured project description into
a structured Functional Specification Document (FSD) in Markdown, or surgically
updates an existing FSD with new requirements, corrections, or expansions.

## 1. Purpose

This skill:

- Generates a canonical FSD from a rough description (**initial mode**).
- Updates or expands an existing FSD using a delta description (**evolve mode**).
- Dynamically adjusts depth and verbosity based on inferred system complexity.
- Ensures full requirement traceability (FR / NFR <-> test coverage).
- Surfaces risks, assumptions, and constraints as first-class content.
- Produces deterministic, agent-consumable Markdown.

It supports embedded systems, networking, SDR, IoT, cloud backends, mobile apps,
multi-service orchestrations, and hybrid hardware/software projects.

## 2. Invocation

### 2.1 Mode A — Initial Generation

Start a new FSD from scratch.

```
/fsd-writer
<rough description text>
```

Behavior:
1. Parse the rough description.
2. Ask clarifying questions if critical information is missing (Section 5).
3. Infer complexity tier (Section 6).
4. Generate the complete FSD (Section 7).
5. Write the file (Section 10).

### 2.2 Mode B — Evolve Existing FSD

Update, expand, refactor, or correct an already existing FSD.

```
/fsd-writer update <path-to-existing-fsd>
<delta description — changes, additions, clarifications, new constraints>
```

If no path is given, search the project for an existing FSD:
1. Check `Documents/*-fsd.md`
2. Check `Documents/*-FSD.md`
3. Check `docs/*-fsd.md`
4. Check project root for `*-fsd.md`

Behavior:
1. Read the existing FSD in full using the **Read** tool.
2. Parse the delta description.
3. Ask clarifying questions only if the delta introduces architectural ambiguity.
4. Apply changes surgically — preserve all unaffected sections verbatim.
5. Regenerate only the sections affected by the delta.
6. Maintain numbering, cross-references, and the traceability matrix automatically.
7. Write the updated file using the **Edit** tool (preferred) or **Write** tool
   (if changes are too extensive for surgical edits).

### 2.3 Mode C — Grill (deep interview for thin inputs)

Use when the rough description is too thin to infer architecture safely:
fewer than ~3 sentences, no codebase to explore, or no clarity on
protocol/platform/operator. The default behaviour for Mode A is to ask 1-3
questions per round and infer aggressively; Mode C escalates that into a
depth-first interview that resolves the design tree branch by branch.

```
/fsd-writer --grill
<rough description text>
```

Behaviour:

1. Identify the highest-impact unresolved decision (the one that gates the
   most downstream choices — usually connectivity, platform, or operator).
2. Ask **one question at a time**. Wait for the answer before asking the next.
3. Every question must come with the skill's **recommended answer**, not
   just a list of options. The user should be able to accept with one word.
4. Resolve dependencies in order. Do not ask about a downstream choice
   (e.g. OTA mechanism) until its prerequisite (connectivity) is fixed.
5. If a question is answerable from the codebase, config files, or
   `CLAUDE.md` — **explore, do not ask** (see Section 4.3).
6. Stop grilling once the design tree is resolved enough to generate the
   FSD without `(assumed)` markers on architecture-critical fields.
7. Generate the FSD (Section 7) and write the file (Section 10).

Mode C is for the initial interview only. Once the FSD exists, switch to
Mode B (evolve) for incremental changes.

## 3. Tool Usage

This skill uses the following Claude Code tools:

| Tool | When |
|------|------|
| **Read** | Read existing FSD (evolve mode), read project files for context |
| **Glob** | Find existing FSD files, scan project structure for architecture clues |
| **Grep** | Search for protocols, frameworks, dependencies in project source |
| **Write** | Create new FSD file (initial mode) or full rewrite |
| **Edit** | Surgical updates to existing FSD sections (evolve mode) |
| **AskUserQuestion** | Clarifying questions when critical info is missing |
| **Task** (Explore) | Deep codebase exploration when the project has existing source code |

### 3.1 Context Gathering (Before Generation)

Before writing the FSD, the skill should gather context from the project when
source code exists:

1. **Glob** for project structure — `**/*.c`, `**/*.h`, `**/*.py`, `**/*.ts`,
   `**/Cargo.toml`, `**/package.json`, `**/CMakeLists.txt`, `**/go.mod`, etc.
2. **Grep** for protocols and frameworks — BLE, WiFi, MQTT, HTTP, gRPC, REST,
   WebSocket, LoRa, OCPP, etc.
3. **Read** key config files — `sdkconfig.defaults`, `platformio.ini`,
   `docker-compose.yml`, `Makefile`, build configs.
4. Use findings to pre-fill architecture sections and reduce clarifying questions.

### 3.2 Evolve Mode — Diff Discipline

When updating an existing FSD:

- **Never regenerate the entire file.** Only touch sections affected by the delta.
- Use the **Edit** tool with precise `old_string` / `new_string` pairs.
- If a delta adds a new phase, insert it and renumber subsequent phases.
- If a delta adds new FRs, assign the next available FR number in the correct group.
- Always update the traceability matrix when FRs or tests change.
- If the delta invalidates existing content, remove or revise it — do not leave
  contradictions.

## 4. Interaction Model (Clarifying Questions)

### 4.1 When to Ask

The skill must ask clarifying questions when critical architecture-affecting
information is missing. "Critical" means it affects:

- System architecture or component decomposition
- Protocol selection (BLE vs WiFi vs LoRa vs cellular)
- Interface definitions (API style, command format)
- Safety or regulatory constraints
- Multi-phase decomposition
- Hardware or platform selection
- External integrations (MQTT broker, cloud service, Home Assistant, etc.)

### 4.2 How to Ask

Use the **AskUserQuestion** tool with:
- 1-3 precise questions per round in Mode A; **one question at a time** in Mode C.
- **Every question must include the skill's recommended answer**, not just
  a list of options. The user must be able to accept with one word.
- Order questions by dependency. Do not ask about a decision whose
  prerequisite is still open (e.g. don't ask about OTA mechanism before
  connectivity is fixed).
- Phrase questions to unblock the FSD, not to explore nice-to-haves.

Example (good — recommendation up front):

```
Q: How does the device connect?
Recommended: WiFi — matches the dashboard requirement and existing infra.
Other options: BLE, LoRa, Cellular, USB-only.
```

Example (bad — options without a pick):

```
Q: How does the device connect? WiFi / BLE / LoRa / Cellular / USB-only?
```

Multi-question rounds (Mode A only):

```
Questions:
1. How does the device connect?
   Recommended: WiFi. Other: BLE, LoRa, Cellular, USB-only.
2. Do you need OTA firmware updates?
   Recommended: Yes (WiFi). Other: Yes (BLE DFU), No.
3. Who is the primary operator?
   Recommended: Installer/technician. Other: End user, Automated backend.
```

### 4.3 When to Explore or Infer Instead of Asking

**Explore before asking.** Before asking ANY question, check whether it is
answerable from project artefacts:

- Read config files: `sdkconfig.defaults`, `platformio.ini`, `package.json`,
  `Cargo.toml`, `CMakeLists.txt`, `docker-compose.yml`.
- Grep for protocol/framework usage in source (HTTP/REST, gRPC, WebSocket,
  message queues, DB/auth SDKs, BLE/WiFi, etc.) and check whether a domain pack
  matches (Section 14) for domain-specific detection patterns.
- Read `README.md`, `CLAUDE.md`, and any existing FSD or design docs.

If the answer is in the codebase, **explore — do not ask the user**. Asking
a question whose answer is already in the repo wastes attention and
signals that the skill did not gather context (Section 3.1).

**Infer silently** only when:
- The detail does not significantly change high-level architecture, AND
- The cost of being wrong is low.

Safe inferences:
- "web API" mentioned → assume HTTP + JSON
- "logs" mentioned → assume structured logging to console / file / serial
- "dashboard" mentioned → describe generic "dashboard system" without naming tools
- "database" mentioned without type → assume PostgreSQL for relational, SQLite for embedded

When inferring, mark the inference in the FSD with `(assumed)` or group them in
**Section 5: Risks, Assumptions & Dependencies**.

## 5. Complexity Scaling Rules

The skill dynamically scales FSD depth based on inferred system complexity (Low / Medium / High). Complexity is inferred from component count, protocol count, external integrations, real-time constraints, and domain.

For the full complexity tiers table, complexity signals, and per-section scaling behavior matrix, read `references/complexity-scaling.md`.

## 6. Information Extraction & Inference Rules

Given the rough description, the skill must extract or infer the following:

### 6.1 Project Name

Derive a short, descriptive name:
- "ESP32 BLE HID Keyboard" (embedded)
- "Multi-Tenant Billing API" (cloud back-end)
- "Offline-First Notes App" (mobile)

### 6.2 System Purpose & Goals

Extract in 2-4 sentences: what problem is solved, for whom, in what environment.

### 6.3 System Components

Identify major components:
- Hardware / platforms (MCU, SBC, server, cloud)
- Software services / apps / daemons
- User-facing components (mobile app, web UI, CLI)
- External integrations (Home Assistant, OCPP backend, MQTT broker)

If components are implied but not explicit, infer and mark as assumptions.

### 6.4 Functional Requirements (FR)

Convert each described behavior into FR-x.y items:
- Group logically (Communication, Data Processing, User Interaction, Safety)
- Assign priority: **Must** / **Should** / **May**
- Use "shall" language: "The system shall..."

Example:
> "Device sends sensor readings every minute and on threshold events."

Becomes:
- **FR-1.1** [Must]: The device shall send periodic sensor measurements at a
  configurable interval (default: 60 s).
- **FR-1.2** [Must]: The device shall send an immediate measurement when a
  threshold condition is met.

### 6.5 Non-Functional Requirements (NFR)

Extract or infer key NFRs with priorities:
- Performance (latency, throughput)
- Reliability / uptime
- Accuracy / precision
- Scalability
- Power consumption (embedded)
- Security and privacy (authentication, encryption, access control)

### 6.6 Interfaces & Data Models

From the description and answers:
- Identify protocols (BLE, WiFi, USB HID, HTTP, MQTT, LoRa, OCPP, etc.)
- Describe endpoints, characteristics, topics, commands
- Define payload structures (fields, units, types)
- Specify direction (client -> server, device -> cloud, etc.)

### 6.7 Phases

At minimum define:
- **Phase 1**: Infrastructure / Foundation
- **Phase 2**: Core Functional Features
- **Phase 3+** (optional): Optimization, UX, analytics, etc.

Each phase must include: Scope, Deliverables, Exit Criteria, Dependencies.

### 6.8 Operational Procedures

Extract or infer:
- Deployment / flashing / installation
- Configuration / provisioning
- Normal operation workflows
- Failure recovery (reset, re-provisioning, safe-mode)

If not covered in the description, provide a generic but plausible set for the
domain.

### 6.9 Verification & Validation

From extracted requirements:
- Create test cases that verify FRs and critical NFRs
- Organize by phase and feature area
- Use structured format: Objective, Preconditions, Steps, Expected Result
- Build the traceability matrix (Section 8)

### 6.10 Component Layering & Test Architecture

Give every FSD a layered component architecture (§2.4) that the test strategy
falls out of (§8.0):

- Classify each component into a layer with a strict one-way dependency —
  **L0 Foundation/platform → L1 Interfaces → L2 Application logic**. The L0-vs-L1
  line is **ownership** ("did we implement and test the protocol?"): a
  library/managed client to an external service is foundation; a hand-written
  decoder/driver/handler is an interface.
- Draw a layered component diagram in §2.4 (stacked layer boxes, components on one
  row per layer).
- In §8.0, define the test tiers (cost-ordered execution environments, named per
  platform), map them to the layers, and reference a **generated** component × tier
  coverage matrix. This skill declares the structure; a traceability tool fills in
  status.

Platform-independent — contents differ for embedded / cloud / mobile. For the
profiles, the diagram convention (and the Mermaid layout gotcha), and the matrix,
read `references/test-architecture.md`.

## 7. Canonical FSD Structure

All generated or updated FSDs must conform to the canonical structure (Sections 1-11: System Overview, Architecture, Phases, Requirements, Risks, Interfaces, Operational Procedures, V&V, Troubleshooting, Appendix, Related). Section 11 is recommended (not mandatory): a list of `[[wikilinks]]` to related FSDs/ADRs/runbooks for Obsidian-vault navigation. For the full template with all subsections and section inclusion rules, read `references/canonical-fsd-structure.md`.

## 8. Traceability Matrix (Mandatory)

Every FSD must contain a traceability matrix in Section 8.4.

Rules:
- Every FR and NFR with priority **Must** or **Should** must appear in >= 1 test.
- Every test case must reference the FR(s) / NFR(s) it validates.
- Requirements with no test coverage must be flagged as `GAP`.
- **May**-priority requirements may have test coverage but it is not mandatory.
- When updating an FSD (evolve mode), the matrix must be regenerated to reflect
  any added, removed, or changed requirements and tests.

## 9. Formatting & Style Rules

- Output pure Markdown — no HTML tags.
- Use heading levels exactly as defined in Section 7.
- Use bullet lists for requirements; tables for tests, interfaces, and diagnostics.
- Use concise, unambiguous engineering language.
- Use **"shall"** for requirements ("The system shall...").
- Use **"must"** for constraints ("The device must operate on 3.3V").
- Avoid marketing language, filler, and subjective qualifiers.
- Keep requirement IDs stable across evolve updates — never renumber existing IDs
  unless explicitly asked to refactor numbering.
- Use `(assumed)` inline for inferred details.

## 10. Output File Naming & Location

### 10.1 Default Location

If the user does not specify a target path:

```
Documents/<project-name-kebab-case>-fsd.md
```

Create the `Documents/` directory if it does not exist.

Examples:
- `Documents/esp32-ble-hid-keyboard-fsd.md`
- `Documents/multi-tenant-billing-api-fsd.md`
- `Documents/offline-first-notes-app-fsd.md`

### 10.2 Explicit Path

If the user provides a path, use it exactly. Do not relocate or rename the file.

### 10.3 Evolve Mode

When updating, write to the same file that was read. Confirm the path before
writing if it was auto-detected.

## 11. Example Output Snippet

For a complete example FSD snippet (medium-complexity BLE HID Keyboard project) showing expected tone, structure, and detail level, read `references/example-output.md`.

## 12. Evolve Mode -- Detailed Behavior

When updating an existing FSD, follow strict rules for what to preserve, update, add, and remove. Key principles: never renumber existing IDs, always update the traceability matrix, flag contradictions before overwriting. For the complete evolve mode rules (preserve/update/add/remove/conflict resolution), read `references/evolve-mode.md`.

## 13. Quality Checklist

After generating or updating an FSD, the skill must verify:

- [ ] Every **Must** and **Should** FR/NFR appears in the traceability matrix.
- [ ] Every traceability row with no test is marked `GAP`.
- [ ] No `<placeholder>` or `TODO` text remains (flag to user if unresolvable).
- [ ] Section numbering is sequential with no gaps.
- [ ] All phases have scope, deliverables, and exit criteria.
- [ ] §2.4 Component Layering (with a layered diagram) and §8.0 Test Architecture are present.
- [ ] The file has been written to the correct path.
- [ ] (Evolve mode) Unaffected sections are identical to the original.

Report any checklist failures to the user before finalizing.

## 14. Domain Packs

The skill core is **domain-neutral**. Some domains have recurring components,
detection signals, layer profiles, and standard test libraries; these live in
**domain packs** under `references/domains/<domain>.md` and are loaded only when
the project matches that domain — keeping the core applicable to any system.

### Selecting a pack

1. Detect the domain from the description, the codebase, and config files (each
   pack lists its own detection signals).
2. If a pack matches, **read `references/domains/<domain>.md`** and apply it:
   its **layer profile** (concrete L0/L1/L2 contents for §2.4), its **tier names**
   (for §8.0), and its **standard test libraries** (feature detection → test specs
   to fold into §8 and the traceability matrix).
3. If no pack matches, use the platform-independent core only — the architecture
   layering and test tiers from `references/test-architecture.md` still apply; pick
   tier names that fit the platform (e.g. cloud: unit / integration / staging).

### Available packs

| Pack | Domain | File |
|------|--------|------|
| `esp32` | ESP32 firmware (ESP-IDF / Arduino-ESP32): WiFi, BLE, MQTT, OTA, NVS, captive portal, watchdog, logging | `references/domains/esp32.md` |

### Adding a pack

Create `references/domains/<domain>.md` following the same shape: **detection
signals · layer profile (§2.4) · tier names (§8.0) · standard test libraries**
(a feature-detection table pointing to spec files under
`references/domains/<domain>/`). Then add a row to the table above.
