# Karkain Process Manager — First Implementation Backlog

## First vertical slice

**Slice goal.** Demonstrate real work end-to-end with no fabricated data: read a fixed process fixture, build a parent/child process graph, serialize a deterministic process snapshot in which every record is `unverified`, and print a stable graph summary. This proves the inspection pipeline and honest degradation while real process enumeration stays BLOCKED.

**Karkain modules/files expected.** `src/main.kark` (entry), `src/cli.kark` (file-driven invocation), `src/probe_registry.kark`, `src/process_model.kark`, `src/graph_builder.kark`, `src/serializer.kark`, `src/diagnostics.kark`, `fixtures/process-sample.txt`, `fixtures/expected-snapshot.txt`, `tests/self_test.kark`. Stdlib: `io` (PRESENT), `json` (PARTIAL, optional), `collections`/`string` (PRESENT). No `system()` call is needed in this slice.

**Exact input** (`fixtures/process-sample.txt`, LF endings):
```
pid=100 name=alpha parent=1
pid=101 name=beta parent=100
```

**Exact expected observable output** (`out/process-snapshot.txt`, byte-stable):
```
schema_version=1
source=shell-out-derived
process pid=100 name=alpha parent=1 status=unverified
process pid=101 name=beta parent=100 status=unverified
graph:
1 -> 100 -> 101
```
Stdout on success: `snapshot written: 2 processes, 0 verified`. Exit code 0. A second run over the same input must produce a byte-identical file.

**Verification command pattern (Karkain v1.2.0, from the repository root):**
```
$env:KARKAIN_ENGINE = "go"
karkain lint src/main.kark
karkain run src/main.kark --target native-x86_64-windows
karkain build src/main.kark -o kprocess --target native-x86_64-windows
.\kprocess.exe
```
Go engine + C-free Windows target are the verified baseline; default `kcc` is environment-BLOCKED. `get_args()` returns an empty array, so input is the fixed fixture file.

## Backlog items

| ID | Feature | Files/modules | Dependencies | Acceptance criteria | Tests | Complexity | Blocking Karkain capability | Definition of done |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| PROC-001 | Process record model and status enum | `src/process_model.kark` | none | Status limited to verified/unverified/unavailable | unit | S | none | Model compiles with fixture |
| PROC-002 | Parent/child graph builder | `src/graph_builder.kark` | PROC-001 | Fixture yields `1 -> 100 -> 101`; cycles handled | unit | M | none | Known-answer graph passes |
| PROC-003 | Deterministic text serializer | `src/serializer.kark` | PROC-001, PROC-002 | Output byte-identical to expected snapshot | unit, regression | S | none | Diff empty |
| PROC-004 | Fixture reader and slice wiring | `src/main.kark`, `src/cli.kark` | PROC-001-003 | Compiled `.exe` writes snapshot and summary | actual-compiled-executable | M | `get_args()` empty -> fixed input | `karkain build` then `.exe` exit 0 |
| PROC-005 | `probe` diagnostic subcommand | `src/diagnostics.kark`, `src/probe_registry.kark` | PROC-004 | Reports enumeration as unverified with reason | unit, integration | M | process APIs BLOCKED | Status report matches fixture |
| PROC-006 | Shell-out adapter (gated) | `src/shell_adapter.kark` | PROC-005 | Allowlisted commands only; failure -> unavailable | integration, security | M | `system()` capture UNVERIFIED | No fabricated records |
| PROC-007 | Snapshot diff engine | `src/diff_engine.kark` | PROC-003 | Stable delta between two fixtures | unit, E2E | M | none | Known-answer diff passes |
| PROC-008 | Append-only snapshot store | `src/snapshot_store.kark` | PROC-003 | Repeated runs append; partial line tolerated | integration, recovery | M | crash-atomic writes PARTIAL | Store test passes |
| PROC-009 | JSON snapshot export | `src/serializer.kark` | PROC-003 | Simple labelled fields only | unit, compatibility | M | `json` PARTIAL | Export fields verified |
| PROC-010 | Action dry-run planner (read-only) | `src/action_planner.kark` | PROC-001 | Plans require confirmation; no live action | unit, security | L | process control APIs BLOCKED | Dry-run output deterministic |

## Dependency order and gates

1. PROC-001 before PROC-002/003. 2. PROC-002 + PROC-003 before PROC-004 slice. 3. Gate: expected snapshot diff empty and compiled `.exe` exits 0. 4. PROC-005/006 probe gate before any process fact is emitted. 5. PROC-007/008/009 diff, store, export. 6. PROC-010 dry-run last. No gate may pass with fabricated records or live destructive action.

## Status

All items: **NOT STARTED**. No tests have been run for this application. Implementation must not begin until all designs and repository initialization are complete. Real process enumeration and handle/DLL inspection remain BLOCKED until a `system()` capture probe passes; GUI process tree BLOCKED; Linux/macOS execution NOT VERIFIED. This product is distinct from Karkain System Monitor (host metrics).
