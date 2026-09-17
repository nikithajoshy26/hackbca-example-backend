---
name: consolidate-design-docs
description: Merges the already-generated HLD and LLD documents from each component repo of a project (e.g. frontend and backend) into a single unified HLD and LLD for an Enterprise Architect audience. Discovers component repos at runtime, is runner-agnostic, and is resume-safe. Does not re-analyze source code.
---

# Consolidate Design Documents

You are merging documentation that has **already been generated** (by the
generate-design-docs skill or an equivalent) — you are not analyzing source code
and you are not regenerating anything from scratch.

This skill is reused across many projects, so it never assumes specific repo
folder names, and it runs in different agents. It discovers the component repos at
runtime and builds output incrementally so it never hangs on large documents. Its
output uses the same section taxonomy as the generate-design-docs skill, so a
consolidated doc has a section for every section that skill produces.

## Execution Protocol (runner-agnostic — read this FIRST)

This skill runs in different agents (e.g. the GitHub Copilot coding agent on
github.com, or Copilot in VS Code). Do not assume any specific tool name — use
whatever file-read, file-create, file-edit, directory-list, branch, and
pull-request capabilities the current runner provides.

**Build every document incrementally. Never generate a whole document in a single
step before taking an action.** Composing a long document entirely in one
generation and writing it only at the end is the main cause of the agent hanging
on large documents (the consolidated LLD especially). Apply this identically to
BOTH the consolidated HLD and the consolidated LLD:

1. **Create the output file EARLY.** The moment you begin a document, write it to
   disk containing only its Title & Metadata section and the next section — do not
   wait until the whole document is composed.
2. **Then add one section at a time, in the fixed section order below**, each as a
   separate edit/append. Write each section to disk as soon as it is composed, then
   move straight on to the next — never hold a whole document in the buffer.
3. **One section per write.** Small adjacent sections may be combined; never batch
   the whole document into one write.

**Run continuously — do not wait for the user between sections.** Proceed
automatically from each section to the next, and from the consolidated HLD to the
consolidated LLD, through to the run's end state (a pull request opened, or the
no-change stop). Writing a section to disk is NOT a stopping point: after each
write, continue to the next section in the same run. Never end your turn with only
a statement of what you are about to do — if you say you will add a section, add it
in that same turn. Do not ask the user to say "next", "continue", or "yes" to
proceed, and do not pause for confirmation between sections. The only unavoidable
pauses are the runner's own file-write approval prompts, which you cannot control;
as soon as one is approved, continue straight to the next section without waiting
to be told.

**Be idempotent and resume-safe.** Before writing a section, if its `##` heading
already exists in the file, replace that section in place; otherwise add it. Every
run rewrites every section from the current inputs, so re-running is always safe
and always reflects the latest source docs. (This is the "regenerate fully each
run" rule in Hard Rule 4, applied section by section.) If you are re-invoked after
an interruption, do NOT restart from scratch: first read the current state (which
output files exist, which sections each already contains, and whether the
branch/PR exist), then continue from the first incomplete piece. Re-writing a
section that already exists is safe, but do not create a second branch or a
duplicate PR if one already exists.

## Step 0 — Locate the component docs (do this FIRST, exactly once)

1. **If the invocation explicitly names the component repos** (e.g. "consolidate
   `web/` and `api/`"), use exactly those paths and skip discovery.
2. **Otherwise, auto-discover.** List the current workspace's root folder(s) and
   their immediate subdirectories (one level deep only), and collect every folder
   containing BOTH `docs/HLD.md` and `docs/LLD.md`. Match the exact filenames
   `HLD.md` and `LLD.md` — a consolidated output file does NOT count as an input.
3. Do this enumeration **exactly once**. Do not repeatedly rescan or wait for
   files to appear.

**Outcome — exactly one of these, never a loop:**
- **Fewer than two** components found: STOP and report how many you found, their
  paths, and where you scanned. Do not read source code or guess contents.
- **Two or more** found: proceed. Each discovered folder is a "component."

## Component roles

For each component, determine its role (typically frontend/client vs
backend/service) by reading ONLY that component's own `docs/HLD.md` overview —
never its source code. If the role can't be determined, label it by folder name
and proceed. Best-effort, one read per component, never loop.

## Inputs (read-only)

For each component, its `docs/HLD.md` and `docs/LLD.md`. Read each once, in full.
These are your ONLY inputs — never read source code.

## Output target

Write the consolidated docs into the `docs/` folder of the **consolidation target
repository**:
- If the invocation names a target repo/path, use it.
- Otherwise use the repository you are operating in (VS Code: the open workspace
  repo; Copilot coding agent: the repository the session is attached to).
Open the PR (if there are changes) in that same repository. Do not write
consolidated output into the individual component repos unless one of them is
explicitly the target.

Output files:
- `docs/HLD_consolidated.md`
- `docs/LLD_consolidated.md`

(Adjust the filenames if the target repo has an existing `docs/` naming
convention — keep whatever distinguishes the consolidated doc from per-component
docs.)

## Hard Rules

1. **Do not re-analyze code.** Your only inputs are the per-component HLD/LLD
   markdown files. Note gaps rather than inferring from anything else.
2. **Do not simply concatenate.** The result should read as one coherent system
   description, not stacked per-component docs.
3. **Preserve accuracy.** Only reorganize, merge, and de-duplicate — do not alter
   or reinterpret factual claims.
4. **Regenerate fully each run — do not attempt incremental diffing of source.**
   Produce both consolidated documents in full from the current inputs each run,
   rewriting each section in place (see Execution Protocol). Do NOT try to compute
   "what changed since last consolidation" — that comparison is a common cause of
   spinning. The runner's diff/status detects whether the regenerated output
   differs from what's committed (see Process).
5. **Validate merged Mermaid diagrams** using the rules below — a diagram valid in
   each source doc can still break after merging.

## Mermaid Rules for Merged Diagrams

Self-contained — do not look them up in another skill.

- **Always wrap every node label and every edge label in double quotes**, no
  exceptions, even single words. `A["Web UI"]`, not `A[Web UI]`. `X -->|"HTTPS"| Y`,
  not `X -->|HTTPS| Y`. Punctuation in an unquoted label is the most common parse
  error — e.g. `-->|DB queries (Devices/ApiKeys/Users)|` breaks;
  `-->|"DB queries (Devices/ApiKeys/Users)"|` is safe.
- **Rename colliding IDs before merging** — identical IDs silently overwrite each
  other in Mermaid.
- **Rename colliding sequence-diagram aliases** the same way.
- **Never use a reserved word as an ID or alias** (`end`, `graph`, `subgraph`,
  `class`, `style`, `loop`, `alt`, `opt`, `note`, etc.).
- **One statement per line; declare direction once; balance every `activate` with
  a `deactivate`.**
- After merging, re-read once and confirm every label is quoted, no IDs collide,
  no reserved words as IDs. One pass — do not re-parse repeatedly.

## Merge Logic

The consolidated docs use the SAME section taxonomy as the generate-design-docs
skill — one consolidated section per section that skill produces. For each
section, combine every component's content into one coherent whole (do not stack
per-component copies), and keep any section that exists in only one component,
clearly scoped to it. Group content by component role: in the common
two-component case the groups are **Frontend** and **Backend**; with more
components, use each component's role or folder-name label. Read
"frontend"/"backend" below as "client-side component(s)"/"server-side
component(s)."

### HLD merge (`docs/HLD_consolidated.md`) — fixed section order
1. Title & Metadata — project name, last updated date, doc owner; list the
   component repos merged and their last-updated dates.
2. Executive Overview — one overview of the whole system (all components).
3. Objective — combined objective for the system.
4. Architecture Description — combine the components' diagrams into a single
   `flowchart TB` showing them as connected layers/systems, each connection
   labeled with its protocol (HTTPS/REST, gRPC, WebSocket). Merge — do not place
   diagrams side by side.
5. Core Workflows — unified end-to-end workflows spanning client → server where
   applicable.
6. Data Flow — unified end-to-end data flow across components.
7. Key Features — combined; scope a feature to its component where it belongs to
   only one.
8. Infrastructure & Deployment Overview — combined across all components.
9. Deployment Strategy — combined across all components.
10. Data Protection — full data lifecycle across all components.
11. Security Requirements — one section; if components differ, describe each and
    how they relate (e.g. client obtains a token the server validates).
12. Integrations — all integrations from every component in one table; note which
    component owns each.
13. Environment Variables & Secrets Inventory — one inventory, labeled by
    component (names and purposes only, never values).
14. Change Log — entry noting this consolidation run and which component docs
    (with dates if present) were used.

### LLD merge (`docs/LLD_consolidated.md`) — fixed section order
1. Title & Metadata.
2. Module/Component Breakdown — one top-level group per component, with a
   "Cross-Cutting" note where one component's module depends on another's.
3. Key Classes / Functions — merged, keeping per-component attribution; only
   architecturally significant ones.
4. Data Models / Schemas — one section; flag where types/schemas in different
   components represent the same entity.
5. Sequence Diagrams — prefer merged end-to-end diagrams for cross-component
   workflows over separate per-component ones; write each diagram as its own
   append.
6. Error Handling & Retry Behavior — merged, per-component attribution where
   useful.
7. Configuration & Environment-Specific Behavior — merged.
8. Known Limitations / Technical Debt — merged.
9. Change Log.

## Process

Do these steps in order, in a single continuous run. They are ordered
dependencies, not pause points — do not stop for user input between them (see the
Execution Protocol's "run continuously" rule).

1. **Discover** the component repos (Step 0), once. Fewer than two → STOP and
   report. Otherwise continue.
2. **Read** each component's `docs/HLD.md` and `docs/LLD.md` in full, once, and
   determine each component's role.
3. **Build the consolidated HLD** at `docs/HLD_consolidated.md`: create the file
   after its Title & Metadata section, then add each following section in the
   fixed order (validating Mermaid as you go), replacing any section already
   present. Confirm the file exists.
4. **Build the consolidated LLD** at `docs/LLD_consolidated.md`, only after step 3
   — same incremental, section-by-section approach. Do not generate the whole LLD
   before the first write. Confirm the file exists.
5. **Detect changes** using the runner's diff/status capability. If neither
   output file differs from what's committed, STOP — no branch or PR.
6. **Branch**: `docs/consolidated-update-<YYYYMMDD-HHMM>` off the default branch.
7. **Commit** both files: `docs: automated consolidated HLD/LLD update <date>`.
8. **Open the PR** against the default branch, titled
   `docs(hld): automated consolidated update <date>`, labeled `automerge`, body
   noting which component docs (with dates) were used. Never push directly to the
   default branch.

If re-invoked after an interruption, read current state (which files/sections
exist, whether the branch exists, whether a PR exists) and resume from the first
incomplete step above. Re-writing an already-present section is safe; do not
create a second branch or a duplicate PR if one already exists.
