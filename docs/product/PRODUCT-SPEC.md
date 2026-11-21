# Karkain Process Manager — Product Specification

**Document status:** Design COMPLETE. Research = COMPLETE. Repository creation = COMPLETE. Application implementation = NOT STARTED. Application compile/execute verification = NOT STARTED (toolchain path itself VERIFIED on Karkain v1.2.0).


## A. Product definition

**Product name.** Karkain Process Manager (`kprocess`).

**Purpose and problem.** Interactive process tools export little machine-readable data: [VF] Process Explorer shows handles/DLLs in two GUI windows, [VF] Task Manager shows resource tabs, [VF] System Informer shows graphs and stacks. [RI] The underserved need is scriptable, deterministic process facts with reproducible fixtures and cross-platform output parity. Karkain v1.2.0 has no portable process-enumeration API (BLOCKED/PARTIAL: `system()` shell-outs only, stdout capture UNVERIFIED) and no GUI toolkit, so handle/DLL forensics and GUI process trees cannot be built. This product inspects processes; host-level CPU/memory/storage monitoring belongs to the separate Karkain System Monitor product.

**Vision.** A small, auditable CLI/TUI whose process snapshots and text process graphs can be committed, diffed and used as test fixtures, with every field labelled by source and status.

**Intended users/segments.** [RI] Developers and power users needing scriptable process facts; analysts wanting reproducible fixtures; mixed Windows/Unix teams wanting one output format; Karkain platform evaluators.

**Usage scenarios.** (1) Capture a process snapshot with a probe gate. (2) Export a text parent/child process graph. (3) Diff two snapshots to see what changed. (4) Generate a fixture for tests. (5) Plan (dry-run) an action with confirmation. All scenarios degrade to `unverified/unavailable` fields when probes fail.

**Market positioning.** Read-only-by-default, text-first, probe-gated. Not a Task Manager replacement; it optimises for determinism over interactivity.

**Primary value proposition.** Structured, diffable process data with explicit provenance and no GUI dependency.

**Explicit non-goals/boundaries.** No GUI process tree; no handle/DLL/symbol forensics; no kernel memory dumps; no wait-chain/stack analysis in early releases; no destructive-by-default actions; no host resource metrics (Karkain System Monitor); no language/stdlib modification.

## B. Functional architecture

| Module | Responsibility |
| --- | --- |
| `cli` | Invocation contract; constrained by `get_args()` returning an empty array (see F) |
| `probe_registry` | Register process-data probes and their status |
| `shell_adapter` | Wrap documented `system()` shell-outs; fixed allowlist, no untrusted interpolation |
| `process_model` | Typed `ProcessRecord`, `ProcessGraph`, status and source-label structures |
| `graph_builder` | Derive parent/child links when a source provides them; handle cycles/duplicates |
| `serializer` | Deterministic text and simple JSON output (PARTIAL `json`) |
| `diff_engine` | Compare two process snapshots |
| `action_planner` | Dry-run plan and confirmation/audit for future start/stop operations |
| `diagnostics` | `probe` and `self-test` subcommands |

**Core workflows.** Load config -> run probes -> build records/graph -> serialize deterministically -> write snapshot. **Admin workflows.** Choose field set, name a snapshot, review a dry-run action plan. **Error/recovery workflows.** Failed probe or unparsable output yields `unverified/unavailable` for that field and continues; serializer failure aborts with non-zero exit and no partial file. **Roles and permissions.** Single local user; no auth model; OS privileges not requested. **Configuration/customization.** Versioned text config for field selection. **Import/export.** Process snapshots are the interchange format. **Integration requirements.** Stable exit codes and file output for scripts/CI; no network service.

## C. Competitor analysis

Competitors are **Process Explorer (Sysinternals)**, **Windows Task Manager** and **System Informer (formerly Process Hacker)**, documented in `docs/competitor-analysis/COMPETITORS.md` and sourced from `domain5-systems.md` [all 2026-11-21].

| Capability (Karkain proposed) | Process Explorer | Task Manager | System Informer | Karkain v1.2.0 reality |
| --- | --- | --- | --- | --- |
| Scriptable structured export | No | No | No | PROPOSAL (shell-out + `json` PARTIAL) |
| Handle/DLL inspection | Yes (core) | No | Yes | **BLOCKED** - no process APIs |
| Native GUI process tree | Yes | Yes | Yes | **BLOCKED** - no GUI toolkit |
| Wait-chain / dependency view | Partial | Yes | Partial (stacks) | PROPOSAL as text graph |
| Cross-platform | Windows only | Windows only | Windows only | Targets listed; execution UNVERIFIED |
| Cost/licence | Free download | Bundled | MIT | No product price defined |

**Platform/deployment.** [VF] Process Explorer Windows 11+/Server 2019+, portable ZIP; Task Manager in-box on supported Windows; System Informer Windows-only portable download. **Licensing.** Process Explorer and Task Manager exact terms [UA]; System Informer MIT [VF] (successor to GPL-era Process Hacker [VF]). No prices invented. **Strengths/limitations.** [RI] Process Explorer: deepest handle/DLL forensics, Windows-only. Task Manager: universal but shallow. System Informer: broadest and open source, but Windows-only and rebrand-confused. **Market gaps.** [RI] Deterministic, reproducible, cross-platform process inspection with machine-readable output. **Karkain differentiation.** PROPOSAL: diffable process snapshots and text graph, probe gated, no GUI dependency. **Features deliberately not copied.** GUI trees and icons, handle close/kill UI, symbol servers, wait-chain trees, kernel dumps, themes.

## D. Technical architecture

**The UI is CLI/TUI/service-first because Karkain v1.2.0 has no native GUI toolkit (BLOCKED); a GUI is a planned layer contingent on ecosystem capability and is not in any current release.** Single-process CLI, no daemon; process boundaries only if a future service mode is approved. Component boundaries match B. UI architecture: stable line-oriented text, no curses dependency required for MVP. Domain model: `ProcessRecord` (id, name, owner, attributes), `ProcessGraph` (edges), each field with `source` and `status`. Storage: local append-only files (E). API/interface: CLI subcommands, file output, documented exit codes. Event/message: none in MVP; `async` builders are PRESENT but not needed. Logging/diagnostics: probe report plus `self-test`. Configuration: versioned file. Plugin/extension: planned CLI probe sources; no binary plugin ABI. Security boundary: read-only by default; any action operation requires explicit dry-run then confirmation and is audit logged.

## E. Data design

**Entities.** `ProcessRecord` (pid, name, owner, parent id, attributes with source/status), `ProcessGraph` (ordered edges), `Snapshot` (schema_version, records, graph), `ProbeResult`, `Baseline`, `ActionPlan`. **Relationships/ownership.** `Snapshot` owns records and a graph; the local user owns snapshots. **Validation.** Stable field order, source label required, status enum, schema version present; graph edges validated for cycles/duplicates. **Lifecycle.** created -> serialized -> appended -> retained/pruned. **Persistence.** File-based append-only text (NDJSON-style) plus manifest for baselines. Relational `db2` is PRESENT/PARTIAL but unnecessary for immutable snapshots; text keeps fixtures diffable. **Transactions.** Single-record append; crash-atomic writes NOT VERIFIED (PARTIAL), so readers tolerate a partial last line. **Backup/restore.** File copy. **Migration.** `schema_version` field. **Import/export.** Snapshot text/JSON. **Audit history.** Probe results, config hash and any action plan stored with the snapshot.

## F. Karkain implementation architecture

Compiler/runtime: Karkain v1.2.0 only; working baseline `KARKAIN_ENGINE=go` + `--target native-x86_64-windows` (C-free backend). Available stdlib: `io` PRESENT; `system`/`env`/`process`/`time` PARTIAL shell wrappers (output capture UNVERIFIED; `get_args()` returns an empty array in the checked-in module); `filesystem` PARTIAL (PowerShell shell-outs); `json` PARTIAL (field extraction only); `crypto`/`encoding`/`collections` PRESENT; `db` PRESENT/PARTIAL (not required). Required ecosystem: none external. Missing dependencies: portable process-enumeration APIs (BLOCKED), handle/DLL APIs (BLOCKED), GUI toolkit (BLOCKED), full JSON DOM (PARTIAL), cross-platform metadata (PARTIAL). FFI/OS integration: only the Karkain stdlib `system()` shell-out, used as a Karkain-compatible native interface justified by the absence of process builtins; not a replacement language. Native opportunities: deterministic serializer, graph builder and diff in pure Karkain. Build/packaging: one `.exe` via `karkain build`. **Blockers (no Karkain changes, no other language):** (1) real process enumeration BLOCKED pending a passing capture probe; (2) handle/DLL inspection BLOCKED; (3) GUI process tree BLOCKED; (4) `get_args()` empty -> first slice uses a fixed input file; (5) Linux/macOS execution NOT VERIFIED.

## G. Non-functional requirements

- **Performance.** Workload: snapshot of up to 500 processes, 100 repeated captures on the ~4 GB machine. Method: `karkain bench` plus wall-clock. Threshold: recorded from baseline before any claim; none asserted now.
- **Memory.** Same workload; method: host RSS sampling. Criterion: bounded buffer, no growth with repeated captures.
- **CPU.** Criterion: shell-out dominated, no busy loop; method: `prof` where available.
- **Scalability.** Workload: 10,000 stored snapshots; method: timed `diff`; criterion: cost scales with the compared inputs.
- **Reliability.** Criterion: identical input yields byte-identical snapshots and graph order.
- **Security.** Criterion: fixed shell-out allowlist; injection inputs rejected; actions default to dry-run with confirmation.
- **Privacy.** Criterion: all data local, no network calls.
- **Portability/maintainability/accessibility.** Text-only, locale-independent, module boundaries per B.
- **Observability/recovery.** Probe status always reported; failures yield `unverified/unavailable` and no crash.

## H. Compatibility and portability

Supported OS: Windows x86-64 VERIFIED toolchain; Linux cross-target output PRESENT, execution NOT VERIFIED; macOS NOT VERIFIED. Architecture: x86-64 first. Compatibility: Karkain v1.2.0 only. Package: single executable; install/uninstall = copy/delete. Config/data migration via `schema_version`; no backward compatibility before v1.2.0. Upgrade/rollback: retain previous binary and copy data directory. Cross-platform differences: shell-out commands differ and must be re-probed. Unsupported: GUI, handle/DLL forensics, wait chains, remote hosts, non-x86-64 targets.

## I. Testing strategy

All tests PLANNED; none run for this application. Unit: serializer/graph/diff with fixed fixtures (criterion: byte-identical). Integration: probe registry + shell adapter stub (criterion: correct status on pass/fail). E2E: run from a fixed input file (criterion: snapshot with expected fields). Platform-specific: Windows x86-64 only until others verified (criterion: `.exe` runs). Security: allowlist and injection tests (criterion: rejected). Performance: workload from G (criterion: baseline documented). Regression: snapshot corpus (criterion: no drift). Compatibility: v1.2.0 only. Actual-compiled-executable: `karkain build` then execute (criterion: exit 0, expected output). Failure/recovery: forced probe failure (criterion: `unverified/unavailable`, no crash).

## J. Implementation roadmap

- **MVP / vertical slice:** deterministic process snapshot schema plus probe-gated adapter and one verifiable snapshot workflow; gate = compiled executable emits byte-stable snapshot with provenance.
- **Core functional release:** text process graph and diff; gate = reproducible graph from fixed fixtures.
- **Advanced release:** config, baselines, action dry-run; gate = Windows probes pass and output stays deterministic.
- **Production-readiness:** docs, packaging, Linux/macOS execution verification; gate = planned suite executed and passing.
- **Dependencies:** process model before graph/serializer/diff; probe gate before field emission.
- **Risks/mitigations:** capture UNVERIFIED -> `unverified/unavailable`; `get_args()` empty -> file-driven slice; no process APIs -> stay probe-gated; GUI BLOCKED -> CLI/TUI; no fictional dates or completion claims.
